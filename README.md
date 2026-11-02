<p align="center"><img src="assets/branding/veylora_icon_256.png" width="128" alt="Veylora logo"></p>

# Veylora

*Create what you envision.*

A Flutter app that lets you line up a shot by floating a reference image from your
gallery over the device's live camera preview, tuning its opacity, and capturing the
shot — a photo or a clip, framed to a ratio you pick. The reference is only a guide
and is never saved into your media. Built for Android and iOS.

> **Capture, video and saving are live.** The camera preview, the reference overlay
> and its opacity/position controls, the capture-ratio framing, photo capture and
> video recording all work. The shutter saves the camera's own image — cropped to the
> frame you saw when the ratio calls for it, and otherwise byte for byte — straight to
> the photo library; the preview, the controls and the reference overlay are never part
> of it. Clips are muxed by the platform recorder and saved as they were filmed.

## Features

- Live camera preview with a reference image floated over it as a translucent guide.
- Opacity 0–100 % with the exact percentage, presets and a reset; drag, pinch and
  two-finger rotate anywhere on the preview, with quarter-turn snapping.
- Fit / Fill / Center sizing, reset, lock, hide and remove — all without changing
  the captured output.
- 4:3 / 16:9 / 1:1 / Full capture framing; the spill outside the frame is dimmed
  (or hidden) and is never part of the shot.
- Photo capture that saves the camera's own bytes, cropping only when the ratio
  needs it, and platform-muxed video at the camera's native ratio.
- Saves to the system photo library (`Pictures/Veylora` on Android, `Photos` on
  iOS) and keeps an in-app history of photos, clips and reference images.
- Share, delete, replay and "use as reference" from that history.
- Dark / light / system theme, preferred lens, grid, sensor spill, default opacity
  and haptics — all persisted on the device.
- Permissions requested only when needed, with a rationale sheet first and a clear
  **Open settings** path for permanently denied states.
- Nothing leaves the device: no account, no network, no analytics.

---

## Requirements

| Tool | Version used |
| --- | --- |
| Flutter | 3.44.6 (stable) |
| Dart | 3.12.2 |
| Android SDK | platform **android-37.0** (`compileSdk 37`, `compileSdkMinor 0`), NDK 28.2.13676358, CMake 3.22.1, build-tools 36.0.0, `minSdk 24`, `targetSdk 36` |
| Xcode (iOS) | 15+ on macOS, deployment target 13.0 (required by `camera_avfoundation`) |

Dependencies (latest mutually compatible stable releases, resolved by `flutter pub`):

| Package | Version | Purpose |
| --- | --- | --- |
| `flutter_riverpod` | 3.4.3 | state management |
| `go_router` | 18.0.2 | declarative routing |
| `camera` | 0.12.1 | live camera preview, stills and video |
| `image_picker` | 1.2.4 | picking a reference image |
| `video_player` | 2.14.1 | playing recorded clips in the gallery |
| `image` | 4.12.0 | cropping a still to the selected capture ratio |
| `gal` | 2.3.3 | saving captures to the photo gallery |
| `share_plus` | 13.3.1 | system share sheet |
| `permission_handler` | 13.0.2 | runtime permission requests |
| `shared_preferences` | 2.5.6 | local settings storage |
| `path_provider` | 2.1.6 | app directory / file handling |
| `path` | 1.9.1 | cross-platform path joins |
| `cupertino_icons` | 2.0.0 | the Cupertino glyph font Flutter's adaptive widgets fall back to on iOS |

---

## Project structure

Feature-first layout: cross-cutting concerns in `core`, feature slices in `features`,
and re-usable UI in `shared`.

