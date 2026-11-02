# Releasing Veylora

Veylora ships through **GitHub Releases**. Pushing a version tag such as
`v1.0.0` runs `.github/workflows/release.yml`, which:

1. installs the pinned Flutter SDK and dependencies;
2. runs `flutter analyze` and the full test suite (the release fails if either fails);
3. builds a **signed** Android APK and AAB;
4. builds a **signed** iOS IPA *when Apple signing is configured*;
5. creates a GitHub Release with generated release notes;
6. attaches the downloadable assets.

> Assets are named `Veylora-<version>-Android.apk`, `Veylora-<version>-Android.aab`
> and `Veylora-<version>-iOS.ipa`. The request contained an `Ovrilo-…` example;
> the workflow uses the project's actual brand. Change `APP_NAME` at the top of
> `.github/workflows/release.yml` to rename every artifact.

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

## 2. Required GitHub Secrets

Add these under **Repository → Settings → Secrets and variables → Actions →
New repository secret**. The workflow only ever writes them to the ephemeral
runner; never commit any of them.

### Android (required)

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

The workflow also verifies the built APK: if it is ever signed with the Android
debug key (`CN=Android Debug`), the release fails instead of publishing an
unpublishable artifact.

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

---

## 3. Download the Android APK

1. Open the repository's **Releases** page.
2. Pick the release (e.g. **Veylora v1.0.0**).
3. Download `Veylora-v1.0.0-Android.apk`.
4. On the phone, allow **Install unknown apps** for the browser/file manager,
   then open the file to install.

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
| `Android release` fails before building | A missing `ANDROID_*` secret. The workflow stops rather than falling back to debug signing. |
| “signed with the debug key” error | `ANDROID_KEYSTORE_BASE64`, password or alias is wrong; the keystore did not decode into `android/app/upload-keystore.jks`. |
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
