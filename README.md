# Fix: custom video wallpaper goes **black after unlock** on macOS (Sonoma / Sequoia / Tahoe)

Turn any video into a **custom aerial / live wallpaper** that keeps playing after you
unlock your Mac — instead of turning into a **black screen**.

The real root cause and a working, self-contained encoder (Swift + VideoToolbox) are in
this repo. As far as I can tell, **this is the first place the actual cause is documented.**

---

## Does this match your problem?

If you've hit any of these, you're in the right place:

- You added your own `.mov` as an animated/aerial wallpaper and the desktop **turns
  black after you unlock** (or after wake from sleep / screensaver).
- The wallpaper **plays once, then freezes or goes black** and never animates again.
- Custom **dynamic / live / aerial wallpaper stops animating** on macOS 14 Sonoma,
  15 Sequoia, or 26 Tahoe.
- You used a helper app such as **VideoPaper**, or dropped a file into
  `~/Library/Application Support/com.apple.wallpaper/aerials/videos/`, and it doesn't
  stick.
- Console shows:
  ```
  [video-player] Error handling sample: Error Domain=WallpaperExtensionKit.VideoSampleReadingErrors Code=4.
  [video-player-layer] VideoPlayerLayer failed to acquire next sample due to error: ...Code=4
  ```

There is **no crash report** — the video decodes fine, it just never composites, so the
desktop is black.

## TL;DR — the fix

Apple's built-in aerials are **HEVC videos with temporal scalability** (multiple HEVC
**temporal sub-layers**, described by `tscl` / `tsas` sample groups). The wallpaper
engine's unlock animation ("ramp down" — the smooth slow‑motion‑to‑freeze effect) reads
each frame's temporal-layer info. If it's missing, it throws
`VideoSampleReadingErrors.noTemporalInfo` (**Code=4**) and shows black.

**No public encoder produces this** — not `ffmpeg` (`libx265` or `hevc_videotoolbox`),
not Apple's own `avconvert`. You have to drive **VideoToolbox** directly with
`kVTCompressionPropertyKey_BaseLayerFrameRate` on the hardware encoder. That's exactly
what [`encode_temporal.swift`](encode_temporal.swift) does.

```bash
# build (or just run: swiftc -O encode_temporal.swift -o encode_temporal)
./build.sh

# encode: repeat an 18s clip 17x -> ~5 min, ~12 Mbps (matches Apple's aerials)
./encode_temporal input.mov output.mov 17 12

# verify the temporal sample groups are present
python3 groups.py output.mov      # expect: sgpd 'tscl', sgpd 'tsas', csgm 'tscl', csgm 'tsas'
```

Then set `output.mov` as your wallpaper (via VideoPaper, or by replacing the `.mov` the
aerial system already references). Lock/unlock — it keeps playing. ✅

## Requirements

- macOS 26 (Tahoe) — developed/tested on 26.4.1 (build 25E253), Apple Silicon.
  The same mechanism exists on Sonoma/Sequoia; adjust as needed.
- Xcode command-line tools (`swiftc`) — for building the encoder.
- `python3` — only for the optional `groups.py` verifier.
- `ffmpeg` — optional, only if you want to loop/pre-process elsewhere.

## Usage

```
./encode_temporal <input.mov> <output.mov> [loopCount] [bitrateMbps]
```

- `input.mov` — your source clip (any codec AVFoundation can read).
- `loopCount` — repeat the source content back-to-back with continuous timestamps
  (default `1`). Use this to reach a long duration from a short clip.
- `bitrateMbps` — average bitrate (default `10`). Apple's aerials are ~10–12 Mbps 4K.

The encoder:
1. decodes the source with `AVAssetReader` (to 10-bit pixel buffers),
2. re-encodes with `VTCompressionSession` (HEVC **Main 10 / 10-bit**, hardware,
   **2 temporal sub-layers** via `BaseLayerFrameRate = fps/2`), writing a BT.709 `colr` atom,
3. writes via `AVAssetWriter`, which emits the `tscl` / `tsas` sample groups.

> **Why 10-bit (Main 10)?** Apple's aerials are 10-bit. If you encode 8-bit, playback
> and the unlock ramp still work, but the **preview thumbnail** in System Settings →
> Wallpaper renders as garbled black‑and‑white noise (the poster generator expects
> 10-bit). Encoding Main 10 fixes the preview. The included encoder outputs 10-bit by
> default.

## The looping caveat (make it long)

There's a **second, separate** issue: the wallpaper's looping is not robust for **short**
clips. An ~18 s clip plays, loops once, then freezes (`rate 0.00`, permanent black); the
log shows `Error copying sample buffer. Buffer was nil.` at the loop boundary. Apple's
built-in aerials are long (~5–8 min) so they essentially never loop during a session.

