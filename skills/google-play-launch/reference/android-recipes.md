# Android recipes

Exact commands and file templates for [SKILL.md](../SKILL.md)'s Step 3. Nothing here needs reading top to bottom — jump to the section for the manifest item you're implementing.

## Scaffold commands

The Step 1 detection ladder, expanded:

| Detected | Command | Produces |
|---|---|---|
| `android/` with Gradle files present | (nothing — already scaffolded) | — |
| `@capacitor/core` in `package.json` | `npx cap add android` | `android/` as a Gradle project with `app/` module |
| `expo` in `package.json` | `npx expo prebuild -p android` | `android/` as a plain Gradle project |
| `react-native` in `package.json`, `android/` missing | — report, don't scaffold | — |
| `pubspec.yaml` present | `flutter create --platforms=android .` | `android/` wired to the existing Flutter project |
| Web app, no native shell yet | `npm i -D @capacitor/core @capacitor/cli @capacitor/android && npx cap init && npx cap add android` | Same as the Capacitor row above |

After any scaffold step, run `npx cap sync android` (Capacitor) or the stack's equivalent before touching native files — it's what copies the web build into `android/app/src/main/assets/public` and keeps `AndroidManifest.xml`/`capacitor.config.json` in sync.

## Icon + splash + feature graphic

**Pipeline (Capacitor):**

1. Get or make a source icon: a single square SVG or a high-res PNG. If none exists, ask the user for one before fabricating brand art from scratch.
2. Raster it to `resources/icon.png` at exactly 1024×1024 (the generator downsamples from there):
   ```
   # from SVG
   rsvg-convert -w 1024 -h 1024 icon-source.svg -o resources/icon.png
   # or, from a large PNG
   sips -z 1024 1024 icon-source.png --out resources/icon.png
   ```
   Unlike Apple, **Google allows alpha** in the 512×512 store icon and applies its own corner mask — ship a full-square icon and let Play round it. Don't pre-round corners.
3. Generate `resources/splash.png` and `resources/splash-dark.png` at 2732×2732 — a simple centered mark on a solid background; Android 12+ crops it into the system splash circle, so keep the mark inside the middle third.
4. Generate everything from those three source files:
   ```
   npx @capacitor/assets generate --android
   ```
   This writes the launcher icons into every `res/mipmap-*` density (including the adaptive-icon foreground/background layers), the splash variants into `res/drawable*`, and a 512×512 `icon-only.png` you can upload as the Play store icon.
5. `npx cap sync android` so the resource changes are picked up by the Gradle project.

**Feature graphic — Play-only, required, and no generator makes it.** Play requires a 1024×500 JPEG/PNG "feature graphic" on every store listing. Compose one from the icon + a brand background:

```
# ImageMagick: solid brand background + centered icon at ~360px
magick -size 1024x500 xc:"#1b1e22" \
  \( resources/icon.png -resize 360x360 \) -gravity center -composite \
  store-assets/feature-graphic.png
```

Better: put the app name in the graphic too (text-drawn or from a wordmark asset) — this image is what shows at the top of the listing and in editorial placements.

**Non-Capacitor fallback** (native/Expo/RN/Flutter projects, or no `@capacitor/assets`): Android Studio's **Image Asset Studio** (right-click `res` → New → Image Asset) generates the full adaptive-icon set from one source image; Expo/RN projects instead set `icon`/`splash`/`adaptiveIcon` paths in `app.json` and prebuild regenerates the resources for you. Adaptive icon spec: [developer.android.com — adaptive icons](https://developer.android.com/develop/ui/views/launch/icon_design_adaptive); the foreground layer's safe zone is the middle 66dp of 108dp — keep the mark inside it or launchers crop it.

## `AndroidManifest.xml`

Minimum edits for orientation lock, and the permission audit. Merge into the existing elements — don't replace the file (path: `android/app/src/main/AndroidManifest.xml`):

```xml
<!-- Portrait-only example — on the MAIN activity element -->
<activity
    android:name=".MainActivity"
    android:screenOrientation="portrait"
    ... >
```

Then audit permissions. The **merged** manifest is what Play sees — libraries inject their own permissions, so check the merge, not just the source file:

```
cd android && ./gradlew :app:processDebugMainManifest 2>/dev/null
# merged output lands under app/build/intermediates/merged_manifests/debug/
grep uses-permission app/build/intermediates/merged_manifests/debug/*/AndroidManifest.xml
```