```
lib/
├── main.dart                       # entry point -> bootstrap()
├── app/
│   ├── bootstrap.dart              # error zones, runApp, ProviderScope
│   ├── app.dart                    # MaterialApp.router
│   └── router/
│       ├── app_routes.dart         # route paths + names
│       └── app_router.dart         # GoRouter provider, error handling
├── core/                           # framework-agnostic building blocks
│   ├── constants/                  # app + storage keys
│   ├── errors/                     # AppException hierarchy + AppErrorHandler
│   ├── logging/                    # AppLogger
│   ├── permissions/                # AppPermission, PermissionService
│   ├── storage/                    # KeyValueStore + SharedPreferences impl
│   ├── files/                      # FileService (capture directory/files)
│   └── theme/                      # colors, spacing, typography, CameraTheme, AppTheme
├── shared/
│   ├── widgets/                    # AppButton, AppIconButton, AppSlider, views, sheets
│   └── extensions/                 # BuildContext helpers (colors, text, snackbars)
└── features/
    ├── home/       presentation/pages/
    ├── camera/     presentation/{pages,providers,widgets}/
    ├── overlay/    domain/, presentation/providers/
    ├── gallery/    domain/, data/, presentation/{pages,providers,widgets}/
    │               # picker, gallery saver, share, local history
    └── settings/   domain/, data/, presentation/{providers,pages}/
```

Each feature follows `domain` (models) → `data` (services/repositories) →
`presentation` (providers, pages, widgets). Providers are co-located with the service
or repository they expose.

---

## Installation

```bash
git clone <your-repo-url> Veylora
cd Veylora
flutter pub get
```

Verify your toolchain:

```bash
flutter doctor -v
```

---

## Releases

Tagged builds (`v1.0.0`, `v1.0.1`, …) are built and published to GitHub Releases
by `.github/workflows/release.yml`. Download the Android APK (installable) and AAB
(for Google Play) from the repository's **Releases** page; a signed iOS IPA is
attached only when Apple signing secrets are configured. See
[RELEASE.md](RELEASE.md) for the release workflow, the required GitHub Secrets,
TestFlight distribution and troubleshooting.

---

## Running on Android

1. **Install the required SDK components.** `permission_handler_android` 14 compiles
   against `compileSdk 37`, and Flutter pins NDK `28.2.13676358`, so the app is
   configured with `compileSdk 37`. Google publishes API 37 as the
   **minor-versioned** platform `android-37.0`, which is why
   `android/app/build.gradle.kts` also sets `compileSdkMinor = 0` — without it AGP 9
   looks for a platform named `android-37` and fails with
   `Failed to find target with hash string 'android-37'`. Install the components
   (AGP downloads anything still missing on the first build, as long as the SDK
   directory is writable and the licences are accepted):

   ```bash
   sdkmanager "platforms;android-37.0" "build-tools;36.0.0" \
              "ndk;28.2.13676358" "cmake;3.22.1"
   flutter doctor --android-licenses   # accept any pending licenses
   ```

   The Android SDK directory must be writable by your user (`sdk.dir` in
   `android/local.properties`); otherwise Gradle cannot auto-install missing components
   and the build fails with `The SDK directory is not writable`. If the SDK lives in a
   shared, admin-only folder, run `sdkmanager` from an elevated prompt.

   If Kotlin fails with `Could not close incremental caches` (the project and the pub
   cache are on different drives on Windows), add `kotlin.incremental=false` to your
   user-level `~/.gradle/gradle.properties`.

2. Start a device or emulator, then run:

   ```bash
   flutter devices
   flutter run -d <device-id>          # or simply: flutter run
   ```

3. Build an APK:

   ```bash
   flutter build apk --debug           # debug
   flutter build apk --release         # release (add your signing config first)
   ```

---

## Building and shipping on iOS

iOS builds require macOS with a current Xcode and its command-line tools. Flutter
3.44 uses **Swift Package Manager** by default, so there is no `Podfile` and no
`pod install` step.

> **Verified status:** the iOS app was **not built or run during this audit** —
> that needs macOS and an Apple signing identity. Everything below is the path to
> produce and sign it; the shared Dart code, analyzer and test suite are verified
> on the development machine.

### Build and sign with Xcode

1. Install Xcode, open it once, and accept the licence
   (`sudo xcodebuild -license accept`).
2. `flutter doctor -v` must report the iOS toolchain; `flutter precache --ios`
   downloads the engine artifacts.
