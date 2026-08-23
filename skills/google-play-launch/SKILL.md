---
name: google-play-launch
description: Use when the user wants to prepare or ship an Android app for Google Play — "get my app on Google Play", "ship to Google Play", "Play Store submission", "Android release prep", "closed testing". Runs recon on the repo, writes a plan, implements every automatable step (icon/splash/feature graphic, AndroidManifest, signing scaffold, screenshots, listing copy, privacy policy), and tracks progress on MANIFEST.html. Leaves only the steps that genuinely require a human with a Play Console account.
---

# Google Play launch prep

This folder is portable — copied wholesale into any project's `.claude/skills/`, it works there too. Nothing here is specific to the app it first shipped from.

Detailed commands, file templates, and exact Gradle/XML live in [reference/android-recipes.md](reference/android-recipes.md) — load that file when you reach the step that needs it, not up front. This file is the procedure; that file is the cookbook.

## Step 1 — Recon (read-only)

Don't write anything yet. Detect how this app is built. First match wins:

1. `android/` with Gradle files already present → work on it directly, whatever built it.
2. `@capacitor/core` in `package.json` → `npx cap add android` if `android/` is missing.
3. `expo` in `package.json` → `npx expo prebuild -p android`.
4. `react-native` in `package.json` → `android/` should already exist; if it doesn't, report that rather than scaffolding blind — a missing `android/` on an RN project usually means something else is wrong.
5. `pubspec.yaml` exists → `flutter create --platforms=android .`
6. Any web app with a build output directory — a `build` script in `package.json`, *or* a bundler config file (`vite.config.*`, `webpack.config.*`, etc.); either one alone is enough to match this rung — → offer Capacitor: `@capacitor/core @capacitor/cli @capacitor/android`, `npx cap init`, `npx cap add android`.
7. None of the above → stop and report what you found. Don't guess at a framework.

While you're in here, also note: the app's display name, whatever package name (`applicationId`) is already set, even if it's a placeholder — flag `com.example.*` hard, Play refuses it outright — the current `versionCode`/`versionName`, whether `docs/` or `store-assets/` (or equivalent) already exist, and whether a `MANIFEST.html` is already present from a prior run.

Also grep for the things that turn manifest Phase 08 from boilerplate into real work, because each one is a named rejection or removal reason and the user almost certainly doesn't know it applies to them:

| Grep for | If found | Manifest item |
|---|---|---|
| account creation / signup / `createUser` | in-app account deletion is required — Google's User Data policy additionally wants a **web URL** where deletion can be requested without installing the app | Phase 08 |
| `fetch(`, `XMLHttpRequest`, `WebSocket`, analytics/ad SDK imports, storage key namespaces | raw material for the Data safety form — write it to `docs/STORE-FACTS.md`, don't answer the form from memory | Phase 08 |
| ad SDK imports (AdMob / `com.google.android.gms.ads`, AppLovin, ironSource, Unity Ads…) | the ads declaration, content rating and Data safety form must all agree the app shows ads — a mismatch is a policy strike | Phase 08 |
| restricted permissions in `AndroidManifest.xml` (`READ_SMS`, `READ_CALL_LOG`, `MANAGE_EXTERNAL_STORAGE`, `QUERY_ALL_PACKAGES`, `ACCESS_BACKGROUND_LOCATION`, Accessibility services) | each needs a permission declaration form and a core-functionality justification, or the release is rejected | Phase 08 |
| child/family-directed wording, or a target audience including under-13s | the Families policy applies — stricter ads, data and content rules than the general store | Phase 08 |

One flag that's account-level, not grep-level: a **personal Play Console account created after November 13, 2023** must run a 12-tester / 14-day closed test before it can apply for production access. Organization accounts are exempt. Report it in the plan and the handoff — it is the single biggest calendar-time item in the whole process, so it should start on day one, not after everything else is ready.

