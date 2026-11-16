# phone-use skills

Agent skills for [phone-use](https://phoneuse.dev): give your agent a phone. It drives the iOS Simulator
or an Android device or emulator on your own machine, free and with no account.

## Install

```bash
npx skills add Derivate-Inc/phone-use-skills
```

This teaches Claude Code, Codex, Cursor or any skills-compatible agent the phone-use CLI: picking a
phone, driving apps by label, installing your build, running several phones in parallel, and checking
results honestly.

Then install the CLI:

```bash
curl -fsSL https://phoneuse.dev/install | bash
```

On a Mac with Xcode, `open -a Simulator` and your agent is ready. For Android, run
`phone-use connect android` with a device or emulator on adb.

Cloud phones, which run on our hardware, are on a [waitlist](https://phoneuse.dev/waitlist). The skill
tells the agent to stay on the local phone and what to say when `phone-use create` answers with the
waitlist.

Docs: https://phoneuse.dev/docs
