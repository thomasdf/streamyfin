# Playback Fix Reference

**This is reference material carried over from prior investigation in a different repository. Treat all file:line citations as starting points that must be re-verified against the current Streamyfin codebase. Things may have moved, been renamed, or already been partially fixed.**

## Symptom

Downloaded files in Streamyfin have several playback problems that appear to stem from a common root: the container and timestamp state of the saved file does not match what the player expects. Observed symptoms on Android (Tab S9+):

- Black screens on first playback, sometimes clearing after skipping around
- Seek to the start (second 0) plays fine
- Seek to an arbitrary timestamp (e.g. minute 50) often fails to start, or takes ~60 seconds of visible stall before playback resumes — consistent with a linear scan of the full file because the container has no usable seek index
- Seeking a second time often fails entirely
- Some users report crashes after 10-15 minutes of playback (upstream issue #974 — likely the same cluster, may or may not be fixed incidentally by the other fixes)

## Root Cause Hypotheses (to verify)

Five suspected bugs that stack on each other. File:line citations were valid in an earlier snapshot of the repo — **verify before relying on them.**

### Hypothesis 1: `master.m3u8` → `stream` URL rewrite

- **Suspected file:** `utils/jellyfin/media/getStreamUrl.ts` around line 83
- Streamyfin appears to rewrite the HLS playlist URL into Jellyfin's `/stream` endpoint. `/stream` is designed for live playback, not file download.
- Consequence: the response body is a concatenation of HLS segments without a top-level container index. No MP4 `moov` atom, no MPEG-TS PAT/PMT at coherent intervals, timestamps may reset at segment boundaries.

### Hypothesis 2: MPEG-TS bytes saved as `.mp4`

- **Suspected file:** `providers/Downloads/hooks/useDownloadOperations.ts` around line 113
- The download request asks Jellyfin for `container: "ts"` (MPEG transport stream), but the file is saved as `${filename}.mp4`.
- Consequence: MPV and the Android hardware decoder select the MP4 demuxer based on file extension, then fail to parse MPEG-TS bytes. Visible as black frames and broken seeks.

### Hypothesis 3: `CopyTimestamps: false` in the download profile

- **Suspected file:** `utils/profiles/download.js` around line 68
- The Jellyfin download profile sets `CopyTimestamps: false`, which tells ffmpeg to rewrite PTS/DTS from zero rather than carrying the source timestamps through.
- Consequence: PTS/DTS discontinuities at segment boundaries, and the player's seek target (in source-timeline seconds) no longer maps to a valid packet timestamp.

### Hypothesis 4: `hr-seek: no` for local files

- **Suspected file:** `modules/mpv-player/android/src/main/java/expo/modules/mpvplayer/MPVLayerRenderer.kt` around line 130
- MPV is configured with `hr-seek: no`, forcing keyframe-only seeks. When combined with the corrupt container state above, the player often has no valid keyframe near the seek target.
- Consequence: seeks land nowhere, producing black frames or infinite stall.

### Hypothesis 5: Black loading overlay stays visible

- **Suspected file:** `app/(auth)/player/direct-player.tsx` around lines 977-992
- If MPV never fires `isPlaying: true` (e.g. because the file is malformed), the loading overlay stays on top of the video, masking any underlying playback progress.
- Consequence: even if playback would have worked, the UI hides it.

## Candidate Fixes (treat as proposals, not prescriptions)

### Fix A: rename saved file to `.ts`

```typescript
// useDownloadOperations.ts
// const videoFile = new File(Paths.document, `${filename}.mp4`);
const videoFile = new File(Paths.document, `${filename}.ts`);
```

### Fix B: `CopyTimestamps: true` in download profile

```javascript
// download.js
// CopyTimestamps: false,
CopyTimestamps: true,
```

### Fix C: `hr-seek: yes` for local files

```kotlin
// MPVLayerRenderer.kt
// MPVLib.setOptionString("hr-seek", "no")
MPVLib.setOptionString("hr-seek", if (url.startsWith("file://")) "yes" else "no")
```

Fixes A+B+C are the minimal set that should be tried first. They are one-line changes, surgically targeted, and each has a clear theoretical justification. Whether they fully resolve the symptoms needs to be tested on real hardware.

### Open questions

- Does `master.m3u8` → `/stream` rewrite still happen in current code? If so, is there a cleaner fix (e.g. request a progressive MP4 download from Jellyfin instead, using `Container=mp4&videoCodec=h264`)?
- Is the black-overlay issue (Hypothesis 5) worth including in the same PR, or deferred until the underlying playback is fixed?
- Does the Android hardware decoder path (`mediacodec-copy`) need separate treatment from the software decoder path?

## Related upstream issues (not exhaustive — verify)

- **#974** (open) — "App crashes after 10-15 minutes of downloaded video playback" — may share root cause
- **#1527** (open) — "Resume restarts video / long loading times" — resume exercises seek logic, likely same cluster
- **#1516** (open) — large repo-rewrite PR; unlikely to merge, not a reference for our approach
- **#1523** (open) — offline resume progress; adjacent but different problem
- **#342** (open) — HDR black screen; separate (MPV tone mapping config), not this cluster
