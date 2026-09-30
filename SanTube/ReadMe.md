 # SanTube
 # SanTube – Just Download

 A Brave-inspired, user-first Android application designed for students to download, manage, and share educational content offline.
 > **Learn Offline, Share Knowledge**  
 > A premium, high-performance Android application built with Kotlin and Jetpack Compose for downloading, managing, and viewing tutorial videos and audio offline.

# https://drive.google.com/file/d/1x5VWtlueRhYVsiGZgypKSJeOrcbJZPtL/view?usp=drive_link
# Contributing to SanTube

Thanks for taking a look at SanTube. This document is the map: what the app is made of, where a
given kind of change actually lives, what's expected of a pull request, and the house rules the
existing code already follows so a contribution fits in without a rewrite.

---

## Table of Contents

- [Before you start](#before-you-start)
- [Mental model of the app](#mental-model-of-the-app)
- [Where things live](#where-things-live)
- [Code style](#code-style)
- [Common contributions, mapped out](#common-contributions-mapped-out)
- [Testing](#testing)
- [Device testing](#device-testing)
- [Commits & pull requests](#commits--pull-requests)
- [Reporting a bug](#reporting-a-bug)
- [Design principles (read before adding a feature)](#design-principles-read-before-adding-a-feature)

---

## Before you start

- Android Studio Ladybug (2024.1+) or newer, JDK 17, `compileSdk 36` installed.
- Build once before touching anything, so a build failure you hit later is clearly yours:
  ```bash
  ./gradlew :app:assembleDebug :app:testDebugUnitTest
  ```
- No account, API key, or backend is needed to build or run SanTube — it talks directly to
  whatever site a link points at, plus GitHub (to self-update its download engine).

---

## Mental model of the app

Everything in SanTube answers one of two questions: **"get me this video/audio"**, or
**"show me what I already have."** Nearly every file belongs to one side or the other.

```
                              SanTube
                                 │
              ┌──────────────────┴──────────────────┐
              │                                      │
        GETTING MEDIA                          BROWSING WHAT'S
        (a link in, a file                     ALREADY DOWNLOADED
         on disk out)                          (Reels, Downloads,
              │                                 Share, History tabs)
              │                                      │
   ┌──────────┼──────────┐                 ┌─────────┼─────────┐
   │          │          │                 │         │         │
 detect     analyze    download          Room DB   Reels     file
 platform   (yt-dlp)   (WorkManager +      (single  feed     actions
   │          │         yt-dlp/aria2c/     source    (one     (open,
   │          │         ffmpeg)            of truth  shared   share,
   │          │          │                 for the   player)  delete)
   │          │          │                 library)
   ▼          ▼          ▼
Platform.kt FormatPicker DownloadWorker.kt
            .kt          + DownloadActions/
                          Notifications/Store
```

A second, smaller axis: **foreground vs. background.**

```
              UI (Compose, main thread)
                        │
          ┌─────────────┼──────────────┐
          │                             │
  DownloadViewModel            survives the UI closing
  (what the screen               │
   currently shows)      ┌───────┴────────┐
          │               │                │
          │        DownloadWorker   ReelsPlaybackService
          │        (WorkManager,     (MediaSessionService,
          │         one per          one shared ExoPlayer,
          │         download)        keeps Reels playing)
          │               │                │
          └───────┬───────┴────────────────┘
                   │
           DownloadStore / Room DB
        (shared state everyone reads,
         so the UI, the notification
         buttons and the background
         worker never disagree)
```

If you're not sure where a change belongs, find which of these two diagrams it's answering.

---

## Where things live

```
app/src/main/java/com/example/myapplication/
│
├── MainActivity.kt             all 6 tabs (Home/Reels/Downloads/Share/History/Profile),
│                                dialogs, navigation, theming — the Compose UI shell
├── DownloadViewModel.kt        the screen's brain: link analysis state, active task list,
│                                lesson list, dark mode / settings plumbing
├── DownloadWorker.kt           the actual download: builds the yt-dlp request, picks the
│                                downloader (native for YouTube, aria2c otherwise), merges
│                                streams with ffmpeg, writes the final file
├── DownloadActions.kt          pause / resume / cancel, shared by the UI, the notification
│                                buttons and the worker so they can't disagree
├── DownloadNotifications.kt    every notification a download shows through its lifecycle
├── DownloadStore.kt            the in-progress task list (SharedPreferences + StateFlow)
├── FormatPicker.kt             turns yt-dlp's raw format list into the rows the quality
│                                dialog shows
├── ProgressTracker.kt          turns yt-dlp/aria2c's raw output into a percentage
│
├── data/
│   ├── Platform.kt             URL → YouTube / Instagram / Facebook / Reddit / direct-media /
│   │                            generic-web
│   ├── YtDlpEngine.kt          starts yt-dlp/ffmpeg/aria2c once per process, pre-compiles
│   │                            yt-dlp in the background, self-updates it at most daily
│   ├── StorageRepository.kt    where files live on disk, MediaStore publishing, size/
│   │                            duration/MIME helpers
│   ├── LessonRepository.kt     the Room-backed library + reconciling it against what's
│   │                            actually on disk
│   ├── LessonDao / LessonEntity / LessonExtensions / SanTubeDatabase   Room plumbing
│   ├── PreferencesRepository.kt  dark mode, Quick Link visibility, Background Play, the
│   │                            yt-dlp update timer — small persisted settings
│   └── ReelArtwork.kt          the generic artwork Reels falls back to for audio with no
│                                captured thumbnail
│
└── ui/
    ├── ReelsScreen.kt          the vertical Reels feed itself
    ├── ReelsPlaybackService.kt the MediaSessionService keeping Reels playback alive in
    │                            the background
    └── theme/                 Compose theme (colors, type)
```

Tests mirror this under `app/src/test/java/...` (unit tests — `FormatPickerTest.kt`,
`ProgressTrackerTest.kt`, pure logic) and `app/src/androidTest/java/...` (instrumented,
`FormatSelectionDialogScrollTest.kt` — anything that needs a real Compose layout pass or a
real device to prove, like "does this fit on a cramped screen").

---

## Code style

The existing code follows a few rules consistently. Match them rather than introducing a second
style:

- **Comments explain *why*, never *what*.** A well-named function doesn't need a comment saying
  what it does. A comment exists when there's a non-obvious reason behind a choice — a past bug,
  a platform quirk, a constraint that isn't visible from the code itself.
  ```kotlin
  // REPEAT_MODE_ONE: a reel loops itself, matching the original per-item player behaviour.
  val player = ExoPlayer.Builder(this, renderersFactory).build().apply { repeatMode = Player.REPEAT_MODE_ONE }
  ```
  not
  ```kotlin
  // Set repeat mode to one
  player.repeatMode = Player.REPEAT_MODE_ONE
  ```
- **No speculative abstraction.** A one-off case stays inline. Don't add a strategy interface,
  a config flag, or a "for future use" parameter for something only one caller needs today.
- **No dead code, no commented-out code, no `// TODO` left unowned.** If something is unused,
  delete it.
- **Single source of truth for shared state.** `DownloadStore` and the Room database exist so
  the UI, the notification, and the background worker read the same thing instead of maintaining
  their own copies that can drift apart. If you're adding state that more than one of those needs
  to agree on, put it in one of those, not in a new parallel variable.
- **Validate at boundaries, trust internal code.** yt-dlp's raw output and user-supplied URLs get
  checked; a value already produced by another function in this codebase doesn't need re-checking
  "just in case."
- **Kotlin/Compose conventions already in use**: `StateFlow` for anything the UI observes,
  `remember`/`rememberSaveable` scoped tightly to what actually needs to survive recomposition,
  `DisposableEffect` paired with an `onDispose` that actually undoes what the effect did,
  composables named for what they render (`FocusedReelItem`, `ReelsFilterBar`), files organized
  by responsibility (`data/` for repositories and platform logic, `ui/` for screens that aren't
  in `MainActivity.kt`).

---

## Common contributions, mapped out

```
"I want to add support for another site
 (TikTok, Vimeo, SoundCloud, ...)"
        │
        ├─ 1. data/Platform.kt — add an enum entry + hostname match in detectPlatform()
        ├─ 2. Confirm yt-dlp itself supports the site (it almost certainly does — SanTube's
        │      platform list is about UI labeling and site-specific quirks, not extraction)
        ├─ 3. If the site needs special handling (e.g. Reddit's separate video/audio streams),
        │      that goes in DownloadWorker.kt's format-selection branch
        └─ 4. Add a fixture-based test in FormatPickerTest.kt if the format list shape differs

"I want to add a Reels feature"
        │
        ├─ Per-item UI/behavior (a button, an overlay, a gesture)  → ui/ReelsScreen.kt,
        │    inside FocusedReelItem
        ├─ Feed-wide behavior (filtering, ordering)                → ui/ReelsScreen.kt,
        │    the top of ReelsScreen() where `reelLessons` is derived
        ├─ Playback/background-service behavior                    → ui/ReelsPlaybackService.kt
        └─ A new persisted on/off setting                          → data/PreferencesRepository.kt
             + wire it through DownloadViewModel → MainActivity → the screen

"I want to change how/where downloaded files are organized on disk"
        └─ data/StorageRepository.kt — this is the one place that decides folder layout,
             naming, and MediaStore publishing. Don't build paths by hand elsewhere.

"I want to add a new download notification state, or change what a
 notification shows"
        └─ DownloadNotifications.kt — every notification a task can show lives here;
             DownloadActions.kt is what the notification's buttons actually call into.

"I want to change what counts as 'the library' (what shows up in
 Reels/Downloads/Share/History)"
        └─ data/LessonRepository.kt — all four tabs read the same Lesson list from here.
             Changing it changes it everywhere at once; that's intentional.

"I want to add a new screen/tab"
        └─ MainActivity.kt — add to the `when (selectedTab)` block and the bottom
             NavigationBar's item list. Keep the new screen's own Composable in ui/ if it's
             more than a small addition, the way ReelsScreen.kt already is.
```

If a change doesn't fit cleanly into one of these, it's a sign to look at
[Where things live](#where-things-live) again before picking a file — misplaced logic is the
easiest way to make a small change hard to review.

---

## Testing

```
Pure logic, no Android framework needed
        → app/src/test/  (JVM unit test, fast, runs with :app:testDebugUnitTest)
        → e.g. "given this raw yt-dlp format list, does FormatPicker build the right rows"

Needs a real Compose layout pass, a real screen size, or device behavior
        → app/src/androidTest/  (instrumented, runs on a device/emulator)
        → e.g. "does the download-bundle button stay reachable when the dialog is squeezed
          onto a shorter screen" (FormatSelectionDialogScrollTest.kt is a good template:
          it reproduces a real reported bug with a realistic data shape, not a synthetic one)
```

Before opening a PR:
```bash
./gradlew :app:testDebugUnitTest      # must pass
./gradlew :app:assembleRelease        # must compile clean under R8 too — a class or method
                                       # a test never exercises can still break release-only
```
If you touched anything ProGuard/R8-sensitive (reflection, Jackson models, JNI), also update
`app/src/main/keepRules/rules.keep` and verify with a release build, not just debug.

A logic change with no test isn't reviewable the same way a logic change with a test is — add one
in the matching `test`/`androidTest` location above when you can.

---

## Device testing

Some of this app can only really be verified on a device: whether a download actually survives
the app being closed, whether a notification's buttons work, whether the Reels feed still plays
audio with the screen off, whether a format that analyzes correctly also actually downloads
(sites intermittently reject specific formats independent of what their own metadata claims).

- `adb install app/build/outputs/apk/debug/app-debug.apk`, or `./gradlew :app:installDebug`
  with a device connected.
- `adb logcat -s SanTubeTiming SanTubeDownloader SanTubeEngine` — SanTube's own log tags.
- A real download exercises real third-party infrastructure (YouTube, etc.) — avoid hammering
  the same video repeatedly in a short window; sites that rate-limit will start failing in ways
  that look like a SanTube bug but aren't.

---

## Commits & pull requests

- Keep a PR to one thing. A bug fix doesn't need to also refactor the file around it.
- Write the commit message around **why**, not a restatement of the diff — "fix pause not
  surviving app restart (partial file was written to cacheDir, which Android can clear)" tells a
  reviewer (and future-you) something the diff alone doesn't.
- If the change is user-visible, mention what changed for the user, not just which function
  changed.
- Run the [Testing](#testing) commands above before pushing — a red build is the fastest way to
  stall a review.
- Don't bundle formatting-only changes to unrelated code into a functional PR; it hides the
  actual diff.

---

## Reporting a bug

Include:
- The link/site involved (if download-related) — many bugs are site-specific, not general.
- Whether it happens on analysis, on the actual download, or in playback afterward — these are
  different subsystems (see the [mental model](#mental-model-of-the-app) above) and "it doesn't
  work" covers very different causes depending on which stage fails.
- The exact error text, if any was shown — SanTube tries to surface the real underlying reason
  (e.g. an HTTP status from the source site) rather than a generic failure message, and that text
  usually points straight at the cause.
- Android version and, if it's a playback/UI bug, the device — decoder and screen-size
  differences between phones are a real source of bugs here, not a theoretical one.

---

## Design principles (read before adding a feature)

These came out of real decisions already made in this codebase — worth keeping in mind so a new
feature doesn't reopen a settled trade-off:

- **One shared player, not one per item.** Reels was originally one `ExoPlayer` per visible feed
  item; it was rewritten to a single shared player because a phone's hardware video decoder is a
  limited, often single-instance resource — preparing two videos on it at once could fail
  outright. Don't reintroduce multiple concurrent players without re-confirming this isn't still
  true on real hardware.
- **Analyze once, reuse the result.** Picking a quality doesn't re-extract the page — it reuses
  the analysis from moments earlier (`--load-info-json`) to avoid asking the source site twice
  for the same thing in quick succession.
- **The download engine updates itself, carefully.** yt-dlp self-updates at most once a day, and
  the background pre-compile step that speeds up normal runs is written to never silently
  overwrite a just-installed newer version with a stale one it already had in hand. If you touch
  `YtDlpEngine.kt`, preserve that ordering guarantee — a silently-stale download engine is a real
  category of bug here (outdated extractors get rejected by source sites in ways that look like
  "the site is blocking me" but aren't).
- **A failure should say why, not just that it failed.** Downloads and playback both prefer
  surfacing the real underlying reason (an HTTP status, "this format isn't supported on this
  phone's decoder") over a generic "something went wrong" — that's what makes a bug report
  actionable instead of a guess.
