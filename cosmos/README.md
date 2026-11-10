# COSMOS plugin for the F Prime cFS reference

One COSMOS plugin gem carrying both ground styles side by side on the same cFS UDP link:

- the unmodified [nasa/cfs-cosmos-plugin](https://github.com/nasa/cfs-cosmos-plugin) (`cfs-cosmos-plugin/` submodule):
  cFE/cFS commands, telemetry, EVS, file packets as its `CFS-<n>` targets;
- one separate top-level COSMOS target per F Prime app (`FPRIME_APP` for `FPrimeDeployment`), generated from the F Prime
  topology dictionary by [fprime-cfs-cosmos-one-app-one-target-generator](https://github.com/LeStarch/fprime-cfs-cosmos-one-app-one-target-generator)
  (`libs/fprime_cfs_cosmos` submodule).

Nothing is written into the nasa submodule: the build copies it, patches the number of cFS instances, merges the
generated bundles into the copy (`fprime-cfs-cosmos-merge`, which also installs the compatibility helpers the generated
targets need and places the F Prime `MAP_TARGET` lines before the CFS ones) and builds the gem from the copy.

## Building

```bash
pip install -r requirements.txt           # includes libs/fprime_cfs_cosmos (generator + merge tools)
make SIMULATION=native prep
make                                      # cFS + F Prime apps, dictionaries
make cosmos_plugin                        # -> build-artifacts/native/default_cpu1/cosmos/openc3-cosmos-cfs-<version>.gem
```

Intermediate targets: `make cosmos_bundles` (generated F Prime targets under `.../cosmos/bundles/<TARGET>/`),
`make cosmos_merge` (the merged plugin tree under `.../cosmos/cfs-cosmos-plugin/`; its `.stage.stamp` marks a complete
stage, so deleting the tree rebuilds it). The CMake options live in the *arch* build's cache, which cFE configures
itself (the `make prep` options do not reach it), so set them there and rebuild:

```bash
cmake -DCOSMOS_NUM_CFS_TARGETS=2 build-artifacts/native/default_cpu1 && make cosmos_plugin
```

`COSMOS_PLUGIN=OFF` skips all of this (without the generator tools on `PATH` the targets are skipped with a warning
instead), `COSMOS_PLUGIN_VERSION` sets the gem version (default `1.0.0`), `COSMOS_NUM_CFS_TARGETS` the number of cFS
instances the nasa plugin connects to (default 1; its source hard-codes 2, and its `plugin.txt` only knows two
`cfs_<n>_intf_ip` addresses, so 1 or 2). Ruby's `gem` is used when installed, otherwise the build runs it in the OpenC3
`cmd-tlm-api` container (`COSMOS_GEM_BUILDER=host|docker` forces one, `COSMOS_IMAGE` overrides the image). Every
change to the dictionaries, the nasa submodule, the generator's `src/`/`pyproject.toml` or the options above rebuilds
the affected steps; after a
`libs/fprime_cfs_cosmos` bump also re-run `pip install -r requirements.txt` so the installed tools match.

The built gem is checked (`check-plugin.sh`): every file each bundle's `manifest.json` lists and the helpers it requires
are in the stage and the gem, the F Prime `MAP_TARGET` lines precede the CFS ones (nasa's ID-less file packets would
otherwise swallow every F Prime packet) and the instance count is the requested one. The merged `plugin.txt` itself is
only grep-checked, not ERB-rendered: COSMOS renders it at `cli load`. CI (pull requests, pushes to `devel`/`main`) builds the gem, renders the generated target
against the real openc3 gem (`libs/fprime_cfs_cosmos/tests/cosmos/run.sh`), exercises the docker gem builder once and
publishes gem, `vars.json` and `compose.override.yaml` as the `cosmos-plugin` workflow artifact.

The gem keeps nasa's name `openc3-cosmos-cfs` (the gemspec is theirs): in a COSMOS scope it *replaces* an installed
nasa plugin (`openc3.sh cli unload openc3-cosmos-cfs-<old>.gem__0` first) rather than sitting beside it.

Adding another F Prime app is one `cosmos_add_fprime_target(...)` call in `cmake/cosmos_plugin.cmake`.

## Running

1. COSMOS project ([openc3-project](https://github.com/OpenC3/cosmos-project)): copy `compose.override.yaml` next to its
   `compose.yaml` (publishes UDP 2234 on the operator's loopback so `to_lab` telemetry from a cFS on the same host
   reaches the plugin's `CFS-1` interface; widen it to `"2234:2234/udp"` for a cFS on another machine), then
   `./openc3.sh run`.
2. Install the plugin with the variables for this reference (`vars.json`):

   ```bash
   ./openc3.sh cli load /path/to/openc3-cosmos-cfs-1.0.0.gem --variables /path/to/vars.json
   ```

   - `cfs_1_intf_ip`: address COSMOS sends commands to (`ci_lab`, UDP 1234). For a cFS on the docker host this is the
     gateway of the COSMOS compose network, `172.18.0.1` on a host where it is the first user-defined network — check
     with `docker network inspect <network> -f '{{(index .IPAM.Config 0).Gateway}}'` (`docker network ls` names it);
     use the cFS host's address otherwise.
   - `global_tlm_output_ip`: address the plugin's TO_LAB enable-output command tells cFS to send telemetry to (UDP
     2234). `127.0.0.1` reaches the published port when cFS runs on the docker host; otherwise the COSMOS host's address.
   - `fprime_fprime_app_target_name`: name of the generated F Prime target (default `FPRIME_APP`).
3. Start the reference (`build-artifacts/exe/cpu1/core-cpu1`) and send `CFS-1 TO_LAB_CMD_ENABLE_OUTPUT` (its `DEST_IP`
   defaults to `global_tlm_output_ip`; the plugin's `CFS_FLEET`/`CFS` test procedures send it too) — `to_lab` sends
   nothing until that command enables output, and it has a single
   destination, so run COSMOS or `fprime-gds` against one cFS instance, not both. On one host they also contend for
   UDP 2234 itself: with the override above the operator container holds the port, and `fprime-gds`'s `cfs-lab` adapter
   fails with `Address already in use` until COSMOS (or just `openc3-operator`) is stopped.

Then `FPRIME_APP` carries the F Prime commands (`CfsCore.cmdDisp.CMD_NO_OP`, ...), channelized and packetized telemetry
(`HEALTH`, ...), and `CFS-1` the cFE/cFS ones; both targets decode on the same `CFS-1_INTF`.

## Known limitations

- F Prime events arrive as the `FPRIME_APP FPRIME_EVENTS` packet (stream id 0x0802): one opaque `RECORD` block that
  COSMOS shows as raw bytes. Readable events come from the fprime-gds event decoder through the
  [fprime-cosmos](https://github.com/fprime-community/fprime-cosmos) `cosmos` data transport
  (`pytest --data-transport-selection cosmos --cosmos-target FPRIME_APP --cosmos-layout cfs`, as CI runs `test/int`).
- CI renders the generated target in the real openc3 gem but not the merged `plugin.txt` (the generator's load test has
  no merged-plugin mode yet); an unbalanced ERB edit there would surface at `openc3.sh cli load`.
- The nasa plugin has no VARIABLE for the number of cFS instances (`$cfs_fleet_globals_num_cfs_targets`), hence the
  staged-copy patch; `COSMOS_NUM_CFS_TARGETS=2` maps the F Prime target onto both `CFS-<n>_INTF`s and needs 2235/udp
  published and `cfs_2_intf_ip` added to `vars.json` (nasa's default is `172.17.0.1`, docker0, not the compose gateway).
- `cosmos/cfs-cosmos-plugin` pins the tip of nasa's `dev` branch (the compatible layout the merge was validated
  against), not a release tag; bump deliberately and let `make cosmos_plugin` (patch detector, `check-plugin.sh`) and
  the CI load test judge the new tree.
- `libs/fprime_cfs_cosmos` points at the `nasa-plugin-compat` branch of the `lestarch-autobot` fork, i.e.
  [LeStarch/fprime-cfs-cosmos-one-app-one-target-generator#1](https://github.com/LeStarch/fprime-cfs-cosmos-one-app-one-target-generator/pull/1).
  When that PR merges, point `url`/`branch` in `.gitmodules` at `LeStarch/...` `main`, `git submodule sync`, bump the
  pin and re-run `pip install -r requirements.txt`.
