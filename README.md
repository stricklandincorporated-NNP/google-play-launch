# google-play-launch

A Claude Code plugin that gets an Android app onto Google Play.

It runs recon on your repo, automates every step that can be automated (icon, splash screen, feature graphic, `AndroidManifest.xml`, signing scaffold, screenshots, listing copy, privacy policy), and generates **`store-assets/MANIFEST.html`** — a beginner-readable walkthrough written for someone who has never shipped an app.

Every step says what to click, in what order, and *why it matters*. Every Google link in it was HTTP-checked.

## Install

One line, in any terminal — Mac, Linux, or Windows via Git Bash/WSL. VS Code's built-in terminal is fine too:

```
git clone https://github.com/stricklandincorporated-NNP/google-play-launch.git /tmp/gpl && mkdir -p ~/.claude/skills && cp -r /tmp/gpl/skills/google-play-launch ~/.claude/skills/ && rm -rf /tmp/gpl && echo "Installed. Restart Claude Code, then run /google-play-launch"
```

Restart Claude Code, then from any app project run:

```
/google-play-launch
```

That's it. The skill lands in `~/.claude/skills/`, so it works in every project and in the VS Code extension.

<details>
<summary>Alternative: install as a plugin instead</summary>

Same result, more steps. `/plugin` is a Claude Code slash command that only works in the **terminal CLI** — in the VS Code extension it reports "/plugin isn't available in this environment."

Open a standalone terminal (macOS Terminal, iTerm, Windows Terminal — not VS Code's integrated one), start Claude Code with `claude`, then at its prompt:

```
/plugin marketplace add stricklandincorporated-NNP/google-play-launch
```

Wait for it to finish, then:

```
/plugin install google-play-launch
```

Restart Claude Code and run `/google-play-launch`.
</details>

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
