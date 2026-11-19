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
| NVN games (the Metal 3 replay itself) | **not tested on a Mac yet**; on iOS 18 the same backend ran Kirby's Return to Dream Land Deluxe |
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
- MetalFX uses the classic spatial and temporal scalers.
- Upstream `main` cannot be merged in: it requires Xcode 27.2 and moves quickly. This branch stays on its base and
  takes fixes selectively.

## Credits

Metal 3 backend: dreampiggy ([nxnative-iOS18](https://github.com/dreampiggy/nxnative-iOS18)), from
Llllll07's [nxnative-ios18-check](https://github.com/Llllll07/nxnative-ios18-check). nxnative: seedds and
contributors. The TotK Metal 3 port in 1918662410-ship-it/MacNativeGames (`m3-macos15`) was a design reference.