That's why the encoder has a `loopCount`: make a **long** file (≈5 min is plenty and
matches Apple) so looping doesn't happen in normal use:

```bash
./encode_temporal source.mov out.mov 17 12   # 18s x17 ≈ 5 min, ~460 MB
```

## Install as a wallpaper

**Option A — VideoPaper (or similar helper app):** just import `out.mov`.

**Option B — manual:** replace the `.mov` the aerial system already references under
`~/Library/Application Support/com.apple.wallpaper/aerials/videos/<UUID>.mov`, then
reload the agent:

```bash
pkill -x WallpaperAgent
```

## Verify it worked

```bash
python3 groups.py out.mov
# sgpd 'tscl' + sgpd 'tsas' + csgm 'tscl' + csgm 'tsas'  == same as a genuine Apple aerial
```

After installing, a clean unlock logs the ramp completing with **zero** `Code=4`:

```
[video-player] Switching player state <private> -> .beginRampingDown
[video-player] Switching player state .beginRampingDown -> <private>   # ramp completes
AVSampleBufferVideoRenderer setReadyForDisplay ... YES                  # stays visible
```

---

## Deep dive: how the root cause was found

The playback runs in
`/System/Library/ExtensionKit/Extensions/WallpaperAerialsExtension.appex`, linking the
private `WallpaperExtensionKit.framework` (only present in the dyld shared cache). Steps:

1. The fatal log is `WallpaperExtensionKit.VideoSampleReadingErrors Code=4` during the
   unlock transition (`Presentation Mode 'locked' -> 'default'` → `.beginRampingDown`).
2. Extract the framework from the shared cache with `/usr/lib/dsc_extractor.bundle`,
   parse its Swift `__swift5_fieldmd` metadata → the enum case order:

   | Code | Case |
   |---|---|
   | 0 | `noAssetReader` |
   | 1 | `noSampleBuffer` |
   | 2 | `noPixelBuffer` |
   | 3 | `noTimeAdjustedSampleBuffer` |
   | **4** | **`noTemporalInfo`** |

3. The framework imports `kCMSampleAttachmentKey_HEVCTemporalLevelInfo`,
   `...HEVCTemporalSubLayerAccess`, `...HEVCStepwiseTemporalSubLayerAccess`.
4. Comparing MP4 sample groups confirms it — Apple aerials have temporal-scalability
   sample groups; ordinary encodes have none:
   ```
   Apple aerial:   sgpd 'tscl'  sgpd 'tsas'  csgm 'tscl'  csgm 'tsas'
   Normal encode:  (none)
   ```

The unlock animation slows the video by dropping the higher HEVC temporal layer; with no
temporal layers there's nothing to drop and no per-frame temporal info → `noTemporalInfo`
→ black.

## What did NOT fix it (so you don't waste time)

All of these were tried and made **no** difference:

| Attempt | Result |
|---|---|
| Container / filename / UUID / manifest catalog entry | ❌ (Apple's *bytes* work under any name) |
| Color atom `colr` / BT.709 | ❌ |
| Track aperture `tapt` | ❌ |
| 8-bit vs 10-bit (Main vs Main10) | ❌ for the black-screen bug — but 10-bit *is* needed to fix the preview thumbnail, see below |
| B-frames / `ctts` | ❌ |
| Frame rate 24 → 240 fps, timescale 240000 | ❌ |
| `sdtp` / `cslg` sample tables | ❌ |
| `MADMLEnhancementRequest ... "Could not obtain ML enhancement result"` | ❌ red herring — genuine Apple aerials log it too |
| Reduce Motion accessibility setting | ❌ |
| Re-encoding with Apple's own `avconvert` | ❌ |
| **HEVC temporal sub-layers (`tscl`/`tsas`) via VideoToolbox** | ✅ **this is the fix** |

## Files

- [`encode_temporal.swift`](encode_temporal.swift) — the VideoToolbox temporal-scalability encoder.
- [`build.sh`](build.sh) — one-line convenience script that compiles `encode_temporal.swift` with `swiftc`.
- [`groups.py`](groups.py) — dumps a `.mov`'s sample-group boxes to verify `tscl`/`tsas`.

## License

MIT — see [LICENSE](LICENSE).

---

<sub>Keywords: macOS custom video wallpaper black screen, aerial wallpaper black after
unlock, custom live wallpaper stops animating, dynamic wallpaper black desktop, macOS
Sonoma / Sequoia / Tahoe wallpaper not working, VideoPaper black screen,
WallpaperAerialsExtension, VideoSampleReadingErrors Code 4, noTemporalInfo, HEVC temporal
sub-layers / temporal scalability, VideoToolbox BaseLayerFrameRate, tscl tsas sample
groups, com.apple.wallpaper aerials videos.</sub>
