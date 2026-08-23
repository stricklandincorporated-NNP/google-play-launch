# google-play-launch

A Claude Code plugin that gets an Android app onto Google Play.

It runs recon on your repo, automates every step that can be automated (icon, splash screen, feature graphic, `AndroidManifest.xml`, signing scaffold, screenshots, listing copy, privacy policy), and generates **`store-assets/MANIFEST.html`** — a beginner-readable walkthrough written for someone who has never shipped an app.

Every step says what to click, in what order, and *why it matters*. Every Google link in it was HTTP-checked.

## Install

These are Claude Code slash commands — run them *inside* a Claude Code session, not at your regular terminal prompt. Start Claude Code first:

```
claude
```

Then, one at a time, at the Claude Code prompt:

```
/plugin marketplace add stricklandincorporated-NNP/google-play-launch
```

Wait for that to finish, then:

```
/plugin install google-play-launch
```

Restart Claude Code, then run:

```
/google-play-launch
```

## What you get

- **Phase 00** — everything an agent can do unattended, before you even have a Play Console account.
- **PREP + Phases 01–13** — the human path: Play Console registration, agreements and tax, Android Studio setup, app identity and signing, build and upload, closed testing, listing, privacy and compliance, pricing, review, release, monetization.
- **Three appendices** — a plain-English glossary, a troubleshooting table keyed to Google's actual error text and policy names, and every link in one place.

Progress is saved in the browser. An "Expand every why" button turns the checklist into a readable guide.

## The one thing to know before you start

A **personal** Play Console account created after November 13, 2023 can't publish to production at all until it runs a closed test with 12 testers opted in for 14 continuous days. That clock is the longest single item in the whole process — recruit your testers on day one, not after everything else is ready. Organization accounts are exempt.

## Requires

Any computer — Windows, Mac or Linux all work for the steps that build and upload.

MIT.
