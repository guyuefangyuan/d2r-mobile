# Testing

[中文版 / Chinese](TESTING.zh-CN.md)

This repository publishes the compatibility framework only and **contains no game files**, so the
tests fall into three tiers:

| Tier | Needs your own game files | Needs an Android device | Typical time |
|---|---|---|---|
| **L0 host self-checks** | no | no | seconds |
| **L1 host + game files** | yes | no (Linux / WSL / Apple silicon Mac) | minutes |
| **L2 on device** | yes | yes | 5–15 min per device |

Read `NOTICE.md` first: the repository does not ship game files, you need to own and dump your own.

---

## L0 — host self-checks (start here)

### L0-a Packager unit tests

No game files, no device. Validates the packager's manifest, path, offline-package and overlay
checks:

```bash
# from the repository root
PYTHONPATH=dm/tools python -m unittest discover -s dm/tools/tests -p test_android_d2r_apk.py -v
```

Expected (measured on Windows + Python 3.12):

```
Ran 17 tests in 0.075s
OK (skipped=1)
```

`test_symlink_escape_fails` is skipped on Windows (the OS refuses to create symlinks) and really
runs on Linux / macOS.

### L0-b Two-device menu layout parity (added by this port)

The same APK must place the menu in the same spot on every device. The tool crops each screenshot
to its own 16:9 picture rectangle and resamples both to 1280×720, so different resolutions and
densities can be compared directly:

```bash
python dm/android/tests/compare_home_layout.py tablet.png phone.png --json result.json
```

Measured output:

```
left  pad.png   3200x2000  content 3200x1800 at 0,100  border [13, 13, 15]
right phone.png 2640x1216  content 2161x1216 at 299,0  border [13, 13, 15]
canvas 1280x720  mean diff 4.93  strong 0.99%  best shift 0px (residual 2.08)
PASS: the two devices lay the menu out the same way
```

Thresholds (change with `--max-mean` / `--max-diff-share`):

- `mean_absolute_difference ≤ 10.0` (0–255 scale)
- `strong_difference_percent ≤ 4%` (share of pixels differing by more than 60)

**What it guards.** If a device fails to load the touch-UI overlay, the game falls back to the
stock gamepad UI: the character panel moves to the other side and the tappable
"create new character" button becomes a ✚ key prompt. Measured in that state:
`mean diff = 14.66 / strong 3.81%` → **FAIL**; after the fix: `4.93 / 0.99%` → **PASS**.
The thresholds sit between those two numbers.

`--json` also reports `mean_difference_by_third` (left / centre / right thirds), which tells you
quickly which side of the picture the difference is on.

### L0-c Host tests that need Linux / WSL

These runners build and execute real C with ASan / UBSan and never touch a device:

```bash
python dm/android/tests/run_compat_checks.py  --output out/compat  --host --vulkan-include <dir containing vulkan/>
python dm/android/tests/run_translation_checks.py --output out/translate
python dm/android/tests/run_pipeline_checks.py --help     # plus the other run_*.py next to it
```

They fail on Windows because `clang` and the ASan runtime are incomplete there; that is expected.
Use `--help` on each script for the full parameter list.

### L0-d Known upstream failures (not an acceptance gate)

```bash
python dm/gfx/tests/test_tbdr_budget.py
```

Current result: **2 passed / 3 failed**.

- This repository already fixed two portability defects unrelated to the port: `tbdr_budget.py`
  split `__file__` on `'/'` to build a path (never works on Windows), and a few `open()` calls
  omitted the encoding (the header is Chinese, and Windows defaults to GBK).
- The remaining 3 are **upstream content drift**: the expected `texture_est_bytes` /
  `unknown_textures` values do not match what the tool prints. Reproduced on the pristine upstream
  commit `62c6db8`, so it is not caused by the changes here. Fixing it means changing the budget
  estimation in `dm/gfx/tools/tbdr_budget.py`, which is out of scope for this repository.

---

## L1 — host with your own game files

### L1-a Environment check

```bash
./setup.sh check
```

Reports "status / where to get what is missing / next command" item by item and changes nothing.
(This path targets an Apple silicon Mac and the iOS build; see `docs/DEPENDENCIES.zh-CN.md`.)

### L1-b Prepare the game files

```bash
./setup.sh <your game package>.xci <prod.keys>
```

Files land in `$DM_ROOT/data/switch/3.3.94012/Update/{ExeFS,RomFS}` (`DM_ROOT` defaults to
`~/dm-work`).

### L1-c Self-consistency checks that still need the game files

```bash
python dm/host/tests/run_manifest.py <directory of patched artefacts> <original main>
```

These scripts take already-prepared artefact paths, which is why they belong to L1.

---

## L2 — on device

### L2-a Build the native library

Use the Android NDK with CMake and the `d2r_mobile_android` shared-library target (see
`dm/android/CMakeLists.txt`, around line 212). The same CMake project also defines dozens of
`EXCLUDE_FROM_ALL` on-device probes (`dm_android_*_probe`); build the ones you need.

### L2-b Package and install

See `docs/ANDROID_D2R_APK.md`. The offline release mode needs `--offline-manifest` +
`--overlay-base` + `--overlay-item-notes` + `--driver-libs` + a signing properties file.
**`--overlay-base` is not optional**:

