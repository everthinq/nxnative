# macOS 15 (Sequoia) / Metal 3 Mac build

This branch (`metal3-mac` of [everthinq/nxnative](https://github.com/everthinq/nxnative)) builds the Mac app for
**macOS 15 on Apple silicon**, where Metal 4 does not exist. Upstream nxnative needs macOS 26.6 and Xcode 27.2; on
macOS 15 its NVN layer stops with `has no Metal 4 (no Metal 4 GPU family)` and nothing is drawn.

It is unofficial and not supported upstream. It is built on dreampiggy's opt-in Metal 3 backend
([nxnative-iOS18](https://github.com/dreampiggy/nxnative-iOS18), branch `ios18-upstream`, proposed upstream as
PR #2647 and declined because it targets an older baseline). See [ios18.md](ios18.md) for that backend's design and
its iOS 18 notes.

## Status

| What | State |
| --- | --- |
| Mac host and app build on macOS 15 with Xcode 26.2 | works (host and launcher `minos 15.0`) |
| Window, present path and display link under `DM_METAL3=1` | works: Balatro (OpenGL, `0100CD801CE5E000`) plays windowed on an M1 / macOS 15.8, no `[RED]` lines, about 45 ms game-to-screen |
| NVN games (the Metal 3 replay itself) | works: Cat Quest II (Unity, `01008BE00E968000`, update v720896, provisional profile) plays windowed on an M1 / macOS 15.8 at a steady 60 fps, no dropped frames, GPU about 2.5 ms per frame, about 1.2 GB |
| Cat Quest (`0100A2F006FBE000`, Unity 5.6, update v131072, provisional profile) | works: plays windowed and fullscreen on an M1 / macOS 15.8 at 60 fps (p99 16.7 ms, a few dropped frames while loading), GPU about 3.5 ms per frame, about 1.1 GB. Needed the two build fixes below |
| Hollow Knight: Silksong (`010013C00E930000`, Unity, update v655360 / 1.0.30000) | works: steady 60 fps over a ~7-minute session (frame time p50 16.67 ms, GPU about 6 ms a frame, 64 frames dropped in all), memory footprint 1.5–2.4 GB, exited cleanly. No rewrite or shader-split fixes needed (3 modules rewritten, 2,450 shaders, Mac app 6.6 GB); the pipeline archive fails with `air-tt` as on Cat Quest, which the Metal 3 backend does not use. Close large apps first on an 8 GB Mac |
| Vulkan games under `DM_METAL3=1` | not tested |

## Requirements

- Apple silicon Mac with BC texture support (every M-series chip), macOS 15.
- Xcode 26 with its Metal toolchain (`xcrun -sdk macosx metal --version` must work).
- Python 3.12 or newer with the packages from the main README, including `zbic`:
  `pip install lz4 "capstone>=5,<6" fonttools zbic backports.zstd nsz cryptography`
- `glslang` (Homebrew) for OpenGL games.

## Building a game

```sh
cd nxnative && source .venv/bin/activate
DM_METAL3=1 python3 scripts/build_ipa_from_nsp.py ~/Games/<game folder> -k ~/Games/prod.keys \
  --platform mac-app --allow-other-metal
```

- `DM_METAL3=1` selects the Metal 3 backend at compile time (`-DDM_METAL3=1` in `dm/host/Makefile`). Without it the
  host is the upstream Metal 4 build, which runs on macOS 15 only for OpenGL / Vulkan games' shader capture and
  presents nothing.
- `--allow-other-metal` is needed for NVN games: the shader chain is pinned to Xcode 27.2's Metal compiler
  (`dm/gfx/tools/dmsh_chain.lock`), and macOS 15 cannot run Xcode 27. Shaders built by another compiler can render
  wrong pictures; report such a case with the game's log.
- Add `-o <path>.app` to put the app somewhere other than the game folder.
- The build folder `<title ID>_build` beside the game files holds the converted NSPs, the extracted game and the
  shader cache (about 10 GB for Silksong). The finished app does not use it, so it can be deleted once the game
  plays; the next build of that game then starts from scratch and takes the full time again.

The app opens a settings window first; press **Play**. Its log is `~/Library/Logs/nxnative/<title ID>.log` (the
line `EXPERIMENTAL classic Metal replay` confirms the Metal 3 backend), its saves and `env.txt` are in
`~/Library/Application Support/nxnative/<title ID>/`.

## What this branch changes

On top of `dreampiggy/nxnative-iOS18@e8d54cd2` (upstream `e0cf2ac7` plus the Metal 3 backend):

1. `dm/mac/min_os.mk`: `MACOS_MIN := 15.0` (upstream 26.6). Make's `:=` wins over an environment `MACOS_MIN`, and
   the NSP script runs `make` without it, so the Mac app was stamped 26.6 and macOS 15 refused to open it. The host,
   its own Metal libraries and every game, OpenGL and Vulkan shader library read this line
   (`dmsh_chain.mac_min`), so they all target macOS 15 (Metal Shading Language 3.2).
2. `dm/gfx/nvn/dmsh_chain.h`: the shader-chain version the build regenerates from that change.
3. `dm/tools/patch_nso.py`: `jump_masked_bit`, a jump table whose index is a low-bit mask with its top bit branched away by a
   `tbnz` (`and w16,w15,#7; tbnz w15,#2,out; ldrsw; add x18; br x18`: 4 entries). Cat Quest `main+0x99e180` stopped the rewrite
   with `jump table 99e180`. Tried only after every other form fails, so other games rewrite as before. Upstream has since replaced
   these recognizers with a general solver (`jump_solver.py`), too large to take onto this base.
4. `dm/gfx/tools/unityfs.py`: Unity 5.6.0b11's Shader type hash (`a70f7abc…`) reads with the 5.5 layout. AssetRipper's
   5.5.0f3 and 5.6.0b11 trees differ only by a plain string for `FastPropertyName { string }` and `int` for `unsigned int`
   `m_SourceMap`, the same bytes; Cat Quest's 5 built-in Shader objects read to their exact ends (35 programs). Before, the shader
   split stopped on `unity default resources` (no type trees).

## Checked on macOS 15

- Compiling the host with `MACOS_MIN=15.0` reports 313 uses of macOS 26 API (`-Wunguarded-availability-new`). Enum
  constants, type names and weak-linked Metal 4 classes (nil on macOS 15) are harmless. The messages that would fail
  on macOS 15 are not reached under `DM_METAL3`:
  - `MTLHeapDescriptor.maxCompatiblePlacementSparsePageSize`, `MTLTextureDescriptor.placementSparsePageSize`:
    only placement-sparse textures, which `sparse_make` never creates under `DM_METAL3`.
  - `CAMetalLayer.residencySet`: compiled out under `DM_METAL3`.
  - Metal 4 compiler, pipeline archives, pipeline data set serializer, counter heaps, `queryTimestampFrequency`:
    behind `pso4_on()`, `gp_active()` and `seg_ts()`, which return 0 under `DM_METAL3`; the `NX3Device` proxy does
    not answer these selectors, so `respondsToSelector:` checks also fail.
- `MTLSamplerDescriptor` `setLodBias:` and `setReductionMode:` are public only from macOS 26, but macOS 15.8 answers
  both (checked with `respondsToSelector:` on an M1), so sampler creation is safe.

## Known limits

- Placement-sparse textures use the whole-texture fallback (as on iOS 18); GPU-written sparse aliases are unverified.
- No Metal 4 pipeline archives: pipelines are built through the device at run time, so first encounters can hitch.
- No GPU timestamps / `DM_NVN_GPUPROF`.
- The build's pipeline-archive step (`pso_offline`, a Metal 4 feature) fails under Xcode 26 with `air-tt command failed`; the
  app is packed without an archive, which the Metal 3 backend does not use anyway.
- MetalFX uses the classic spatial and temporal scalers.
- Once, on an 8 GB M1, the whole Mac stopped responding (only the cursor moved) within half a minute after Cat Quest quit
  cleanly (exit 0), and needed a forced restart. The cause is unknown: the system log lost its last lines in the restart and
  no panic or hang report was written; memory pressure was high during the run. If it happens, press
  Shift+Control+Option+Command+. (period) and wait 2–3 minutes before restarting: the sysdiagnose in `/private/var/tmp/`
  names what was stuck.
- Upstream `main` cannot be merged in: it requires Xcode 27.2 and moves quickly. This branch stays on its base and
  takes fixes selectively.

## Credits

Metal 3 backend: dreampiggy ([nxnative-iOS18](https://github.com/dreampiggy/nxnative-iOS18)), from
Llllll07's [nxnative-ios18-check](https://github.com/Llllll07/nxnative-ios18-check). nxnative: seedds and
contributors. The TotK Metal 3 port in 1918662410-ship-it/MacNativeGames (`m3-macos15`) was a design reference.
