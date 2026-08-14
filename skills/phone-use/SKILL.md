---
name: phone-use
description: Drive cloud iPhones from the phone-use CLI — provision phones in seconds, control real iOS apps, run fleets in parallel, and verify results honestly.
---

# phone-use — give your agent a phone

Cloud iPhones an agent can drive with a CLI. Phones provision in ~2s from a warm
pool, run real iOS apps, and are disposable — create, drive, close.

## Setup

```bash
npm i -g phone-use    # or: bun add -g phone-use
phone-use login       # browser sign-in, saves your API key
```

## The attach pattern (the #1 thing to get right)

Every shell command runs in a fresh shell, so a phone's connection env must be
sourced into EACH command:

```bash
id=$(phone-use create ios --ttl 15m | head -1 | awk '{print $1}')
phone-use env "$id" > /tmp/phone.env
source /tmp/phone.env && phone-use observe
source /tmp/phone.env && phone-use tap "Settings"
```

## Core verbs

- `observe` — frontmost app + visible elements. Pipe through `head`, not `tail`:
  the screen's identity is the `NavigationBar` line at the TOP.
- `screenshot` — the rich channel; use when observe's tree isn't enough.
- `tap "<label>"` — target elements by LABEL text (partial match works). NEVER
  reuse `@e` refs across commands; refs expire every invocation.
- `type`, `scroll`, `drag`, `nav to=back|home`, `open <app>`, `wait_for "<text>"`
- `search <app> <query>` — in-app search. The FIRST arg is the app:
  `phone-use search Wikipedia "Kyoto"`, never `phone-use search "Kyoto"`.
- `batch` — several verbs in one invocation; prefer it for known sequences, it
  is much faster than one CLI call per step.
- `task "<goal>"` — built-in agent runs a whole goal end to end.

## Flag gotchas

Boolean flags are BARE, even where help text shows `<boolean>`:
`--open true` errors — write `--open`. Same for `--submit`. Destructive verbs
like `type` may ask for confirmation; re-run with `--yes`.

## Parallel fleets

Drive N phones from ONE bash call with background subshells — never one at a time:

```bash
for spec in "1:Kyoto" "2:Jupiter" "3:San Francisco"; do
( n="${spec%%:*}"; t="${spec#*:}"
  id=$(phone-use create ios --ttl 15m | head -1 | awk '{print $1}')
  echo "$id" >> /tmp/fleet-ids
  phone-use env "$id" > /tmp/p$n.env
  source /tmp/p$n.env && phone-use search Wikipedia "$t" --open --match "$t"
) &
done
wait
# always clean up:
for id in $(cat /tmp/fleet-ids); do phone-use close "$id" & done; wait
```

## Verify honestly

- A success message is not evidence. Confirm state by reading it back where it
  lives — an element of the thing itself, not a banner saying it worked.
- Webview content (articles, browsers) fills the element tree a beat AFTER the
  action returns. Poll `observe` for a few seconds for an element named for your
  target before declaring success — the first matching substring may be a nav
  icon, not the content.
- A freshly created phone can report running ~30s before it is fully driveable.
  If `observe` says `App: unknown`, wait and retry rather than failing.

## Etiquette

- `close` every phone you create; `--ttl` is the backstop, not the cleanup.
- Don't re-observe after every action — act, then verify once at the end.
- `list` shows what you have running if you lose track.
