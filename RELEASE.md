# Releasing Veylora

Veylora ships through **GitHub Releases**. Pushing a version tag such as
`v1.0.0` runs `.github/workflows/release.yml`, which:

1. installs the pinned Flutter SDK and dependencies;
2. runs `flutter analyze` and the full test suite (the release fails if either fails);
3. builds Android artifacts:
   - **all signing secrets present** → a release-signed APK **and** a production AAB;
   - **signing secrets missing** → a debug-signed APK named `…-UNSIGNED-DEBUG.apk`
     (installable, testing only); the workflow still succeeds;
4. builds a **signed** iOS IPA *only when Apple signing is configured* — otherwise
   the iOS job is skipped and the release notes say so;
5. creates a GitHub Release with generated release notes (never an empty one);
6. attaches the downloadable assets.

> Asset names keep the tag exactly as pushed, e.g. `Veylora-v1.0.0-Android.apk`,
> `Veylora-v1.0.0-Android.aab`, `Veylora-v1.0.0-Android-UNSIGNED-DEBUG.apk`,
> `Veylora-v1.0.0-iOS.ipa`. Change `APP_NAME` at the top of
> `.github/workflows/release.yml` to rename the prefix.

---

## 1. Create and push a version tag

Releases are driven entirely by the tag. `v1.0.0` becomes app version `1.0.0`
and the Android/iOS build number is the workflow run number.

```bash
git checkout main
git pull

# Optional: keep local builds in step with the release.
# pubspec.yaml: version: 1.0.0+1

git tag v1.0.0
git push origin v1.0.0
```

That push is the trigger. To release a fix, tag the next semantic version
(`v1.0.1`), then `v1.1.0`, and so on. Never reuse or move a published tag.

To retry, open the failed run under **Actions** and use **Re-run all jobs**.
While iterating you can also delete and re-push the tag (`git push origin
:v1.0.0`, then push it again).

---

## 2. GitHub Secrets

Add these under **Repository → Settings → Secrets and variables → Actions →
New repository secret**. The workflow only ever writes them to the ephemeral
runner; never commit any of them.

Every secret is optional: without them the workflow still publishes an
installable (debug-signed) Android APK for testing. Add the Android secrets to
publish a production build.

### Android (recommended)

| Secret | Value |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | base64 of the upload keystore `.jks` |
| `ANDROID_KEYSTORE_PASSWORD` | the keystore password |
| `ANDROID_KEY_ALIAS` | the key alias (e.g. `upload`) |
| `ANDROID_KEY_PASSWORD` | the key password |

Create the keystore once (keep it and its passwords safe — losing it means you
can never update the app on Google Play):