3. Confirm the bundle identifier is unique to your account. It is
   `com.veylora.app` today (see `PRODUCT_BUNDLE_IDENTIFIER` in
   `ios/Runner.xcodeproj`), matching the Android `applicationId`/`namespace`;
   change both if that identifier is already taken.
4. Open the **workspace**, not the project, so Swift packages resolve:

   ```bash
   open ios/Runner.xcworkspace
   ```

5. Select the **Runner** target → **Signing & Capabilities**:
   - tick **Automatically manage signing**;
   - choose your **Team**;
   - confirm the bundle identifier is unique to your account.
6. Compile once without code signing to catch errors:

   ```bash
   flutter build ios --release --no-codesign
   ```
7. Archive with **Product → Archive**, then **Distribute App** → App Store
   Connect / Ad Hoc / Development. `flutter build ipa --release` performs the
   archive and export in one step once signing is configured.
8. Upload to App Store Connect with **Transporter** or Xcode's organiser.

### iOS permission macros (Swift Package Manager)

`permission_handler_apple` compiles each iOS permission in or out behind a
`PERMISSION_*` macro. Under SPM it derives those macros from the matching
`NS*UsageDescription` keys in `ios/Runner/Info.plist`, so this app's four keys
(camera, microphone, photo-library read, photo-library add) enable the four
handlers. Two things to remember:

- **Changes are cached.** After editing `Info.plist`, clear DerivedData once so
  the package manifest is re-evaluated:
  `rm -rf ~/Library/Developer/Xcode/DerivedData`.
- **Archives started inside Xcode.app cannot locate the host app**, because
  manifest evaluation runs with `/` as its working directory. If a release build
  suddenly reports every permission as denied, archive with
  `flutter build ipa --release` (which runs from the project directory) or point
  the manifest at the plist explicitly before building:

  ```bash
  launchctl setenv PERMISSION_HANDLER_INFO_PLIST "$PWD/ios/Runner/Info.plist"
  rm -rf ~/Library/Developer/Xcode/DerivedData
  ```

  Set `PERMISSION_HANDLER_VERBOSE=1` to have the manifest print the plists and
  macros it resolved.

### Testing the camera on an iPhone

The iOS Simulator has no camera, so the camera module must be tested on a device.

1. Connect the iPhone, trust the Mac, and enable **Developer Mode**
   (Settings → Privacy & Security).
2. In Xcode, select the **Runner** target → *Signing & Capabilities* → choose your team.
3. `flutter run -d <iphone-id>`, open the camera, and check:
   - the rationale sheet appears before the system camera prompt;
   - denying, then reopening the screen shows **Camera access is blocked** with
     *Open settings* (iOS never re-prompts after a denial); enabling access in Settings
     and returning shows the preview without restarting;
   - rear camera by default; switching to the front camera hides the flash button;
   - flash cycles off → auto → on → torch on the rear camera; pinch and the slider zoom;
     tapping the preview shows the focus reticle;
   - rapid shutter taps capture one photo at a time;
   - Control Center / app switcher releases the camera (green indicator goes off) and
     returning reopens it.

---

## Permissions

Permissions are requested **only at the moment they are needed**, and a rationale
sheet explains the reason before the system prompt appears.

| Permission | When it is requested | Why |
| --- | --- | --- |
| Camera | On opening the shooting screen | Show the live preview that the reference image is aligned against. Used only while that screen is open. |
| Photo library (read) | Never prompted | Reference images come from the system Photo Picker (Android) / PHPicker (iOS), which need no permission. |
| Photo library (add) | When saving a capture | Write the finished photo or clip back to the gallery. |
| Microphone | The first time you start filming | Record sound into your clips. Photos never use it. |

Nothing leaves the device — there is no backend, account or analytics.