> Without the touch-UI overlay the host logs `ANDROID_CUSTOM_OVERLAY base_unavailable=1` and the
> game falls back to the stock Switch gamepad UI. Symptom: the character panel is laid out on the
> other side, "create new character" becomes a ✚ key prompt, and **you cannot create a
> character**. That was the root cause of the phone/tablet home-screen mismatch on 2026-10-22.

### L2-c On-device probes

```bash
python dm/android/tests/run_compat_checks.py --output out/device --serial <adb serial> --run-device
```

`--run-device` cross-compiles the L0-c C tests, pushes them to the device, runs them and brings the
results back.

### L2-d Manual acceptance checklist

After launching the game, confirm each item. The expected log line or behaviour is in brackets.

**1. The touch-UI overlay is active**

```
[dm] ANDROID_CUSTOM_OVERLAY item_notes=1 root=/data/user/0/<package>/files/d2r-mobile/romfs-overlay
[dm] 手柄 / 触屏布局：触摸说法 <N> 条（touch_strings.tsv）
[dm] 手柄 / 触屏布局：开机用触屏版（layouts/xtouch_lay 等三个目录）
```

If not, first look in the same log for `ANDROID_CUSTOM_OVERLAY missing=<file> reason=... errno=...`
(the per-file diagnostic added by this port) and for `logcat -s D2ROverlay` lines
`APK carries no overlay-manifest.json` / `files=4 rewritten=4`.

**2. Menu layout matches the other device** → compare two screenshots with the L0-b tool.

**3. Creating a character works** (the step that used to be stuck)

The "create new character" button inside the game panel **does not respond to finger taps** — the
stock Switch front end only accepts gamepad input. The working entry point is the **"开始"
(Start) button in the top-right corner, which is the ✚ (Plus) key**:

| # | Action | Expected |
|---|---|---|
| 1 | top-right "开始" | "选择英雄职业": all eight class models on screen |
| 2 | bottom-right "确认" = Ⓐ | "选择游戏模式" (standard / expert + season option) |
| 3 | "确认" again | "角色名称", with this app's "输入内容" input dialog on top |
| 4 | type a name → "确认" | character created, first act camp loads, orbs and belt appear |

Verify the save landed (a debug build can use `run-as`):

```bash
adb shell run-as <package> ls files/d2r-mobile/save
# expect a new character directory next to the existing ones
```

**4. Frame-rate tiers**: switch between 30 / 40 / 60 in the system menu; the HUD FPS should be
capped accordingly (60 tier measured 53–55 on Adreno + Turnip, about 38 on the Mali system driver).

**5. Returning from the background does not reload**: switch away and back; the process PID must be
unchanged and the picture must not reload. `onPause` releases the bitmap pool mirror; if
`lowmemorykiller` took the process, memory went up again.

### L2-e Two traps when driving the UI with `adb shell input`

1. **A synthetic tap must be held.** `input tap` presses and releases within a few milliseconds,
   and at 28 fps the game never samples it — the symptom is "I tapped and nothing happened". Use
   `input swipe x y x y 400` (hold 400 ms) to behave like a real finger.
2. **Compute overlay button coordinates, do not guess them from a screenshot.** Overlay buttons are
   sized and offset by `overlayScale = clamp(picture width in dp / 1280, 0.62, 1.0)`. On the phone
   (560 dpi, picture 2161×1216) the "确认" button centre is
   `picture right − 39 − 74`, `picture bottom − 39 − 63` = **(2347, 1114)**. Tapping where it looks
   like it is lands on the game UI next to it.

---

## Measured baseline (2026-10-22)

Use this to tell whether something regressed.

| Item | Tablet Lenovo TB710FU<br>Snapdragon 8 Gen 3 / Adreno 750 / Turnip | Phone vivo V2419A<br>Dimensity 9400 / Mali-G925 / system driver |
|---|---|---|
| Menu layout | baseline | normalised diff 4.93, shift 0 px |
| Create character | works | works (full flow, character enters the game) |
| 60 tier FPS | 53–55 | around 38 |
| 40 tier FPS | 40.0 | 40.0 |
| 30 tier FPS | 30.0 | 30.0 |
| Character-select FPS | — | 19–20 |
| Bitmap optimisation | `usemark_ms` 274→33.8 (−88%), `pool_ms` 542→302 | — |
| Power (60/40/30 tiers) | CPU 274/236/217 %, temp 84.5/70.9/62.6 °C | — |

## Known issues and limits of acceptance

- **Mali reports GPU errors on the character-creation screen**: `logcat` shows several
  `mali: Received a GROUP_ERROR_FATAL error on group(0)` while the seven class-model textures are
  uploaded in one burst. This port stops it from **terminating the process** (a failed query read
  is treated as zero, see `query_read_tolerant()` in `dm/gfx/nvn/nvn_vk.c`) and the picture and flow
  continue, but the root cause is not fixed: the GPU side may still drop frames or occasionally
  show a bad frame.
- **The Adreno system driver is unusable**: on Qualcomm only Turnip works; choosing the system
  driver crashes near `vkCreateShaderModule` (about 71 frames in). `GraphicsDriverPolicy` picks the
  driver by SoC automatically.
- **BC textures are always decoded on the CPU**: `DM_BC_NATIVE` is off by default (turning it on
  corrupts the picture).
- **The L0-b tool only checks layout parity**, not colour correctness or partial rendering. Save
  data differences, a different number of characters and a different FPS readout all count as
  difference, which is why the thresholds are generous (10.0 / 4 %).
- Only the two devices above were tested. Other SoC / driver combinations are unverified.
