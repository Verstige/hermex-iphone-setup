# Hermex iPhone Setup: Connect the App to Your Dashboard

> Connect the Hermex iPhone app to a local Hermes dashboard over the network, with the pairing and reachability checks that stop a half-installed setup.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg)

## What it does

Connect the Hermex iPhone app to a local Hermes dashboard.

Hermex (native iOS client, `hermexapp.com`) connects **straight to the Hermes
dashboard** on port 9119 — NOT the gateway's api_server (8642) and NOT the
gateway Webapp listener (the ephemeral 55492). It authenticates with
`dashboard.basic_auth` (username + password) and upgrades its live-chat socket
via `POST /auth/password-login` → session cookies → `POST /api/auth/ws-ticket`
→ `/api/ws?ticket=`.

## The failure it prevents

Working this by hand means rediscovering the same edge cases every time. They are written down here instead, with the fix for each one.

## When to use it

Reach for this skill when the task matches `hermex-iphone-setup`.

## What's inside

```
plugins/
└── hermex-iphone-setup/
    ├── .claude-plugin/plugin.json
    └── skills/hermex-iphone-setup/
        ├── SKILL.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/hermex-iphone-setup
claude plugin install hermex-iphone-setup@hermex-iphone-setup
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/hermex-iphone-setup.git
cp -R hermex-iphone-setup/plugins/hermex-iphone-setup/skills/hermex-iphone-setup ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "Use the hermex-iphone-setup skill for this task."
```

## License

MIT. See [LICENSE](LICENSE).

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
