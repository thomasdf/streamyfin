# Streamyfin Fork — Download + Playback Reliability Work

This branch (`planning/streamyfin-fixes`) exists to set up a structured, review-sized set of upstream PRs against `streamyfin/streamyfin`. No source code has been modified yet. The planning work itself needs to happen before any implementation.

## Orientation for the next agent

You are picking this up from a diagnostic session where a pile of empirical evidence was collected about download and playback reliability, but **no code was written and no plans were finalized**. Your job is to turn the evidence into plans, branch by branch, PR by PR.

**Do not trust the hypotheses below uncritically.** The symptoms are real and were reproduced live on a Tab S9+, but the file:line citations and specific root-cause claims came from an earlier snapshot of the repo and informal code reading. Before writing any plan, verify:

- The file paths still exist and the cited lines still do what the hypothesis claims
- The upstream issue numbers still refer to the bug the hypothesis claims they do (read the comments)
- There isn't a cleaner fix that bypasses the stated fix entirely (e.g. request a different container from Jellyfin rather than patching the demuxer path)
- There isn't a newer PR already tackling the same area

Dig as deep as you need to. The maintainer of this project accepts well-evidenced PRs but silently closes vague ones as stale. The quality of the evidence *is* the PR.

## Workflow constraint

**One superpowers session per PR.** Do NOT try to plan all three PRs inside a single session. The user will:

1. Read this handoff
2. Start a fresh Claude Code session for **PR 1** (playback container/seek fixes)
3. In that session, invoke `superpowers:brainstorming` with this handoff and the context reference as input, validate scope, then `superpowers:writing-plans` to produce the plan
4. Commit that plan to this fork under `docs/superpowers/plans/`, push, and close the session
5. Repeat for **PR 2** (Android wake/wifi locks) in another fresh session
6. Repeat for **PR 3** (progress/ETA via Jellyfin session polling) in another fresh session
7. Only after all three plans exist does implementation start (each PR still in its own session)

Each session must do its own investigation against the current code. Do not copy conclusions from prior sessions — verify.

## The two themes and the proposed PR structure

Everything below is a starting point. If during brainstorming the scope should change, change it.

### Theme A — Download reliability on Android

**Symptom:** downloads orphan when the device screen times out. The foreground-service download manager inside `modules/background-downloader/` does start properly, but the OkHttp call inside it is not held awake. During Android doze the WiFi radio enters low-power mode, the TCP stream goes idle, and OkHttp's 60-second read timeout fires. The download aborts client-side. Jellyfin, having no way to know the client is gone, keeps transcoding at full speed (observed at 156-161 fps NVENC on an unpatched Tab S9+ v0.51.0), eventually exits ffmpeg cleanly with code 0, and no file ever lands on the device. Streamyfin's download list shows "No downloaded items."

**Upstream references to verify:** #615 (2025-03-19, 13 months open, iOS-reported, 7 comments), #1246 (2025-11-29, Android, 2 comments confirming on v0.51.0), #1298 (2025-12-29, Android, vague).

**Suspected root cause to verify:** `modules/background-downloader/android/src/main/java/expo/modules/backgrounddownloader/OkHttpDownloadManager.kt` constructs an OkHttp client with `readTimeout(60s)`, `callTimeout(0)`, and acquires no `PowerManager.WakeLock` or `WifiManager.WifiLock`. The enclosing `DownloadService.kt` is a `FOREGROUND_SERVICE_DATA_SYNC` foreground service with a persistent notification, but Android 14+ foreground-service visibility alone does not prevent CPU doze and does not prevent WiFi from dropping to low-power mode during a screen-off window. A `PARTIAL_WAKE_LOCK` plus a `WIFI_MODE_FULL_HIGH_PERF` WiFi lock, reference-counted across concurrent downloads and released on complete/error/cancel/service-destroy, is the canonical fix.

**Also in this theme:** inaccurate in-app progress percentage and ETA. Verified live: the transcode stream returned by Jellyfin has no `Content-Length` (can't — the output isn't finalized), so OkHttp's progress callback fires with `totalBytes = -1`. Whatever progress bar Streamyfin shows is either extrapolation from raw byte rate or derived from a bad assumption about file size. Upstream #213 ("Download reached >100% progress before completing") and #241 ("estimated / actual file size download") were both closed as stale without a maintainer response — same underlying cause, different symptoms. The accurate signal is already exposed by Jellyfin: `/Sessions` responses include `TranscodingInfo.CompletionPercentage` reported by ffmpeg itself, accurate to source duration. We used it to give an ETA within a minute of the actual completion time.

### Theme B — Playback of completed downloads

**Symptom cluster:** downloaded files play from second 0 but fail or stall on seeks. Seeking to minute 50 of a 1080p download took about 60 seconds to resume playback on a Tab S9+ — consistent with MPV doing a linear scan for a valid keyframe and coherent timestamp because the container has no usable index. Some seeks fail entirely. Users also report black screens on first playback and mid-playback crashes.

