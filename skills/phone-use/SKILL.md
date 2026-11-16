---
name: phone-use
description: Drive a phone from the phone-use CLI — the iOS Simulator or an Android device or emulator on this machine (free, no account), the user's real iPhone, or a cloud phone on an approved account. Tap apps by label, install your own build, run several phones in parallel, and verify results honestly.
---

# phone-use — give your agent a phone

One CLI, the same verbs on every phone:

- **the iOS Simulator on this Mac** (needs Xcode). Free, no account.
- **an Android device or emulator over adb** (macOS, Linux, Windows). Free, no account.
- **the user's real iPhone**, through iPhone Mirroring on a Mac.
- **a cloud phone**, on our hardware. Waitlist: only approved accounts get one.

Start on this machine. Do not run `phone-use create` unless the user asks for a
cloud phone.

## Setup

```bash
curl -fsSL https://phoneuse.dev/install | bash    # or: npm i -g phone-use (needs Bun)
phone-use doctor      # checks Bun, Xcode, simulators
```

## Pick a phone

```bash
# iOS Simulator (the default: with nothing else chosen, verbs drive the one booted simulator)
open -a Simulator && phone-use observe

# Android device or emulator on adb
phone-use connect android && phone-use observe

# The user's real iPhone (macOS 15+, iPhone Mirroring)
phone-use doctor --mirror    # lists every missing permission and how to grant it
phone-use connect && phone-use observe
```

- If `observe` says no simulator is booted, the Simulator is still starting. Run
  it again in a few seconds.
- `connect` is remembered: every later command drives that phone until
  `phone-use disconnect`. For one command only, set `PHONE_USE_DEVICE=android`
  or `PHONE_USE_DEVICE=mirror` instead.
- On a real phone you are acting on the user's own apps and accounts. Ask before
  anything that sends, buys, deletes or changes a setting. If `observe` shows
  "Unlock iPhone", ask the user to unlock it; do not guess.

## Cloud phones are on a waitlist

`phone-use create ios` gives a cloud phone only to an approved account. For
everyone else it creates nothing, prints a waitlist message and exits 1 (`--json`
prints `{"code":"cloud_waitlist", ...}`). When you see that:

1. Do not retry and do not sign up for another account.
2. Carry on with the local simulator, using the command the message prints.
3. Tell the user they can join the waitlist at https://phoneuse.dev/waitlist.

On an approved account, every shell command runs in a fresh shell, so the
phone's connection env must be sourced into EACH command:

```bash
phone-use login
id=$(phone-use create ios --ttl 15m | head -1 | awk '{print $1}')   # or: create android
phone-use env "$id" > /tmp/phone.env
source /tmp/phone.env && phone-use observe
source /tmp/phone.env && phone-use tap "Settings"
phone-use close "$id"
```

`close` every cloud phone you create; `--ttl` is the backstop, not the cleanup.
`phone-use list` shows what is running. A fresh cloud phone can report running
about 30s before it is fully driveable: if `observe` says `App: unknown`, wait
and retry.

## Core verbs

- `observe` — frontmost app + visible elements. Pipe through `head`, not `tail`:
  the screen's identity (on iOS, the `NavigationBar` line) is at the TOP.
- `screenshot` — the rich channel; use when observe's tree isn't enough.
- `tap "<label>"` — target elements by LABEL text (partial match works). NEVER
  reuse `@e` refs across commands; refs expire every invocation.
- `type`, `scroll`, `drag`, `nav back|home`, `open <app>`, `wait_for "<text>"`
- `search <app> <query>` — in-app search. The FIRST arg is the app:
  `phone-use search Wikipedia "Kyoto"`, never `phone-use search "Kyoto"`.
- `batch` — several verbs in one invocation; prefer it for known sequences, it
  is much faster than one CLI call per step.
- `task "<goal>"` — built-in agent runs a whole goal end to end (needs
  `ANTHROPIC_API_KEY`).

Your own iOS build (a simulator build, not an App Store app):

```bash
phone-use sim install ./MyApp.app    # local simulator
phone-use install ./MyApp.app        # attached cloud phone
phone-use open <bundle id>
```

## Flag gotchas

Boolean flags are BARE, even where help text shows `<boolean>`:
`--open true` errors — write `--open`. Same for `--submit`. Destructive verbs
like `type` may ask for confirmation; re-run with `--yes`.

## Several phones in parallel

Locally, boot one simulator per lane and pin each command to its simulator with
`PHONE_USE_DEVICE=<udid>`. Run the lanes from ONE bash call with background
subshells, never one at a time:

```bash
xcrun simctl list devices available        # pick or create one device per lane
xcrun simctl boot <udid-a>; xcrun simctl boot <udid-b>
( PHONE_USE_DEVICE=<udid-a> phone-use open Settings && PHONE_USE_DEVICE=<udid-a> phone-use observe ) &
( PHONE_USE_DEVICE=<udid-b> phone-use open Maps     && PHONE_USE_DEVICE=<udid-b> phone-use observe ) &
wait
```

With more than one simulator booted, always set `PHONE_USE_DEVICE`; without it
the CLI expects exactly one. Shut down the simulators you booted when done.

On an approved cloud account the same shape works with one `create` per lane
(write each `phone-use env "$id"` to its own file and source it in that lane),
then `close` every id at the end.

## Verify honestly

- A success message is not evidence. Confirm state by reading it back where it
  lives — an element of the thing itself, not a banner saying it worked.
- Webview content (articles, browsers) fills the element tree a beat AFTER the
  action returns. Poll `observe` for a few seconds for an element named for your
  target before declaring success — the first matching substring may be a nav
  icon, not the content.
- Don't re-observe after every action — act, then verify once at the end.