- Remove any `<uses-permission>` the app doesn't genuinely need; each one shows on the store listing and widens the Data safety form.
- If the app posts local notifications and targets Android 13+ (it does, given Play's target-API floor), `POST_NOTIFICATIONS` must be declared **and** requested at runtime — declaring it alone silently no-ops.
- Restricted permissions (`READ_SMS`, `READ_CALL_LOG`, `MANAGE_EXTERNAL_STORAGE`, `QUERY_ALL_PACKAGES`, `ACCESS_BACKGROUND_LOCATION`) each trigger a Play Console declaration form and a likely rejection without a core-functionality justification — flag them in the plan, don't just ship them.

## Version & identity (`build.gradle`)

All three identity facts live in `android/app/build.gradle` under `defaultConfig`:

```groovy
defaultConfig {
    applicationId "com.yourname.appname"   // permanent once shipped; com.example.* is refused
    versionCode 2        // integer, must strictly increase on EVERY upload
    versionName "1.1.0"  // the human-readable version users see
    targetSdkVersion 36  // must meet Play's current floor — check the target API level page
    ...
}
```

Cross-stack note: Capacitor reads `appId` from `capacitor.config.json` at `cap add` time only — after that, `build.gradle` is the source of truth. Expo/RN mirror these in `app.json`; keep them agreeing or the built app and the store record disagree. Play's target-API floor rises every August 31 — never trust a hardcoded number, check [the requirement page](https://support.google.com/googleplay/android-developer/answer/11926878).

## Upload keystore + signing config

**Scaffold the wiring; the user creates the keystore.** The passwords must be theirs, and the file must be backed up somewhere that survives a dead laptop. Give them this command to run:

```
keytool -genkeypair -v \
  -keystore upload-keystore.jks \
  -alias upload \
  -keyalg RSA -keysize 2048 -validity 10000
```

Then wire Gradle to read it from an untracked properties file. `android/key.properties` (NEVER committed):

```properties
storeFile=/absolute/path/to/upload-keystore.jks
storePassword=<theirs>
keyAlias=upload
keyPassword=<theirs>
```

`android/app/build.gradle` additions:

```groovy
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            if (keystorePropertiesFile.exists()) {
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
            }
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            ...
        }
    }
}
```

And the `.gitignore` entries (add to `android/.gitignore`):

```
key.properties
*.jks
*.keystore
```

With Play App Signing (the default for new apps), this keystore is only the **upload key** — Google holds the real app signing key, and a lost upload key can be reset through Play Console support. Still treat it as a secret: anyone holding it can push builds to your testing tracks.

## Privacy facts prep (`docs/STORE-FACTS.md`)

One file, one source of truth. Google's Data safety form, the account-deletion answer, and the privacy policy must all agree — write the grepped facts once and derive all three. Structure the doc as the form's own questions:

- **Collected?** For each data type (location, personal info, identifiers, app activity…): does the app or any SDK send it off the device? (`fetch(`/`XMLHttpRequest`/`WebSocket` targets, SDK docs for anything linked.)
- **Shared?** Does anything go to a third party (analytics, ads, crash reporting)?
- **Encrypted in transit?** (HTTPS-only is "yes".)
- **Deletable?** Can the user request deletion — and if accounts exist, where are the in-app path and the web link?
- **Local-only storage** — list what's persisted on-device (localStorage keys, files, databases); local-only data is *not* "collected" in Play's definition, but the privacy policy should still name it.

## Screenshots

**Web-shell app (Capacitor/similar):** Playwright with the Chromium engine (Android's WebView is Chromium), run against the dev server or a built preview. Play is flexible on exact sizes — JPEG or 24-bit PNG (no alpha), each side between 320px and 3840px — but there's also a hard aspect-ratio cap that's easy to miss: **the longer side can't be more than 2× the shorter side**. 1080×2400 (2.22:1) *fails* that cap even though it's a completely normal modern phone resolution — use **1080×1920** (1.78:1) instead, comfortably inside the limit. Minimum 2, maximum 8 per device type.

```js
// scratch script, not a dependency — npx playwright install chromium first
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext({ viewport: { width: 1080, height: 1920 }, deviceScaleFactor: 1 });
  // Suppress any "install to home screen" PWA prompt before it can cover a shot.
  await context.addInitScript(() => localStorage.setItem('installPromptDismissed', 'true'));
  const page = await context.newPage();
  await page.goto('http://localhost:3000/some-screen', { waitUntil: 'networkidle' });
  await page.screenshot({ path: 'store-assets/screenshots/phone/01-screen-name.png' });
  await browser.close();
})();
```

If the app has a first-open intro/reveal animation, don't just wait it out with a fixed timeout — that both wastes time and risks screenshotting a half-finished transition frame. Find its skip control (or drive the same click a real user would to reach the target screen) instead.

Tablet sets (7-inch and 10-inch — e.g. 1200×1920 and 1600×2560) are only required if you want Play to present the app as tablet-optimized; shoot them from the same script with bigger viewports.

**Native app:** boot an emulator and use adb — no Playwright needed:

```
adb exec-out screencap -p > store-assets/screenshots/phone/01-screen-name.png
```

Shoot a deliberate sequence (5–8 screens telling the app's story in gallery order), not every screen mechanically — the first two or three are what most people actually look at on the listing page. Convert any RGBA output to 24-bit before upload (`magick in.png -alpha off out.png`) — Play refuses screenshots with an alpha channel.

## Listing copy template

Google's hard limits — count characters, don't estimate:

| Field | Limit | Notes |
|---|---|---|
| App name | 30 chars | Shown everywhere; keyword-stuffing it is a policy violation |
| Short description | 80 chars | The line under the name; expandable on tap |
| Full description | 4000 chars | Lead with what the app does in one sentence |
| Release notes ("What's new") | 500 chars | Per release, shown on the listing |

```markdown
## App name (30 char max)
<name — count it>

## Short description (80 char max)
<one line — count it>

## Full description (4000 char max)
<full description>

## Release notes (this version, 500 char max)
<what's new>
```

No keyword field exists on Play — search indexes the name, short description, and full description themselves, which is why the copy matters more here than on iOS.

## Privacy policy template

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Privacy Policy — <!-- REPLACE: app name --></title>
</head>
<body>
<h1>Privacy Policy</h1>
<p><!-- REPLACE: app name --> · Last updated: <!-- REPLACE: publish date --></p>

<h2>What this app does with your data</h2>
<p><!-- REPLACE: one paragraph, accurate to what docs/STORE-FACTS.md found by grepping the actual code — network calls, third-party SDKs, what's stored and where. --></p>

<h2>What's stored, and where</h2>
<ul>
<!-- REPLACE: one <li> per thing actually persisted, and whether it's local-only or synced to a server -->
</ul>

<h2>Third parties</h2>
<p><!-- REPLACE: name every third-party SDK/service that receives any data, or state plainly that there are none. --></p>

<h2>Account and data deletion</h2>
<p><!-- REPLACE if the app has accounts: describe the in-app deletion path AND link the web deletion page (both are required by Play's User Data policy). Otherwise a one-line "no accounts, nothing to delete server-side" is enough. --></p>

<h2>Children's privacy</h2>
<p><!-- REPLACE if the app is directed at children; otherwise a one-line "doesn't knowingly collect from anyone" is enough for apps with no accounts. --></p>

<div class="contact">
  Questions? Contact: <!-- REPLACE: a real, monitored address -->
</div>
</body>
</html>
```

Keep it a real, hostable static page — a GitHub Pages page off the app's own repo is the fastest zero-cost place to put it. The same URL goes in Play Console's App content page **and** must be reachable from inside the app.

Draft it once as `store-assets/privacy-policy.html`. If the app also ships elsewhere, reuse the same link rather than writing a second document — two policies for one app eventually say two different things, and the one that contradicts a questionnaire answer is the one a reviewer reads.

## Build gates

Run before ticking Phase 00's build-gate item. Never tick it without actually running these:

| Stack | Commands |
|---|---|
| Capacitor / web-shell | `npm run build` (or equivalent) → grep the built output for known dev-only markers → `npx cap sync android` → `cd android && ./gradlew assembleDebug` |
| Expo | `npx expo prebuild -p android` (if config changed) → `cd android && ./gradlew assembleDebug`, or `eas build --platform android --profile preview` |
| React Native | `cd android && ./gradlew assembleDebug` |
| Flutter | `flutter build apk --debug` (and `flutter build appbundle` for the release artifact) |
| Native Gradle | `./gradlew assembleDebug` |

Check for the literal `BUILD SUCCESSFUL` line. The release artifact for upload is always the AAB: `./gradlew bundleRelease` → `android/app/build/outputs/bundle/release/app-release.aab` (needs the signing config from above; unsigned bundles upload nowhere).

Also run the project's actual test and lint commands (`npm test`, `npm run lint`, or their stack equivalents) — a passing Gradle build with a broken test suite isn't a passing gate.