### Android — `android/app/src/main/AndroidManifest.xml`

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="29" />
<!-- Captured only while a clip is being filmed. -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-feature android:name="android.hardware.camera.any" android:required="true" />
```

`WRITE_EXTERNAL_STORAGE` is capped at API 29, the last version that needs it to save
into the gallery. API 30+ uses scoped storage, and the Android 13+ photo picker needs
no storage permission at all.

### iOS — `ios/Runner/Info.plist`

```xml
<key>NSCameraUsageDescription</key>
<string>…</string>
<key>NSMicrophoneUsageDescription</key>
<string>…</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>…</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>…</string>
```

With SPM, `permission_handler_apple` (9.6.2) **derives its `PERMISSION_*` macros from
these `Info.plist` keys automatically** — the only permissions declared here are the
ones compiled into the app. No Podfile macros are needed. If you edit the plist, clear
DerivedData once so the package manifest is re-evaluated:

```bash
rm -rf ~/Library/Developer/Xcode/DerivedData
```

---

## Architecture notes

- **State management** — Riverpod 3. `CameraSessionController` (an auto-dispose
  `AsyncNotifier`) owns the `CameraController`: it opens the rear camera by default,
  releases it when the app leaves the foreground and reopens the same lens on return,
  guards against overlapping captures, coalesces zoom calls, and owns the photo/video
  mode and the recording clock. Widgets never touch the plugin; `CameraService` is the
  only seam over `package:camera`. Camera providers opt out of Riverpod's automatic
  retry, so failures surface once with an explicit retry. Settings use a keep-alive
  `AsyncNotifier` with optimistic updates.
- **Framing** — `FrameGeometry` is the one place the capture frame, the focus mapping
  and the still crop are worked out, so the preview and the saved file always agree.
- **Routing** — a single `GoRouter` provided by `appRouterProvider`, with a shared
  `errorBuilder` for unknown routes.
- **Error handling** — every failure is normalised to a sealed `AppException`
  (`CameraFailure`, `PermissionFailure`, `GallerySaveFailure`, …) by `AppErrorHandler`,
  which also logs and offers a `guard()` helper. Framework, platform and zone errors are
  captured in `bootstrap.dart`.
- **Theme** — a reusable dark-first system in `core/theme` (`AppColors`, `AppSpacing`,
  `AppTypography`, `AppTheme`) plus a `CameraTheme` `ThemeExtension` for camera-specific
  tokens (scrim, shutter, guides). Controls meet the 48 dp minimum tap target and expose
  semantics labels.

---

## Testing

```bash
flutter analyze     # static analysis — expects zero issues
flutter test        # unit + widget tests
```

Camera tests drive the real `CameraController` against an in-memory
`CameraPlatform` fake (`test/features/camera/fakes`), covering default lens selection,
switching, capture concurrency, lifecycle release/reopen, error states and the
permission screens. Gallery tests cover the history store and deleted files:
metadata round-trips through a fake key-value store, unreadable or corrupt stored
data degrades to an empty history, and an entry whose file has gone is flagged
rather than failing.

Overlay tests cover the geometry (fit / fill / center sizing, drag / pinch /
rotate with their clamps, quarter-turn snapping), the settings controller and the
tray, plus an integration test that drives the overlay over the camera screen.
Widget tests cover the camera page (permissions, lifecycle release / reopen,
framing, capture and saving), the gallery, the home screen, the theme and the
shared widgets.

There is **no `integration_test/` device suite**: the camera, the gallery saver and
the photo library are covered by unit and widget tests against platform fakes, and
the real hardware is verified manually on a device (see **Testing the camera on an
iPhone**).

---

## Camera screen

The shooting screen is full-screen and always dark, whatever the app theme.
The preview fills the frame and the controls float on a soft scrim, so they
stay legible over any scene without boxing the feed in.

| Control | Behaviour |
|---|---|
| Close (top left) | Leaves the shooting screen |
| Recording clock (top left) | Appears beside close while a clip is filming: a red dot and the elapsed time |
| Ratio (top right) | Cycles 4:3 → 16:9 → 1:1 → Full; greys out while filming, because a clip records at the camera's native ratio |
| Grid (top right) | Shows or hides the rule-of-thirds guide; the choice is remembered |
| Flash (top right) | Cycles off → auto → on → torch on the rear camera; hidden on the front |
| Gallery (bottom left) | Opens the app's own history (see **Gallery and history**) |
| Reference (bottom left) | Chooses the reference image to float over the preview |
| Shutter (bottom centre) | Captures the camera's own image, or films a clip in video mode |
| Switch (bottom right) | Front / rear; hidden when the device has only one camera, and disabled while filming |
| Mode (bottom right) | Photo ⇄ video; the disc turns red in video mode and becomes a stop square while filming |
| Zoom pills | Lens stops (0.5× / 1× / 2×…) and drag-to-zoom; hidden when the camera cannot zoom |

Every control has a 48 dp minimum touch target and a semantics label, and the
bars respect the safe area in both portrait and landscape. On short (landscape)
screens the overlay tray starts folded to its header.

## Capture ratio

The ratio frames the shot: the preview is masked to it, and a photo is cropped to
match. It is a framing choice only — the camera keeps running at its own
resolution, and **switching it never restarts the session**.

| Mode | Shape |
|---|---|
| 4:3 | The default |
| 16:9 | Widescreen |
| 1:1 | Square |
| Full | Whatever the available preview area actually is, measured after the safe-area insets — never a hard-coded 9:16 |

The frame is the largest rect of the chosen shape inside the available preview
area. The rest of the camera stream is **dimmed, not hidden**, so the part of the
sensor the shot leaves out stays visible; the caption inside the frame shows the
camera stream's size and the size the capture will be
(`1920×1080 → 1440×1080`). Settings has a switch to hide the spill entirely.

The reference overlay, the alignment grid and the focus reticle are all scoped to
the frame, so the overlay stays aligned with the visible framing when the ratio
changes — and switching it never touches the overlay's opacity, position, scale,
rotation or lock state.

### Stills: crop only when it is needed

- If the selected ratio already matches the camera's output, the **camera's own
  bytes are saved untouched** — no decode, no re-encode.
- Otherwise the still is decoded, oriented, cropped to exactly the frame and
  encoded once at quality 95, off the UI thread. EXIF is dropped because the
  orientation is baked into the pixels.
- If the crop cannot be decoded or encoded, the original bytes are kept: a
  capture is never lost to a failed crop.

### Video

Clips are muxed by the platform recorder, which handles orientation and
audio/video synchronisation. **A clip is never transcoded**, so it is recorded at
the camera's native ratio (16:9 for 720p/1080p) — cropping a clip to an arbitrary
ratio needs a frame-accurate transcoder, which this app does not ship. The ratio
control says so rather than pretending to apply.

### Device notes

- `package:camera` exposes **no aspect-ratio API and no way to enumerate supported
  output sizes**, so a ratio is achieved by masking the preview and cropping the
  still in Dart. The live preview crop is therefore a presentation choice, while the
  sensor capture stays full-frame until the crop step.
- The crop assumes the still shares the preview stream's aspect ratio (true for
  `ResolutionPreset.veryHigh`). If a device disagrees, the crop is computed from the
  still's own dimensions and stays visually equivalent.
- `veryHigh` is 1080p, which is **16:9**, so on most devices 4:3 is a centred crop of
  the stream rather than a native capture size.
- Entering video mode is the one change that reopens the camera, because audio is only
  opened for video; the ratio is pure layout and never restarts the session.
- Re-encoding a cropped still drops EXIF (orientation is baked into the pixels).
  Uncropped captures keep their metadata.

## Settings

Preferences are kept on the device and restored the next time the app launches:

| Setting | Behaviour |
|---|---|
| Theme | System, dark or light; the camera screen stays dark either way |
| Preferred camera | The lens the shooting screen opens with (rear or front); you can still switch while shooting |
| Capture ratio | 4:3 (default) · 16:9 · 1:1 · Full |
| Sensor spill | Dim, rather than hide, the parts of the stream outside the frame |
| Alignment grid | Rule-of-thirds guide over the preview; also toggled from the camera bar |
| Reference image opacity | The opacity a new reference image starts at |
| Haptic feedback | Short vibrations when you tap a control or take a photo or clip |

The grid and the reference overlay are guides only — neither is ever part of a
captured photo.

---

## Reference overlay

Tap the reference button (the tile left of the shutter) to choose one reference
image. It is drawn between the live preview and the camera controls, so it can
never cover the shutter, flash, zoom or switch buttons. Its settings sit in a
glass tray above the zoom and shutter, so adjusting the overlay never hides the
controls you need; flick the tray down to shrink it to a chip.

| Control | Behaviour |
|---|---|
| Opacity tab | Slider 0–100% with the exact percentage; changing it never moves the image |
| 25 / 50 / 75 / 100% | Opacity presets; ↺ resets to the default opacity from Settings |
| Drag / pinch / two-finger twist | Move, resize and rotate (anywhere on the preview while unlocked); rotation snaps to quarter turns within 4° |
| Frame tab: Fit · Fill · Center | Whole image visible · covers the screen, cropping edges (the only mode that crops) · original pixel size; all keep the aspect ratio |
| Reset position | Undoes drag, pinch and rotation within the current mode |
| Lock | Freezes the overlay; camera tap-to-focus and pinch-zoom work again |
| Eye | Hide / show without losing any settings |
| Trash | Remove the image and free its memory |
| Minimize | Tap the handle on top of the tray (or flick it down) to shrink it to a small chip showing the opacity, lock and hidden state; tap the chip (or flick up) to restore it on the same tab |

Notes:

- **The overlay is never part of the photo.** Capture saves exactly what the camera
  produced; a test compares the saved bytes with the camera output.
- Images are decoded once at no more than 2048 px on the longest side, so a 48 MP photo
  costs ~12 MB of GPU memory rather than ~190 MB. EXIF orientation is applied. Unreadable
  or unsupported files show a message and keep the current image.
- Position is stored as a fraction of the screen, so it survives rotation. On short
  (landscape) screens the options start collapsed to the opacity row.
- While the overlay is unlocked, pinching resizes the overlay; to zoom the camera, tap a
  lens stop (0.5× / 1× / 2×…) or drag across the zoom pill, or lock the overlay.

## Capture and saving

Tap the shutter to capture through the active `CameraController`. When the capture
ratio already matches what the camera produces, the camera's own file is written to
the system photo library **byte for byte** — no resize, no re-encode. Otherwise it is
cropped to the frame the preview showed with a single high-quality re-encode (see
**Capture ratio**). Either way the app stays on the live preview: there is no review
screen and nothing to dismiss, because the photo is already in the gallery when the
flash clears.

In video mode the same button starts filming and then stops it; the platform recorder
writes the clip and it is moved into the app's own captures directory as `Veylora_<ms>.mp4`
and handed to the library untouched.

The reference overlay is a guide only. It is never composited into a photo or a clip.
Saving requests the photo library permission only at that moment, with a rationale
sheet first, and the microphone is requested only when the first clip starts. Both stay
quiet afterwards — the flash and the haptic are the confirmation, and a message only
appears when something goes wrong. The shutter is guarded while a capture is in flight,
so repeated taps cannot save twice. The app keeps its own copy of every capture, which
is what the gallery below lists.

## Gallery and history

**Gallery** on the home screen opens the app's own history, split into three tabs:

| Tab | What it holds |
|---|---|
| Photos | Every photo this app captured, from the app's own copy of the file |
| Videos | Every clip this app filmed |
| References | Every reference image that has been floated over the preview, so it can be picked again without reopening the system picker |

Photo thumbnails are decoded at thumbnail size, so a 12 MP file never lands in memory
just to draw a small square; a clip is drawn as a play glyph (poster frames are a
follow-up). Tapping one opens a full-screen preview offering:

| Action | Behaviour |
|---|---|
| Share | Opens the system share sheet (`share_plus`), anchored to the button on iPad |
| Delete | Forgets the entry and deletes the app's copy; files already in the system library are untouched |
| Play / pause | Clips only, played by `video_player` with the orientation and audio they were recorded with |
| Use as reference | Reference images only: loads it straight back onto the camera as the guide |

Only a path, a kind, an export flag and a timestamp are stored — a JSON list in
shared preferences — so the history stays light and the media stays on disk.
Files can be removed outside the app, so every entry is checked on load: a missing
file is marked **Missing** and the preview explains it instead of failing.
Nothing leaves the device: there is no account, no synchronisation and no backend.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Failed to find target with hash string 'android-37'` | API 37 ships as `platforms;android-37.0`. Keep `compileSdkMinor = 0` in `android/app/build.gradle.kts` and install that package. |
| `NDK ... did not have a source.properties file` | A partial NDK download. Delete `…/sdk/ndk/28.2.13676358` (or run `sdkmanager "ndk;28.2.13676358"`) and let AGP install it again. |
| `Could not close incremental caches` (Windows) | The project and the pub cache are on different drives. Add `kotlin.incremental=false` to your user-level `~/.gradle/gradle.properties`. |
| `The SDK directory is not writable` | Run `sdkmanager` from an elevated prompt, or point `sdk.dir` in `android/local.properties` at a writable SDK. |
| Build warns the `CupertinoIcons` font is missing | Keep `cupertino_icons` in `pubspec.yaml`; it is the font Flutter's adaptive widgets use on iOS. |
| Camera screen says access is blocked | The user denied permanently. **Open settings** deep-links to system settings; iOS never re-prompts after a denial. |
| Captures never reach the system gallery | The photo-library add permission was refused, or storage is full. Saving asks with a rationale first and reports storage errors. |
| A history tile says **Missing** | The file was removed outside the app. The row is kept and the preview explains it; delete it to clear the row. |
| The Android release build is debug-signed | No `android/key.properties` — see **Release steps**. |
| iOS: every permission reports denied | The SPM manifest could not locate the app. Build with `flutter build ipa`, or set `PERMISSION_HANDLER_INFO_PLIST`, then clear DerivedData. |
| A capture is saved uncropped although a ratio is selected | The still already matched the ratio (nothing to do) or the crop failed — a capture is never lost to a failed crop. The log says `The capture could not be cropped`. |

