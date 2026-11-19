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
| Rune Dice (`0100F5102672C000`, Unity, update v327680 / 1.3.0, provisional profile) | works: played for about 45 minutes at a steady 60 fps (frame time p50 16.67 ms, GPU 1.5–9 ms a frame by scene, single load frames of 100–180 ms), host process about 320 MB. No code fixes needed (3,655 shaders, Mac app 0.8 GB, build about 11 minutes); the pipeline archive fails with `air-tt` as on the other games. The mouse is the Switch touch screen and, as on a docked Switch, only works in **Handheld** console mode; the game plays from the keyboard or a controller |
| Cat Quest III (`010088501B8F2000`, Unity, update v983040 / 1.4.7, provisional profile) | works: steady 60 fps with `DM_PRESENT_INTERVAL=display` in its `env.txt` (frame time p50 16.67 ms, 0 dropped frames, GPU about 8 ms a frame, footprint about 2.3 GB); the game itself asks for present interval 2 (30 fps), and the user confirmed it plays at normal speed at 60. Built in 7 minutes (3,096 shaders, Mac app 1.2 GB, work folder 2.9 GB) with no code fixes. It flickered with `DM_M3_DECLARE=set` alone and not with `rw`, which confirmed the cause below; the fence chain (`DM_M3_FENCE`, below) fixed it with the speed of `set` |
| Hades (`0100535012974000`, Vulkan, update v589824 / 1.0.38233) | works: the first Vulkan game under `DM_METAL3=1`. Played from the title through the intro into the game at a steady 60 fps, memory footprint about 2 GB, exited cleanly; screenshots match the expected picture. No code fixes needed: the Vulkan layer (`dm/gfx/vk/hle_vk.m`) already records into plain Metal command buffers and presents through the shared external-texture path (Mac app 3.9 GB, build about 1.5 minutes, 74 SPIR-V shaders, no pipeline-archive step). One `[RED]` line, `dynamic state 3 not implemented`: the game declares depth bias dynamic but never imports `vkCmdSetDepthBias`, with no visible effect. The first open waits about 30 s for XProtect to scan the bundle |
| Hades II (`0100A00019DE0000`, NVN, update v655360 / 1.0.141667) | works: a steady 60 fps on an M1 Air (8 GB) at render scale 2.0 (2560×1440, picked in the settings window), checked in play by the user with the frame-rate counter (change 9). No code fixes needed: 88 shaders, built from its two NSZ files in about 4 minutes (2 of them converting the NSZ files), Mac app 4.7 GB, work folder 16 GB. At 1.5 (1920×1080) the picture looked slightly soft in fullscreen: with the display set to More Space (looks like 1680×1050) the drawable is 3360×2100, so the picture is scaled up bilinearly and macOS scales the result down again to the 2560×1600 panel. 2560×1440 matches the panel's width |
| The Legend of Zelda: Tears of the Kingdom (`0100F2C0115B6000`, NVN, update v720896 / 1.4.3) | shelved (2026-11-19): 30 fps outside the cave on an M1 Air at any render scale, and GPU page faults every minute or so (open). Steady 60 fps in the opening cave in handheld mode with `DM_OPMODE=0 DM_RENDER_SCALE=0.75 DM_PRESENT_UPSCALE=metalfx` in its `env.txt` (GPU about 8 ms a frame), no flicker (checked by the user in play). Outside the cave it drops to 30 fps and below: GPU 14–16 ms at 0.75 and the replay's CPU time about 4 µs a draw, up to 3,900 draws a frame; `DM_RENDER_SCALE_AUTO=0.5-0.9` (automatic scale for the GPU side) replaces the fixed scale in its `env.txt`; in play outside it sat at 0.5 most of the time and the GPU still took 13–20 ms a frame, with the replay's submit at 10–16 ms: a steady 30 fps, CPU-bound. GPU page faults: one froze the picture after 11.5 minutes of play; a later run crashed at the first command buffer error (change 8); with that fixed, a 2.5 minute run outside went on through 5 page-faulted command buffers (open, see the game's README). At render scale 1.0: mostly 60 fps, GPU about 10.5 ms, with drops to 30 and 20 fps whose late frames are CPU time, not GPU. First run: 5 fps, GPU 90–190 ms a frame; half resolution in handheld mode only took 16% off, so not resolution-bound. Cause: the Metal 3 adapter re-declared all ~2,150 registered allocations read+write with `useResources` on nearly every encoder (about 780 times a second), which serialized the GPU. With one residency set on the queue instead (`DM_M3_DECLARE=set`, below): the intro at a steady 60 fps (GPU about 10 ms), the opening cave in handheld mode (1280×720, `DM_OPMODE=0` in its `env.txt`) at 30–60 fps (GPU 13–22 ms; the frame step holds 30 when frames run late), no `[RED]` lines, picture correct in screenshots. The user saw the image flicker in play: the write hazards on argument-buffer pointees that the residency set no longer declares (confirmed on Cat Quest III: it flickered with `set`, not with `rw`). The fence chain (`DM_M3_FENCE`, below) fixed Cat Quest III's flicker but not this game's: its texture uploads go through a second command buffer, which one global chain interleaved with the frame's; one chain per command buffer plus a commit boundary fence (change 7) fixed it. With the fence chain off the GPU took 12.6–15.7 ms against 10.5 with it on. 28,669 shaders compiled in 1 h 10 min on the M1 (build about 1 h 20 min, Mac app 19.4 GB, work folder 28 GB); built with `--gpu none --no-prebake` (the pipeline archive fails on this Mac, and its compiler is documented at about 36 GB). The first open waits about 2.5 minutes for XProtect. The game pauses whenever its window is not key (the user saw it freeze on an external screen in fullscreen when focus moved to the other screen); `DM_FOCUS_KEEP=1` in its `env.txt` keeps it running (controller input still stops until the window is key again), and a run started from a shell needs `DM_FOCUS_KEEP=1 DM_WINDOW_TOP=1`. Memory footprint 2.5–2.8 GB |

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
- Before extracting, the script checks for the extracted game's size plus 6 GB free on the build folder's volume and
  stops with `less than N GB free` otherwise (Hades II: 17 GB). It picks up where it stopped once there is room.

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

5. `dm/gfx/nvn/nvn_metal3_compat.inc`: `DM_M3_DECLARE`, how the Metal 3 adapter keeps argument-buffer pointees resident. `set` (default): one `MTLResidencySet` with every registered allocation, attached to the replay queue and rebuilt at a command-buffer commit only when the registry changed; encoders no longer call `useResources`. `rw`: the adapter's original behaviour, every registered allocation declared read+write on each encoder (each new encoder starts with nothing declared, so in practice on every encoder). `r` and `off` are for diagnosis only (`off` faults the GPU: black screen). A line every 10 s reports the declarations or the residency set's rebuilds and size. Tears of the Kingdom went from 5 fps to 30–60; the other NVN games above were tested before this change and need rechecking with it.
6. `dm/gfx/nvn/nvn_metal3_compat.inc`: `DM_M3_FENCE` (on by default with `DM_M3_DECLARE=set`; `0` turns it off). Without per-encoder declarations Metal no longer orders an encoder that writes a resource only through an argument buffer before one that reads it, which showed as flicker. The replay already issues MTL4 queue barriers at every encoder start (`barrierAfterQueueStages:`) and end (`barrierAfterStages:beforeQueueStages:`), which the adapter ignored; they now drive a chain of two alternating `MTLFence`s: each encoder waits on the fence the previous encoder updated, before the stages the barriers name, and updates the other one at its end (two fences so no encoder waits on the fence it updates). Lazily opened compute and blit encoders, and the switch between them inside one replay encoder, take part in the chain; an encoder that ends without an end barrier updates it and orders everything after it. A line every 10 s counts updates and waits. Cat Quest III: no flicker, GPU time unchanged (about 8 ms a frame).
7. `dm/gfx/nvn/nvn_metal3_compat.inc`: one fence chain per command buffer (`NX3FenceChain`, kept on the native command buffer), and a commit boundary. The replay fills its upload command buffer (`up_begin` / `up_flush` in `nvn_mtl.m`) while the frame's is still open and commits it just before the frame's, after the frame's encoders that read the uploaded textures were encoded. A single global chain interleaved the two buffers' encoders in encoding order, so a frame encoder after an upload encoder waited only on the upload. Now each buffer chains its own encoders; at commit a closing blit encoder waits on the buffer's last chain fence and updates one boundary fence, and each buffer's first encoder waits on that boundary fence. Fence waits resolve in commit order, so the frame's buffer also waits for an upload buffer committed before it though encoded later. The 10 s line also counts waits on the boundary (about 4 a frame in Tears of the Kingdom). Tears of the Kingdom: no flicker, steady 60 fps at render scale 0.75, GPU unchanged (about 8 ms). Cat Quest III and the other NVN games have not been rerun with this change yet.

8. `dm/gfx/nvn/nvn_mtl.m` `gpu_err_has`: it compared an error's domain with `MTL4CommandQueueErrorDomain`, a macOS 26 symbol whose weak import is NULL on macOS 15, so the host crashed (`CRASH sig=11 addr=0 pc=gpu_err_has`) at the first command buffer error of any kind (in Tears of the Kingdom an innocent-victim error after a GPU fault). The Metal 4 domain is now read only under `@available(macOS 26.0, *)`, and on Metal 3 the revoked access is recognised as `MTLCommandBufferErrorDomain` / `MTLCommandBufferErrorAccessRevoked`, so `gpu_lost_check` stops the game with its message instead of leaving it frozen. A faulted command buffer now costs its frame's work, not the process.

9. `dm/host/front_mac.m` `front_mac_fps_label`: `DM_SHOW_FPS=1` (in `env.txt`) now shows a small frame-rate counter in the Mac game window's top-right corner, as the iPhone app does at its top left. It counts the frames put on screen each second (`mtl_present_count`, NVN and Vulkan alike) and stays in the corner when the window resizes or goes fullscreen. Apple's Metal performance HUD (`DM_METAL_HUD`) cannot be cut down to the frame rate on macOS 15: its `MTL_HUD_ALIGNMENT`, `MTL_HUD_OPACITY` and `MTL_HUD_SCALE_OVERRIDE` only move, fade or scale the whole panel.

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
- `DM_M3_DECLARE=set` (the default) keeps pointees resident but no longer declares their reads and writes; the order between encoders now comes only from the fence chains (`DM_M3_FENCE`), which follow the replay's queue barriers, and between command buffers only from the commit boundary fence (commit order). Command buffers made outside the adapter's command buffer wrapper are not in it. Within one render pass, draws that write through a pointee and draws that read it are ordered only by the replay's own intra-pass barriers.
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
