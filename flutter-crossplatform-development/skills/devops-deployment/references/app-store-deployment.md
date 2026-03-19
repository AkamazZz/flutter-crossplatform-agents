---
description: >
  App Store and Play Store deployment automation, code signing, and release management.
---

# App Store Deployment

Deploying Flutter apps to production stores requires platform-specific signing, packaging, and submission steps. This document covers iOS App Store, Google Play, web hosting, and desktop distribution.

## iOS: Certificates, Provisioning, TestFlight, and App Store Connect

### Certificate Types

- Apple Development — local device testing
- Apple Distribution — TestFlight + App Store submission

### Provisioning Profile Types

- Development — run on registered devices
- Ad Hoc — distribute to up to 100 devices
- App Store — TestFlight + App Store submission

### Setup Steps

1. **Create App ID** in Apple Developer Portal (Identifiers → App IDs). Enable only the capabilities you need (Push Notifications, Sign In with Apple, etc.).

2. **Create distribution certificate** (Certificates → + → Apple Distribution). Download and install in Keychain.

3. **Create App Store provisioning profile** (Profiles → + → App Store Distribution). Select your App ID and distribution certificate. Download.

4. **Configure Xcode:**
   - Open `ios/Runner.xcworkspace`
   - Set bundle identifier to match your App ID
   - Under Signing & Capabilities, select your team and provisioning profile (or use automatic signing for development)

5. **ExportOptions.plist** (for CLI builds):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>XXXXXXXXXX</string>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.example.app</key>
        <string>MyApp App Store Profile</string>
    </dict>
</dict>
</plist>
```

### Build and Upload

```bash
# Build IPA
flutter build ipa \
  --flavor prod \
  --target lib/main_prod.dart \
  --release \
  --export-options-plist=ios/ExportOptions.plist

# Upload to App Store Connect (Transporter or xcrun)
xcrun altool --upload-app \
  --type ios \
  --file build/ios/ipa/Runner.ipa \
  --apiKey $APP_STORE_CONNECT_KEY_ID \
  --apiIssuer $APP_STORE_CONNECT_ISSUER_ID

# Or use Fastlane deliver
fastlane deliver --ipa build/ios/ipa/Runner.ipa --skip_metadata --skip_screenshots
```

### TestFlight Distribution

1. Upload build via Xcode Organizer, `xcrun altool`, or Fastlane.
2. In App Store Connect → TestFlight, add internal testers (up to 100, no review required).
3. For external testing (up to 10,000 testers), submit for Beta App Review (usually 24–48 h).
4. Set test notes and expiry (builds expire 90 days after upload).

### App Store Connect Submission Checklist

- Screenshots for all required device sizes (6.7", 6.5", 5.5" for iPhone; 12.9" for iPad if supported)
- App description, keywords, support URL, privacy policy URL
- Privacy nutrition labels (data types collected and linked to user)
- Age rating questionnaire
- Export compliance (if using encryption beyond standard HTTPS, declare in `Info.plist`)

```xml
<!-- ios/Runner/Info.plist — declare standard encryption use -->
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

## Google Play: Upload Key, Play Console Tracks, Staged Rollout

### Generating the Upload Key

```bash
keytool -genkey -v \
  -keystore ~/upload-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias upload
```

Store the keystore securely (password manager, CI secrets). If you lose it before Play App Signing enrollment, you cannot update your app.

**Enroll in Google Play App Signing** (strongly recommended): Google holds the App Signing key; you use the upload key to sign bundles for submission. If your upload key is compromised, you can request a reset.

### android/key.properties

```properties
storePassword=your_store_password
keyPassword=your_key_password
keyAlias=upload
storeFile=/path/to/upload-keystore.jks
```

```groovy
// android/app/build.gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
        }
    }
}
```

### Build and Upload

```bash
flutter build appbundle \
  --flavor prod \
  --target lib/main_prod.dart \
  --release

# Upload via bundletool or Play Console UI, or:
# fastlane supply --aab build/app/outputs/bundle/prodRelease/app-prod-release.aab --track internal
```

### Play Console Release Tracks

- Internal testing — up to 100 testers, instant publish; smoke testing every build
- Closed testing (Alpha) — specific Google Groups or emails; QA team, select users
- Open testing (Beta) — anyone who opts in; public beta, wider feedback
- Production — all users; full release

### Staged Rollout

Limit production releases to a percentage of users to catch issues before full exposure:

1. In Play Console → Production → Create new release
2. Set rollout percentage (e.g., 10%)
3. Monitor ANRs and crash rate in Android Vitals
4. Increase rollout percentage in increments (10% → 25% → 50% → 100%)
5. Halt rollout immediately if crash rate spikes

