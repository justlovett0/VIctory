# VIctory — GTA VI on Windows: fork state & handoff

A working integration branch that boots Grand Theft Auto VI (Xbox Series dump) into a
**running, controllable world**, rendering scene geometry in a debug G-buffer view. The final
lit image is still black (lighting passes untranslated). This document is a handoff for another
agent to continue the work.

**Reality check on rate/stability (read first):** the copy-storm fixes removed the *copy-storm*
TDR and the log meter reads ~20 fps, but the **user's actual presented view is ~1 fps**
(1300–1800 ms/frame). VIctory waits for the GPU to finish *every* submission synchronously
(`host.rs` `WaitForSingleObject … INFINITE`) and the diagnostics probe maps a readback each
present — the log meter counts a different cadence than the screen. Also, a **late ~3.5-minute
world TDR remains** (issue **#1117**, hanging compute `9146291b38ea9ce1`); mitigate with
`VICTORY_SKIP_PIPELINES=9146291b38ea9ce1`. Neither is a quick fix.

Repository: https://github.com/marvisak/VIctory (GPL-3.0-or-later). Rust workspace,
pinned toolchain **1.98.1** (`rust-toolchain.toml`), Windows-only.

---

## 1. Machine / environment (where this was measured)

- Windows 10 21H2, build **19044.7846** (x64)
- NVIDIA **RTX 3080 10 GB**, driver **621.24** (32.0.16.2124) — wave32, mesh shaders, SM 6.6 via Agility
- AMD Ryzen 9 7900X (AVX2), 32 GB RAM
- Rust 1.98.1 `stable-x86_64-pc-windows-msvc`; DXC 1.8.2502 (Windows SDK 10.0.26100)
- Game dump: `D:\- GTA VI\GTA6.Xbox.v1.0.2.1-CYBERLEEK` (unpacked, user-owned), exe
  `game_scarlett_master.exe`, title id `7952987A`, build BLAKE3 prefix `b99f876b74d4a77f`, v1.0.2.1.
- **Windows 10 needs the D3D12 Agility SDK** (in-box D3D12 is SM 6.5). PR #175 supplies it.

All work is on the **D drive** (user rule). Game folder is never modified.

---

## 2. Repo layout / branches

Main clone: `D:\- GTA VI\VIctory` (on `main`). Worktrees:
- **`D:\- GTA VI\VIctory-main175`** — branch `main175`, THE working stack (this document). HEAD **`ac2abbc`** (= `ae21d59` + **#1276**).
- Branch **`park-233-wave64`** (`72caa03`) — the #233 wave64-split merge, **PARKED: it regressed** (early TDR before the cutscene; see §6.9). Recoverable here; do not re-stack bare.
- `D:\- GTA VI\VIctory-175`, `-346`, `-334`, `-233`, `-stack` — earlier experiment stacks (superseded).
- Main clone branch `local-placeholder-tokens` — disproven token/PSO experiments (kept for reference).

`victory.toml` (git-ignored, per-worktree) points `game_dir` at the dump and `cache_dir`/`data_dir`
under `D:\- GTA VI\work\victory\`. **Run from the worktree dir** — config resolves from cwd.

---

## 3. The stack (base + PRs, in merge order)

Base: upstream `main` **`ffe310e`** ("Shaders: the console's waterfall loop over a first-lane table offset").

| PR | Head used | What it does | Why it's here |
|---|---|---|---|
| **#175** | `19aec59` | Agility SDK loader; `cargo xtask agility` downloads SDK 1.620.2 into `target\<profile>\D3D12\`; exe exports `D3D12SDKVersion`/`D3D12SDKPath` | **Required to launch on Win10** (SM 6.6). Without it: init crash at title+0xb5a5f8 |
| **#994** | `993dff8` | Proton support + **two world GPU-fault fixes** (mirror heap-slot re-sync `6f732b6`; bounded root-SRV one-view tables `c69bd9a`); `VICTORY_DEBUG_PS`, `VICTORY_SHOW_TARGET`, debug PS fallback | World runs instead of immediate page-fault TDR; debug views |
| **#747 (local constant)** | — | `descriptor/host.rs`: capped-heap dynamic pool kept at `DYNAMIC` (2^18) not `MIN_DYNAMIC` (2^16) | Fresh world tables need the slots; see §6 gotcha |
| **#302** | `815178d` | Pre-SSA rewrite: compute root args spilled to memory (layout all ≥ user SGPR 16) read via `s[0:1]` table | Helps some computes; **does NOT clear XboxBvhBuilder** (see §7) |
| **#193** | `f99b792` | Runtime stride table (`t0`, space 2) for structured buffers whose stride isn't statically known | Clears "unknown stride" class |
| **#704** | `1f3c184` | Raw-view alias slots for unknown-stride buffers (stacked on #193) | **NVIDIA fix**: raw reads return 0 against structured views without this |
| **#309** | `64c7188` | Forward SGPRs reloaded from VGPR lanes as the spilled value (`sgpr_spill` module) | Clears lane-mask classes; see §6 boot-stall gotcha |
| **#1056** | (merged) | Scalar loads from a root view's address | **Gains ~0** (superseded by #1067); harmless, can drop |
| **#953** | `c42cbb0` | `s_buffer_load` SMEM uses in `descriptor_data` | Clears "scalar memory read as data" |
| **#1067** | `6d30fd7` | "A descriptor only read as data takes its kind from its table" | **The big one**: clears "descriptor read only as data" (303 runtime) |
| **#986** | `10706d2` | Aligned SRV scratch copies | Copy-storm fix (fps + hang) |
| **#1260** | (on #986) | One aligned copy per unaligned SRV range per list | Copy-storm fix |
| **#1267** | `4cefaac` | Copy-back: each declared byte once per submission, shared block | Copy-storm fix (174 MB heap re-copy) |
| **#1254** | (2 commits) | Black-frame trace keeps the draws that END the frame (lighting/composite) | Diagnostic for the black lit image |
| **#1276** | `b95831e` | Scalar loads from a code table at a first-lane address (`s_getpc` + readfirstlane + `s_load`) | Clears "address not from a root parameter" (29); slightly more world geometry. **In the current HEAD** |

**#233 (wave64 split) was merged then PARKED** — see §6.9. It is NOT in the working stack.

Local additions (not upstream PRs):
- **Refusal dumps**: `VICTORY_DUMP_REFUSALS=1` (diagnostics feature) writes each failing runtime
  container to `<cache>/shaders/refusals/<stage>-<hash>.bin` (`shader_cache.rs::dump_refused`,
  called from `pipeline.rs`). Enables offline census of the *actual* runtime shaders.
- **Census path parity**: `apps/victory/src/translate.rs` applies the same `spilled` rewrite +
  `sgpr_spill::forward` as the runtime, so `victory translate` reflects #302/#309.

Fetch a PR head: `git fetch origin pull/<N>/head:pr-<N>` then `git merge pr-<N>`.

---

## 4. Build & run

```bash
cd "D:\- GTA VI\VIctory-main175"
cargo xtask agility                              # SDK into target\{debug,release,diag}\D3D12
cargo build --release -p victory --features diagnostics
# run (from this dir so victory.toml resolves):
VICTORY_DEBUG_PS=1 VICTORY_SHOW_TARGET=2560x1440-f28-3 VICTORY_DUMP_REFUSALS=1 \
  ./target/release/victory.exe run "D:\- GTA VI\GTA6.Xbox.v1.0.2.1-CYBERLEEK\game_scarlett_master.exe" --execute
```

- `--features diagnostics` enables `VICTORY_SHOW_TARGET`, `VICTORY_DUMP_REFUSALS`, draw dumps.
- `VICTORY_DEBUG_PS=1`: slope-coloured debug pixel shader on depth-tested geometry (works in plain release).
- `VICTORY_SHOW_TARGET=<w>x<h>-f<fmt>[-<n>]`: present an intermediate target instead of the final frame.
  `2560x1440-f28-3` = the world G-buffer (game renders internally at 1440p, DXGI format 28 = RGBA8, 3rd target).
- `VICTORY_BLACK_FRAME_DRAWS=<n>` (#1254): how many frame-end draws to trace on a black frame.
- Keyboard acts as a pad (Enter=A, WASD=left stick, mouse=right stick, Esc=Menu). A virtual controller
  works (user runs remote via Parsec).
- Logs: stdout redirected to `D:\- GTA VI\work\victory\<name>.log`; title data/crashes under
  `D:\- GTA VI\work\victory\data\7952987A\`.

**Offline census** (no game launch): `./target/release/victory.exe translate <path>` — accepts
`shaders.rpf` (26,503 containers) or the refusals dir. `--host-lanes` was REMOVED on new main
(auto-detects host wave size). Ranked failure tally + emitted counts.

---

## 5. Current state (measured 2026-11-24, full stack above)

- Boot, first-boot screens, intro cutscenes, menus: **work** (log ~30 fps).
- World: **runs and is controllable for ~3.5 min**, then the **#1117 late TDR** (compute
  `9146291b38ea9ce1`, 28-list submission). `VICTORY_SKIP_PIPELINES=9146291b38ea9ce1` mitigates
  (#1117 reports doubled run length). The copy-storm TDR is gone; this is a *different* hang.
- **Presented rate ~1 fps** (1300–1800 ms/frame) even though the log meter reads ~20 fps — see the
  reality check at the top. Synchronous submit + per-frame probe readback gate the screen.
- Copies: **~0.4–1 MB / frame** (was 270–400 MB) — copy-storm fixes holding.
- In-game untranslated shaders: **~100** (was 444 before #1067; ~466 unique captured). #1276 adds
  slightly more world geometry (the 29 address-not-from-root class).
- **Picture**: G-buffer shows scene geometry (terrain, characters, props) in flat debug colours —
  textureless, low-poly. **Final lit image: black** (lighting/composite passes untranslated).
- ~119 compute pipelines still refused (wave64 on wave32 host). **#233's split hangs on NVIDIA
  (§6.9); use #189 `VICTORY_WAVE64_EMULATE=emulate` instead** (per #1086's working Win10 config).
- Exit on close: teardown segfault (exit 139) — not a world fault. A *world* TDR is exit 5.

### Remaining runtime failure classes (census of the 100)
| Count | Class | Candidate fix |
|---|---|---|
| 40 | a descriptor's address or format read as data | scalar-load family #1055/#1015/#995 |
| 29 | an address not from a root parameter | **#1276** (s_getpc table + readfirstlane + s_load) |
| 11 | an address from a phi | #1253 / phi tracking |
| 9 | image_gather4_l | **#160 / #563 / #694** |
| 8 | a root UAV or typed root view | root-view handling |
| 7 | a lane mask read as data | #585 / #309-adjacent |
| 4 | an address in an SGPR that is no root parameter | **#302 detection defect — XboxBvhBuilder (lighting key)** |

---

## 6. Gotchas / hard-won lessons (READ BEFORE MERGING)

1. **#309 boot stall**: the wave-form analysis in `lower.rs` (`console_lanes`/`split`/`wants_console_flow`)
   MUST use `insts()` (every instruction), NOT a dead-spill-filtered iterator. Filtering resolved spill
   reloads picks wrong wave forms → **title stalls at boot** (black, no audio, ~8 s in). Lowering still
   skips dead spills via `self.dead`. We bisected this; don't reintroduce it.
2. **Stale exe**: if `cargo build` fails, the launch reuses the OLD binary silently. Always confirm
   `Finished` before launching; a failed build means the test didn't test your change.
3. **Nested merge markers**: merging #193 then #704 (both rework `descriptor/host.rs`, `draw.rs`) left
   nested `<<<<<<<`/`=======` markers that `git` reported "clean" but don't compile. After each merge:
   `grep -rn '^<<<<<<<\|^>>>>>>>' crates apps --include='*.rs'` and `cargo check --workspace`.
4. **`descriptor_data` semantic conflict**: #953/#1056 match on `ui.fields` (Image/Buffer/Smem);
   #1067 restructures to `match use_at { Some/None }`. When merging #1067, #953's `Fields::Smem` arm
   must live INSIDE the `Some` arm's inner `match &ui.fields`, not after the outer match.
5. **#1056 vs #1067**: #1056 gains ~0 on "descriptor read only as data" (confirmed by #1247 benchmark
   and our runtime). **#1067 is the real fix** (394→17 offline, 444→100 runtime).
6. **Capped heap split** (#747 + #704): NVIDIA caps shader-visible heaps at 1,000,000. Title writes
   ~1.1M slots. Split = mirrored title slots + dynamic pool + alias region. With 2^18 dynamic + 2^16
   aliases → 672,320 mirrored. High bindless indices still read the guard slot (needs #747-part-2:
   fresh tables reserve from first used entry; + index clamping #955/#584/#483).
7. **SM 6.6 in tests**: `cargo test -p victory-shader` has ~15 environmental failures ("no SM 6.6
   device") because test binaries don't export the Agility symbols. Not regressions. ~108 pass.
8. **Asset policy** (`.githooks/pre-commit`, `cargo xtask policy`): never commit game files/dumps/SDK
   binaries. Refusal dumps live in the cache dir (outside repo), fine.
9. **#233 wave64 split HANGS on NVIDIA — do not stack it bare.** Merging #233 (`72caa03`) caused an
   **early TDR before the cutscene** (~96 s, mesh-heavy 8-list submission, `DXGI_ERROR_DEVICE_HUNG`),
   and many wave64 computes were *still* refused (the console flow refuses per-lane-descriptor
   shaders). Removing #233 (`ac2abbc`, #1276-only) restored a stable ~3.5-min world run. This
   **re-confirms** the earlier `-233` worktree finding (nvgpucomp64 AV + 7-list mesh TDR on this
   RTX 3080). The split is parked on `park-233-wave64`. For wave64, prefer **#189
   `VICTORY_WAVE64_EMULATE=emulate`** (#1086's working Win10 config) or the fork fixes (gangg111
   root-SRV windows / Peppacodes 64 MB worker stacks) *before* attempting the split. **Lesson: check
   known findings before re-stacking a PR that already failed on this hardware.**
10. **Log fps ≠ perceived fps.** The ~20 fps log meter is not what the user sees (~1 fps). Every GPU
   submission is waited on synchronously (`host.rs` `WaitForSingleObject … INFINITE`) and the
   diagnostics probe maps a readback each present (`capture::black`). Don't claim a fps win from the
   meter; it measures a different cadence.

---

## 7. The lighting frontier (why the final image is black)

The G-buffer is written; the lit composite is not. Two known blockers:
- **XboxBvhBuilder.dll computes** fail "an address in an SGPR that is no root parameter"
  (`3b90165a77d95cdf`, `f6e47efa50226beb`, ~4,500 dropped dispatches/frame per #844). No BVH → no
  ray-traced lighting. **#302's `spilled` detection does not fire for these** (their layout/pattern
  differs). Dumped containers available in the refusals dir. Fixing the detection is high-impact.
- **Lighting/composite pixel passes** untranslated. Use #1254's frame-end black-frame trace
  (`VICTORY_BLACK_FRAME_DRAWS`) to name them. Texture sampling variants: #1255/#1280/#1237.

---

## 8. Next ideas (ranked)

1. **Stability first: `VICTORY_SKIP_PIPELINES=9146291b38ea9ce1`** (#1117) to get past the ~3.5-min
   world TDR — currently being tested. Lets the user explore longer and give richer feedback.
2. **Wave64 the SAFE way: #189 `VICTORY_WAVE64_EMULATE=emulate`** (per #1086's working Win10 config),
   NOT #233's split (known NVIDIA hang, §6.9). Clears the ~119 wave64 refusals (lighting/BVH related).
3. **Fix #302 detection for XboxBvhBuilder** (lighting key, 4 SGPR-no-root shaders
   `3b90165a77d95cdf`/`f6e47efa50226beb`) — biggest visual payoff (lit image). #302 family gains 0
   offline per #1247, so needs a *different* approach; investigate the dumped containers.
4. **#1253** (waterfall offset tracking) + **#160 / #563 / #694** (image_gather4_l, 9) — small/targeted;
   #1247 benchmarks the image-sampling overlap.
5. **#747 part 2** (first-used-entry table reservation) + index clamping (#955/#584/#483) — high
   bindless slots read the guard slot.
6. **Texture/sampling passes** (#1255/#1280/#1237) + the lit composite — the final-image frontier.
7. **#233 split — PARKED, do not re-stack bare** (§6.9). Only revisit with the fork fixes
   (gangg111 root-SRV windows / Peppacodes 64 MB worker stacks) layered first.

Useful community data sources: **#1247** (25-PR offline benchmark), **#293** (runtime drops/frame→PR
map), **#1086** (working 12 GB Win10 world config), **#844** (#994 tracking, tester reports),
**#49** (tester specs/results), **#71** (hang investigation).

---

## 9. Upstream findings posted (2026-11-24)

Verified-results comments on: #986 (fps/hang), #1260, #1267, #1067 (descriptor class), #994 (extended),
#193 + #704 (stride/NVIDIA), #309 (masks + boot-stall merge note), #302 (XboxBvhBuilder detection gap).
Earlier: #348 (closed, dup of #84 — the Win10 init crash), #747 (heap data).

AI use throughout: Qwen Code (underlying model name not disclosed by the tool). All measurements from
local runs; visual confirmations by the user. No game data posted or committed.