See [reference/android-recipes.md § Scaffold commands](reference/android-recipes.md#scaffold-commands) for the expanded version of the ladder above, including flags and what each command produces.

## Step 2 — Plan

Write `GOOGLE-PLAY-PLAN.md` at the repo root: what was detected in Step 1, what you're about to change, and what you can't do (anything needing a Play Console account, a real package name, keystore passwords, or a human decision — see Step 3's "ask, never invent" list).

Copy `MANIFEST.html` from this skill folder to `store-assets/MANIFEST.html` (create `store-assets/` if it doesn't exist). Leave the "AGENT: replace this block" status callout as-is for now — you'll fill it in during Step 4. Tell the user this file exists and that opening it in a browser gives them a live checklist.

**What the manifest is, so you don't degrade it.** It's written for someone with no technical background who has never shipped an app: Phase 00 (agent) → PREP → Phases 01–13 → three appendices (glossary, failure modes, every link). Every one of its ~98 steps carries three things, and all three are load-bearing:

1. a one-line **what** (`.step-detail`),
2. a literal **where to click** (`.path`) — `Play Console → your app → App content`, or a shell command,
3. a collapsed **why** (`<details class="why">`) explaining the consequence of getting it wrong.

If you add a step, add all three. If you edit a step, don't drop the `why` to save space — the "why" is the whole reason the file is long, and a beginner following a checklist they don't understand is exactly the failure this replaces.

**Links.** Every URL in the file was HTTP-checked before it shipped. If you add one, check it the same way (`curl -s -o /dev/null -w '%{http_code}' -L <url>`) and add it to Appendix C as well as inline. Never invent a Google URL from memory — the help-center answer IDs return real 404s when wrong, and a dead link in a beginner's checklist is worse than no link.

## Step 3 — Implement

Work through Phase 00 of the manifest, ticking each item as it actually lands — not before. Marking a step done means editing that step's checkbox in `store-assets/MANIFEST.html` in place:

```html
<!-- before -->
<li>
  <label class="step-row"><input type="checkbox" data-key="gp-a-icon"><span class="step-text"><span class="step-title">App icon generated …</span></span></label>
  <div class="step-body">
    <span class="step-detail">Rasterized from a source SVG/PNG …</span>
    <details class="why"><summary>Why</summary><p>…</p></details>
  </div>
</li>

<!-- after: three edits — the input, the title, and one new line in step-body -->
<li>
  <label class="step-row"><input type="checkbox" data-key="gp-a-icon" checked disabled><span class="step-text"><span class="step-title">App icon generated …<span class="tag auto">agent</span></span></span></label>
  <div class="step-body">
    <span class="step-detail">Rasterized from a source SVG/PNG …</span>
    <span class="done-note">Verified 512×512, adaptive layers wired into <span class="inline-code">res/mipmap-*</span>.</span>
    <details class="why"><summary>Why</summary><p>…</p></details>
  </div>
</li>
```

The done-note goes **inside `.step-body`, after `.step-detail`** — not inside the `<label>`. Detail text, links, and the `<details>` deliberately live outside the label, because inside it a click on a link or a disclosure triangle also toggles the checkbox.

Write a real path, a real command, a real number in the note — never "done".

Four rules carried over from what actually worked building this skill in the first place:

- **Never fake a verification.** Run `cd android && ./gradlew assembleDebug` after every native-side edit (`AndroidManifest.xml`, `build.gradle`, resources) and check it actually says `BUILD SUCCESSFUL`. If a step can't be run in this environment (no JDK, no Android SDK), say that plainly in the plan and in the manifest note — don't tick the box and don't claim it passed.
- **Grep, don't assume.** Privacy facts (what the app sends over the network, what it stores, what SDKs it links) come from grepping the actual source — `fetch(`, `XMLHttpRequest`, `WebSocket`, known analytics/ad SDK imports, storage key namespaces — plus the *merged* `AndroidManifest.xml` permission list, never from what the app is "supposed to" do. Write the findings to `docs/STORE-FACTS.md` so Phase 08's Data safety form is answered from one place.
- **Prove the bundle, not the config.** A build flag that's supposed to strip dev tooling can silently stop working. Audit the built output itself (grep the compiled bundle for dev-only markers, and confirm the release build isn't `debuggable`), not the config file that claims to exclude it.
- **Ask, never invent:** package name (`applicationId`), developer public name (shown on every listing), support email, Play category, countries, and **free-vs-paid**, which Play locks permanently the moment you publish. These need a real decision from a human with a Play Console account — a placeholder here becomes a real, hard-to-change identifier once shipped. And **never generate the upload keystore with a password the user didn't choose** — scaffold the `keytool` command and the Gradle `key.properties` wiring, but the keystore itself is theirs to create and back up; losing it means losing the ability to update the app. Leave the corresponding manifest items unchecked and list them explicitly in the handoff.

Icon/splash/feature-graphic generation, `AndroidManifest.xml` edits, the keystore + `key.properties` signing scaffold, screenshot automation, and the listing/privacy-policy templates are all in [reference/android-recipes.md](reference/android-recipes.md) — load it now if you haven't.

## Step 4 — Handoff

Update the manifest's status callout (the block you left alone in Step 2) with this repo's real state — what's done, with a one-line detail each, and what's still open. Report to the user:

- What's done, and what's left, phase by phase.
- A pointer to `store-assets/MANIFEST.html` — tell them to open it in a browser, and mention the **Expand every "why"** button, since that turns the checklist into a readable guide for a first-timer.
- Every value you couldn't fill in yourself (the "ask, never invent" list from Step 3), stated as questions, not left implicit.
- Anything Step 1's grep turned up (account deletion, ads declarations, restricted permissions, Families policy). These are work the user has to schedule, not boxes to tick, and they're the rejections that arrive after everything else looked finished.
- If the account is brand new: point at PREP and the timeline table. The **12-testers-for-14-days closed test** (if it applies to this account) blocks production access and cannot be compressed — a user who starts recruiting testers on day one saves more calendar time than every automated step in Phase 00 combined.

## Step 5 — Optional publish

If an Artifact-publishing tool is available in this session, offer to publish `store-assets/MANIFEST.html` for a shareable live URL — handy for sending to the 12 testers you're about to recruit. This is explicitly optional — the file works fine double-clicked open in any browser, with no server and no build step.