**Upstream references to verify:** #974 (open, "crashes after 10-15 min of downloaded playback"), #1527 (open, "resume restarts video / long loading times"). The specific "seek mid-file fails" wording is not filed; the slow-seek symptom is not filed.

**Suspected root causes:** see `docs/superpowers/context/playback-fix-reference.md`. Five stacked hypotheses. The three with the clearest theoretical justification and the smallest diffs are:

- Save downloaded files with a `.ts` extension instead of `.mp4` (the bytes are MPEG-TS; current naming causes the player to pick the wrong demuxer)
- Change the download profile's `CopyTimestamps` from `false` to `true` (preserves PTS/DTS continuity)
- Set MPV's `hr-seek` to `yes` for local file URLs (enables sub-keyframe seeking)

These are the candidate PR 1 fixes. The other two hypotheses (URL rewrite from `master.m3u8` to `/stream`, and the black-overlay UI issue) are higher-blast-radius changes; decide during brainstorming whether to include.

## Proposed PR split (starting point — open to revision)

- **PR 1 — playback container + seek fixes.** Lowest risk, most obvious, smallest surface area. Three one-line-ish changes to `.ts` extension + `CopyTimestamps: true` + `hr-seek: yes`. Strong theoretical justification, easy to demonstrate with before/after seek-latency numbers. Plausibly resolves the seek-mid-file-fails symptom, seek-takes-60-seconds symptom, and possibly the mid-playback crash (#974) incidentally.
- **PR 2 — Android download wake + wifi locks.** Roughly 30-50 lines of Kotlin in the background-downloader module. Adds `PARTIAL_WAKE_LOCK` and `WIFI_MODE_FULL_HIGH_PERF` WiFi lock, reference-counted, with proper release on all termination paths. Closes #615 / #1246 / #1298 with a clean reproduction protocol + logcat evidence.
- **PR 3 — progress and ETA accuracy from Jellyfin session telemetry.** Roughly 75 lines of TypeScript. During an active download, periodically read Jellyfin `/Sessions`, find the active Streamyfin session, use `TranscodingInfo.CompletionPercentage` as the progress/ETA source. Closes #213 / #241.

Rationale for the split: three independent concerns, each reviewable in isolation, each mergeable without blocking the others. Solo maintainer can review one at a time. If PR 1 merges but PR 2 stalls, users still see the playback improvements. Big-bang PRs die in this project — open issues for 13 months attest to this. Scoped PRs have a real chance.

## User preferences and constraints

- Developer user, not end-user. Expects to iterate on code; does not want hand-holding, but values precise reproduction and evidence
- Laptop is Manjaro, has Android SDK + JDK from Flutter work, will add `bun` if needed. Builds will happen there, not here
- Primary test devices: Samsung Tab S9+ (Android), and an Android phone
- Every PR must ship with concrete reproduction steps and before/after evidence (logcat, screen recording, or server-side session log)
- Smallest-possible diff per PR. No refactors, no driveby style cleanup
- Each plan must include: root-cause analysis grounded in file:line citations of the *current* code, the proposed change, the verification protocol, and relevant upstream issue links
- Each plan must include a reproduction script or snippet so a reviewer can verify on their own machine

## Verification infrastructure that already works

A server-side session poller is straightforward: query Jellyfin `/Sessions` with `X-Emby-Token: $JELLYFIN_API_KEY`, filter `Client == "Streamyfin"`, log `TranscodingInfo.CompletionPercentage`, `Framerate`, and `LastActivityDate` every 10 seconds. An "orphan transcode" is visible when `CompletionPercentage` climbs normally and ffmpeg exits but the device reports zero downloaded. The poller is worth including in the repo (e.g. under `scripts/dev/`) so the PR 2 plan can reference it.

On the Android side, `adb logcat | grep -E "DownloadService|OkHttpDownloadManager|WakeLock"` is the primary signal for wake-lock testing, once debug logging is added.

## Deliverables for the plans you write

Each PR plan should land in `docs/superpowers/plans/YYYY-MM-DD-<pr-topic>.md` and contain:

- One-sentence goal
- Upstream issues closed and their evidence links
- Root-cause analysis citing current code (verified, not copied)
- Change list (file-by-file, with exact patch intent)
- Verification protocol (reproduce the broken behavior, apply the patch, demonstrate the fix)
- Risks / open questions
- Recommended PR title, description template, and commit message style matching the upstream project

When the plans for all three PRs are written and committed to this branch, push and stop. Implementation is a follow-on session per PR.

## References in this branch

- `docs/superpowers/context/playback-fix-reference.md` — the prior-investigation notes for the playback cluster (Theme B). Starting point for PR 1, not a prescription.

## References to the upstream project

- Upstream repo: https://github.com/streamyfin/streamyfin
- Fork origin: https://github.com/thomasdf/streamyfin
- Maintainer: fredrikburmester (solo; PRs merged occasionally by Simon-Eklundh and lancechant)
- Last v0.x release: v0.51.0 (2026-01-06). v0.52.0 has not been cut at time of writing
- PR merge cadence: approximately one non-bot PR per month