## Release steps

### Android

1. Create an upload keystore once and keep it — and its passwords — safe: losing
   it means the app can no longer be updated on Google Play.

   ```bash
   keytool -genkey -v -keystore ~/veylora-upload.jks -keyalg RSA \
           -keysize 2048 -validity 10000 -alias upload
   ```

2. Create `android/key.properties` (already git-ignored):

   ```properties
   storePassword=<your password>
   keyPassword=<your password>
   keyAlias=upload
   storeFile=C:/path/to/veylora-upload.jks
   ```

   `android/app/build.gradle.kts` reads this file when it exists and signs the
   release with it. Without the file, the release build stays debug-signed so
   `flutter run --release` keeps working on a fresh checkout.

3. Bump the version in `pubspec.yaml` (`version: X.Y.Z+N`) and build:

   ```bash
   flutter build apk --release          # APK
   flutter build appbundle --release    # AAB for Google Play
   ```

4. Confirm the certificate is your upload key, not `CN=Android Debug`:

   ```bash
   apksigner verify --print-certs build/app/outputs/flutter-apk/app-release.apk
   ```

### iOS

1. Set the bundle identifier and signing team in Xcode (see **Building and
   shipping on iOS**).
2. Bump the version in `pubspec.yaml`; Flutter writes it into
   `CFBundleShortVersionString` / `CFBundleVersion`.
3. Archive with `flutter build ipa --release`, or **Product → Archive** in Xcode.
4. Upload to App Store Connect (Transporter or Xcode), then TestFlight / App Store.
5. App Review note: the camera and microphone usage descriptions are visible in
   `ios/Runner/Info.plist`; the app uses the system photo picker without read
   permission and asks for photo-library add access only when saving. Nothing is
   uploaded.

## Brand assets

The logo is a "V" made of two layers, an amber arm and a cream arm crossing at
the vertex, echoing a reference lined up over the live view. Every asset (master
SVGs in `assets/branding/`, the in-app mark, Android legacy/adaptive/themed
launcher icons, the iOS AppIcon set and both launch screens) is generated from one
definition:

```bash
pip install pillow
python tool/brand/generate_brand_assets.py
```