## Web: Firebase Hosting, Vercel, and Cloudflare Pages

### Build

```bash
flutter build web \
  --release \
  --dart-define=API_URL=https://api.example.com \
  --web-renderer canvaskit   # or 'html' for smaller bundle size
```

Output: `build/web/`

### Firebase Hosting

```bash
npm install -g firebase-tools
firebase login
firebase init hosting
# Set public directory to: build/web
# Configure as single-page app: Yes
# Set up GitHub Actions deploys: optional

flutter build web --release
firebase deploy --only hosting
```

```json
// firebase.json
{
  "hosting": {
    "public": "build/web",
    "ignore": ["firebase.json", "**/.*"],
    "rewrites": [{ "source": "**", "destination": "/index.html" }],
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [{ "key": "Cache-Control", "value": "max-age=31536000, immutable" }]
      }
    ]
  }
}
```

### Vercel

```json
// vercel.json
{
  "buildCommand": "flutter build web --release",
  "outputDirectory": "build/web",
  "installCommand": "curl -fsSL https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_3.22.0-stable.tar.xz | tar xz && export PATH=$PATH:$PWD/flutter/bin && flutter pub get",
  "routes": [{ "src": "/(.*)", "dest": "/index.html" }]
}
```

Alternatively, use the Vercel GitHub integration with a pre-built `build/web` directory committed to a deployment branch.

### Cloudflare Pages

Connect your GitHub repository in the Cloudflare Pages dashboard:
- Build command: `flutter build web --release`
- Build output directory: `build/web`
- Environment variable: set `FLUTTER_VERSION=3.22.0`

Use Cloudflare Pages Functions for server-side logic or redirects.

## Desktop: DMG, MSI/MSIX, AppImage, Code Signing, and Notarization

### macOS — DMG

```bash
flutter build macos --release

# Package as DMG using create-dmg
brew install create-dmg
create-dmg \
  --volname "MyApp" \
  --window-pos 200 120 \
  --window-size 800 400 \
  --icon-size 100 \
  --icon "MyApp.app" 200 190 \
  --app-drop-link 600 185 \
  "MyApp-1.0.0.dmg" \
  "build/macos/Build/Products/Release/MyApp.app"
```

**Code signing:**
```bash
codesign --deep --force --verify --verbose \
  --sign "Developer ID Application: Your Name (TEAMID)" \
  --options runtime \
  build/macos/Build/Products/Release/MyApp.app
```

**Notarization (required for Gatekeeper):**
```bash
xcrun notarytool submit MyApp-1.0.0.dmg \
  --apple-id your@email.com \
  --team-id TEAMID \
  --password $APP_SPECIFIC_PASSWORD \
  --wait

xcrun stapler staple MyApp-1.0.0.dmg
```

### Windows — MSI/MSIX

```bash
flutter build windows --release

# Create MSIX using msix package
dart pub global activate msix
dart run msix:create
```

```yaml
# pubspec.yaml MSIX configuration
msix_config:
  display_name: MyApp
  publisher_display_name: Example Corp
  identity_name: com.example.myapp
  msix_version: 1.0.0.0
  logo_path: assets/icons/icon.png
  capabilities: internetClient
  certificate_path: cert.pfx
  certificate_password: $CERT_PASSWORD
```

**Code signing (requires Authenticode certificate):**
```bash
signtool sign /fd SHA256 /a /f cert.pfx /p $CERT_PASSWORD MyApp.msix
```

Distribute via Microsoft Store (submit through Partner Center) or direct MSIX download.

### Linux — AppImage

```bash
flutter build linux --release

# Create AppImage using appimagetool
wget -O appimagetool https://github.com/AppImage/AppImageKit/releases/latest/download/appimagetool-x86_64.AppImage
chmod +x appimagetool

# Structure the AppDir
mkdir -p MyApp.AppDir/usr/bin
cp -r build/linux/x64/release/bundle/* MyApp.AppDir/usr/bin/
# Add .desktop file and icon at AppDir root

./appimagetool MyApp.AppDir MyApp-1.0.0-x86_64.AppImage
```

Distribute via direct download, Snap Store (`snapcraft`), or Flathub (`flatpak-builder`).

### Desktop Distribution Summary

- macOS — DMG / PKG — Developer ID + Notarization — Mac App Store or direct
- Windows — MSIX / EXE — Authenticode certificate — Microsoft Store or direct
- Linux — AppImage / Flatpak / Snap — GPG (optional) — Flathub / Snap Store / direct
