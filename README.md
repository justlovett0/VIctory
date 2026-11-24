# VIctory

Runs Grand Theft Auto VI (Xbox Series X|S) natively on Windows, from files
the user supplies.

The title's x86-64 code runs unchanged inside our process; VIctory supplies
the platform underneath it and converts the title's data, ahead of time where
it can and on first use where it must (shaders). See
[docs/architecture.md](docs/architecture.md).

## This fork (`justlovett0`, branch `main175`)

An integration branch stacking open community PRs onto upstream `main`
(`ffe310e`), verified against the real game on **Windows 10 21H2 / NVIDIA RTX
3080 10 GB / driver 621.24**. Full handoff: [fork-details.md](fork-details.md).

What it adds over upstream `main`:

| Change | PR(s) | Effect |
|--------|-------|--------|
| D3D12 Agility SDK loader | #175 | **Required to launch on Windows 10** (in-box D3D12 is SM 6.5); `cargo xtask agility` |
| World GPU-fault fixes + debug views | #994 | Mirror heap-slot re-sync, bounded root-SRV tables; `VICTORY_DEBUG_PS`, `VICTORY_SHOW_TARGET` |
| Capped view-heap split | #747 (local) | Keeps the full dynamic pool on NVIDIA's 1M-capped shader-visible heap |
| Shader translation classes | #302 #193 #704 #309 #953 #1067 #1276 | Spill-table root args, runtime stride + raw aliases, mask spills, scalar/descriptor-as-data, address-not-from-root |
| Copy-storm fixes | #986 #1260 #1267 | Aligned SRV scratch copies, one copy per range, copy-back each byte once (copies ~400 MB → ~1 MB/frame) |
| Black-frame trace | #1254 | Keeps the frame-end draws (lighting/composite) for diagnosis |
| **Wave64 compute emulation** | #189 | **Cures the world-entry GPU hang**: wave64 refusals 119 → 0, world ran ~9 min with 0 device removals (was a ~3.5-min TDR). Auto-selects `Emulate` on wave32 hosts; `VICTORY_WAVE64_EMULATE` overrides |
| Instruction lowering | #1084 #305 #306 #176 #587 | Untracked-mask lane bit, 64-bit image atomics, biased 16-bit samples + conservative depth, packed fp16 inputs, mesh output bounds (cherry-picked from the #1492 bundle) |
| Buffer descriptor stride as data | local | Rebuilds GFX10 buffer WORD1 stride (bits [29:16]) when a descriptor is read as data |

Measured: **494/561 compute programs emit** (was ~313/381 before wave64
emulation); the world is stable and controllable; scene geometry renders in the
debug G-buffer view. **The final lit image is still black** (lighting/composite
passes untranslated) and the presented rate is ~1 fps (synchronous submission +
emulation copy overhead). The remaining frontier is the descriptor
address/format-as-data class and the lighting passes.

Findings are reported back to the upstream PRs as verified-results comments.
AI-assisted (Qwen Code); no game files, keys or dumps are committed.

## Status (2026-11-23)

**Not playable yet.** The title boots, runs its menus at its own 30 fps and
loads the world, where rendering fails.