```bash
keytool -genkey -v -keystore veylora-upload.jks \
        -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

Encode it as a single line:

```bash
# macOS / Linux
base64 < veylora-upload.jks | tr -d '\n'
```

```powershell
# Windows PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("veylora-upload.jks"))
```

The workflow verifies the built APK:

- **secrets present** → the APK must **not** be debug-signed, or the job fails;
- **secrets absent** → the APK is expected to be debug-signed and is published as
  `…-Android-UNSIGNED-DEBUG.apk`, clearly marked as a testing build.

The fallback is produced by the same `flutter build apk --release`. The Gradle
config falls back to the debug signing key when `android/key.properties` is
absent, so the APK is a real release build (AOT, no debugging) — just signed with
a throwaway key. It installs normally on a device, but it cannot be uploaded to
Google Play.

### iOS (optional)

The iOS job runs **only** when `IOS_CERTIFICATE_P12` is present. Without these
secrets the release still succeeds and publishes Android assets only — it never
uploads an unsigned or fake IPA.

| Secret | Value |
| --- | --- |
| `IOS_CERTIFICATE_P12` | base64 of an **Apple Distribution** certificate + private key exported as `.p12` from Keychain Access |
| `IOS_CERTIFICATE_PASSWORD` | the `.p12` export password |
| `IOS_PROVISIONING_PROFILE` | base64 of the **App Store** distribution `.mobileprovision` for `com.veylora.app` |
| `IOS_PROVISIONING_PROFILE_NAME` | that profile's **Name** (as shown in the Apple Developer portal) |
| `APPLE_TEAM_ID` | your 10-character Apple Developer Team ID |

Encode the files the same way (`base64 < file | tr -d '\n'`).

The certificate, profile and bundle identifier must all agree:

- bundle id: `com.veylora.app` (see `BUNDLE_ID` in the workflow);
- profile: an **App Store** distribution profile for that bundle id;
- certificate: **Apple Distribution**, matching the profile.

If you change the bundle identifier, update it in **all** of: the workflow's
`BUNDLE_ID`, `ios/Runner.xcodeproj` (`PRODUCT_BUNDLE_IDENTIFIER`), the Android
`applicationId`/`namespace`, and the provisioning profile.

### What gets published

| Secrets | Android assets | iOS asset | Release state |
| --- | --- | --- | --- |
| Android + iOS | `Veylora-vX.Y.Z-Android.apk`, `…-Android.aab` | `Veylora-vX.Y.Z-iOS.ipa` | normal release |
| Android only | `Veylora-vX.Y.Z-Android.apk`, `…-Android.aab` | none (job skipped, noted in the release) | normal release |
| None / incomplete | `Veylora-vX.Y.Z-Android-UNSIGNED-DEBUG.apk` | none | **pre-release**, titled “… (unsigned debug build)” |

A debug-signed APK is never named `…-Android.apk`, and a testing release is never
titled as a production build. The release is not created at all if no assets were
produced.

---

## 3. Download the Android APK

1. Open the repository's **Releases** page.
2. Pick the release (e.g. **Veylora v1.0.0**).
3. Download the APK:
   - production build: `Veylora-v1.0.0-Android.apk` (release-signed);
   - testing fallback: `Veylora-v1.0.0-Android-UNSIGNED-DEBUG.apk` (debug-signed).
4. On the phone, allow **Install unknown apps** for the browser/file manager,
   then open the file to install. Both files install normally.

The `.aab` is not installable on a device — it is the upload format for Google
Play Console:

```bash
# Google Play Console → your app → Production → Create new release →
# upload Veylora-v1.0.0-Android.aab
```

`adb install` also works for the APK:

```bash
adb install -r Veylora-v1.0.0-Android.apk
```

---

## 4. iOS: install limitations and TestFlight

> **A downloaded IPA cannot simply be installed on any iPhone.**

The IPA produced here is an **App Store distribution** build. It is signed for
Apple's store, not for direct installation, so iOS will refuse to install it by
tapping it. Downloading the `.ipa` from the release is useful for archiving and
for uploading to App Store Connect — not for sideloading.

What that means in practice:

- **Recommended path — TestFlight.** Upload the build to App Store Connect; testers
  install it through the TestFlight app. This is the normal way to put a build on
  iPhones without a Mac.
- **Direct install (ad-hoc).** Requires a different export method (`ad-hoc`), a
  provisioning profile that lists each device's UDID, and a compatible certificate.
  Only those registered devices can install it. Not what this workflow produces.
- **Sideloading tools** (AltStore, Sideloadly) re-sign the app with a developer
  certificate. Free Apple IDs expire after 7 days, and the app must be re-signed;
  this is not an official distribution channel.
- The iOS Simulator has no camera, so camera testing needs a real device.

### Uploading to TestFlight

1. Download `Veylora-v1.0.0-iOS.ipa` from the release (or archive locally).
2. Open **Transporter** (Mac App Store) and sign in with your Apple Developer
   account.
3. Drag the `.ipa` in and click **Deliver**.
4. In **App Store Connect → TestFlight**, wait for processing, then add internal
   or external testers.
5. Testers install **TestFlight** from the App Store, accept the invite, and
   install Veylora.

Alternatively, in Xcode: **Window → Organiser → Distribute App → App Store
Connect → Upload**.

---

## 5. Troubleshooting

| Symptom | Fix |
| --- | --- |
| Workflow did not run | The tag must match `v*` and be pushed (`git push origin v1.0.0`). Check **Actions**. |
| Release contains only an `…-UNSIGNED-DEBUG.apk` | Expected when the `ANDROID_*` secrets are missing or incomplete. The build succeeds by design; add the secrets to publish a production APK + AAB. |
| Release is a pre-release titled “(unsigned debug build)” | Same cause: no complete Android signing secrets. The marking is intentional. |
| “Signing secrets were configured but the APK is debug-signed” error | Signing secrets are set but wrong: check `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`; the keystore must decode into `android/app/upload-keystore.jks`. |
| “No release assets were produced” error | The Android job produced no APK. Read its log; the release job refuses to publish an empty release. |
| `failed to strip debug symbols from native libraries` when building the AAB | Flutter's post-build check needs Android `cmdline-tools` (it runs `apkanalyzer`). GitHub runners ship it; a local SDK may not — install it with `sdkmanager "cmdline-tools;latest"`. |
| `Failed to find target with hash string 'android-37'` | API 37 ships as `android-37.0`; the project sets `compileSdkMinor = 0`. If you edited Gradle, restore it; the workflow also installs the SDK components. |
| Gradle cannot install SDK components | The runner's SDK must be writable and licences accepted; the workflow runs `sdkmanager --licenses`. |
| iOS job shows **skipped** | `IOS_CERTIFICATE_P12` is not set. That is expected when only Android is configured. |
| iOS build: “No signing certificate found” | The `.p12` must contain an **Apple Distribution** certificate and its private key. |
| iOS build: “No profiles for `com.veylora.app` were found” | The provisioning profile is not an App Store profile for that bundle id, or `IOS_PROVISIONING_PROFILE_NAME` does not match its Name. |
| iOS export: “exportArchive … method” error | Newer Xcode renamed the export method. Change `app-store` to `app-store-connect` in the workflow's `Generate export options` step. |
| Every iOS permission reports denied at runtime | The SwiftPM manifest could not locate the app. Build with `flutter build ipa` (the workflow does) or set `PERMISSION_HANDLER_INFO_PLIST`; see the README. |
| iOS signed build failed | No release is published: the release job runs only when iOS succeeded or was skipped, so a broken iOS build cannot ship a half release. Fix the signing secrets and re-run the tag. |
| Release created but an asset is missing | Download the failed job's log. Asset upload only happens after the platform job succeeds; `fail_on_unmatched_files` makes a missing file fail the release. |
| Need to retry a release | Delete the release and tag in GitHub, or re-run the workflow; publishing the same tag again updates the release. |

### Notes

- Do not change a published tag or asset checksums — users and stores pin them.
- Signing material lives only in GitHub Secrets and on the ephemeral runner.
  `.gitignore` excludes `key.properties`, `*.jks` and `*.keystore`; never add
  certificates, passwords or provisioning profiles to the repository.
- The workflow pins Flutter `3.44.6`. Bump `FLUTTER_VERSION` in
  `.github/workflows/release.yml` deliberately, and run the tests locally first.
- A release is published only when the Android job succeeds (and iOS either
  succeeded or was skipped). The release job refuses to create an empty release.
- An unsigned testing release is marked **pre-release** and titled
  “(unsigned debug build)” so it is never mistaken for a production build.