| Stage | State |
|-------|-------|
| Boot, first-boot screens (accessibility, language, settings, legal) | works, 30 fps |
| Intro video with sound | works |
| Start screen, profile and saves | works |
| Loading into the world (about 50 s) | works |
| The world (first mission's intro) | reached and runs with indirect draws; its title card renders, the scene after it is still dark; 0.5-1.8 s per frame |

Tested on one machine: Windows 11, AMD RX 6700 XT; and on Linux under
Proton Experimental: CachyOS, AMD RX 9070 XT (see
[docs/linux-proton.md](docs/linux-proton.md)).

> **Warning.** Entering the world on a real GPU can currently hang it, and
> Windows then resets the display driver (every monitor goes black for a
> moment). To look further, use a diagnostics build with
> `VICTORY_D3D12_WARP=1`, which renders on the software adapter.

What each part does today and what it still needs:

| Subsystem | Today | Needs |
|-----------|-------|-------|
| Loader | Runs the title's executable and its DLLs in our process; crash reports name the faulting code and are kept in `crashes.log` | – |
| Runtime services | One signed-in offline user, licences, packages, saves that survive a kill at any step, text entry dialog | Speech is silent; the dialog and relaunch are untested live |
| Network | Offline by design: the title cannot reach the network | – |
| Files and streaming | Archives and streamed data read, zlib and BCPack decoded in parallel | Loading is limited by writes into title memory |
| Input | XInput pads, keyboard, keyboard stand-in pad | Trigger rumble |
| Audio | Menu, video and loading sound; XMA voices and Opus streams decode | Reverb is silent; Opus results unverified against the title |
| GPU: menus | Draws, compute and presentation on host D3D12; only changed title memory is copied | Frame pacing drifts on one screen in release builds |
| GPU: world | Mesh pipelines, shader-written buffers, descriptor lifetimes, indirect execution (arguments clamped), pipelines made by worker threads and kept on disk | Most of the scene is still missing (untranslated shaders drop draws); frames take seconds; real queries, depth bounds, HDR and scaling are missing |
| Shaders | Console GPU code recompiled to DXIL at first use, cached on disk. Menus: all. World: mesh 8,191 of 8,506, vertex/pixel 10,617 of 12,346, compute 1,643 of 2,539 | 64-bit image atomics, merged geometry stages, the ray-tracing builder's shaders; some compute programs need a GPU with 64-lane waves |
| Textures | Console tiled layouts and detiling for the modes the title uses | Render-target metadata planes |
| Series S mode | Runs through the menus to the loading screen | The world, as above |

## Getting started

Requirements: Windows 10/11 x64, a CPU with AVX2, a Direct3D 12 GPU with
shader model 6.6 (and mesh shaders for the world), Rust (pinned in
`rust-toolchain.toml`).

On Windows 10 the in-box D3D12 runtime stops at shader model 6.5, so the host
refuses every translated pipeline. `cargo xtask agility` fetches Microsoft's
D3D12 Agility SDK runtime (a pinned, hash-checked NuGet package) into
`target\<profile>\D3D12\`, where `victory.exe` makes D3D12 load it. Windows 11
needs nothing.

The title also needs several GiB of available Windows commit at startup.
Check **Committed** under **Task Manager → Performance → Memory**. If
`crashes.log` reports Windows error 1455, close other applications or increase
the Windows paging file size, then retry.

```powershell
cargo xtask install-hooks                # asset-policy check on every commit
copy victory.example.toml victory.toml   # then set your paths (outside the repo)
cargo xtask agility                      # Windows 10 only: newer D3D12 runtime
cargo test --workspace                   # needs no game data
cargo run -p victory -- config           # check your paths
```

Run the title:

```powershell
cargo build --release -p victory
target\release\victory.exe run <game dir>\<the title's exe> --execute
```

The window caption shows the framerate. Saves, settings and `crashes.log`
are kept under the data folder (default `%LOCALAPPDATA%\victory\data`), derived
data under the cache folder.

Input: XInput pads. With none connected the keyboard is a pad while the
title's window has focus:

| Keys | Pad |
|------|-----|
| W A S D / I J K L | left / right stick |
| arrows | D-pad |
| Enter, Backspace, X, Y | A, B, X, Y |
| Q / E, Z / C | shoulders, triggers |
| F / H | stick clicks |
| Esc, Tab | Menu, View |

### Linux (Proton)

VIctory also runs on Linux under Proton Experimental, built with
`cargo-xwin`, and gets as far there as on Windows: see
[docs/linux-proton.md](docs/linux-proton.md) and `scripts/proton-run.sh`.

## Commands

```
victory inspect <exe|dll> [--imports] [--exports]
victory census [paths...] [--missing] [--strict]
victory lib-exports <import.lib> [--list]
victory shader-info [paths...] [--verbose] [--disasm]
victory translate [paths...] [--validate] [--pipelines] [--list] [--jobs N]
victory run <exe> [--execute] [--unimplemented trap|zero] [--strict]
            [--console series-x|series-s] [-- <title arguments>]
victory config
```

Logging: `VICTORY_LOG=victory_loader=trace,info` (a release build logs
info and above).

## Debugging

A release build has no debugging aids. Build them in with

```powershell
cargo build --profile diag -p victory --features diagnostics   # target\diag\victory.exe
```

| Variable | Effect |
|----------|--------|
| `VICTORY_LOG=<target>=debug,...` | debug and trace output |
| `VICTORY_STACKS=20,60` | every thread's stack at those seconds |
| `VICTORY_WATCH=<rva>,...` | logs a title u32 when it changes |
| `VICTORY_KEYS=30:Enter,35:X` | presses keys at those seconds, focus or not |
| `VICTORY_CRASH_SAVE=<rva>:<len>,...` | saves title memory to the cache folder on a crash |
| `VICTORY_CAPTURE=<dir>` | saves presented frames (powers of two, or `VICTORY_CAPTURE_FRAMES=<list>`) |
| `VICTORY_DUMP_TARGETS=1` | with the two above, saves every render target of those frames |
| `VICTORY_D3D12_DEBUG=1` | host debug layer |
| `VICTORY_D3D12_GBV=1` | with the above, GPU-based validation (bad descriptor use reported, not faulted; slow) |
| `VICTORY_D3D12_DRED=1` | on a GPU failure, the allocation hit and the commands in flight |
| `VICTORY_BLACK_FRAME_DRAWS=<n>` | with `VICTORY_LOG=info,victory_gpu::draw=debug`, a black frame's trace keeps its first and last `n/2` draws (default 256), so the passes that end the frame are in it |
| `VICTORY_SKIP_PIPELINES=<hash>,...` | omit draws and dispatches, including indirect ones, whose pipeline contains a listed 16-digit shader hash |
| `VICTORY_GPU_TIMING=1` | with the above, each list of a submission run and timed on its own; slow lists logged with their slowest commands |
| `VICTORY_D3D12_WARP=1` | host device on the software adapter (slow; cannot reset the driver) |
| `VICTORY_SHADER_LOOP_LIMIT=<n>` | translated shader loops stop after `n` iterations per lane instead of 2^20 (own shader cache entries); if a lower `n` makes world frames much faster, loops hitting the limit are what makes them slow |
| `VICTORY_AUDIO_CAPTURE=<file.wav>` | records what the title plays (our process only) |
| `VICTORY_XMA_DUMP=<dir>` | saves the first 32 XMA voices' context and input |

Every build appends crashes and host GPU failures to `crashes.log` in the
title's data folder.

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).
