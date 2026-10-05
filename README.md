<!-- markdownlint-disable MD033 MD041 -->
<p align="center">
  <a href="https://www.pushengage.com/">
    <img src=".github/assets/pushengage-logo.svg" alt="PushEngage" width="280">
  </a>
</p>
<!-- markdownlint-enable MD033 MD041 -->

# PushEngage Agent Skills

Helps an AI coding agent integrate and debug the [PushEngage](https://www.pushengage.com/) mobile SDK — push notifications **and in-app messaging** — across **iOS, Android, Flutter, and React Native**.

The six skills follow the open [Agent Skills (`SKILL.md`) standard](https://agentskills.io/specification), so they work with **any agent that supports it** — Claude Code, Cowork, and other SKILL.md-compatible agents — while also shipping as an installable Claude Code plugin.

## Requirements

- A [PushEngage](https://www.pushengage.com/) account with an app configured in the dashboard.
- An AI coding agent that supports Agent Skills (Claude Code recommended).

## Install

### Claude Code / Cowork (plugin)

```text
/plugin marketplace add awesomemotive/pushengage-skills
/plugin install pushengage
```

Installing the plugin also configures the official [PushEngage MCP server](https://github.com/awesomemotive/pushengage-mcp) (`@pushengage/mcp` via `npx`, Node.js 18+), which lets the agent operate your PushEngage account — send and schedule notifications, build segments, read analytics, manage site settings. On first use it opens a browser window to log in to your PushEngage account; until then the account tools simply stay idle.

### Other agents via the skills CLI ([skills.sh](https://skills.sh))

Works with Cursor, Codex, Copilot, Windsurf, and other SKILL.md-compatible agents:

```bash
npx skills add awesomemotive/pushengage-skills
```

### Manual install

The skills live under `skills/`. Copy the skill folders into your agent's skills directory, or point your agent at this repository. Each skill is a `SKILL.md` directory and activates automatically based on the project type it detects.

## Usage

After installing, just talk to Claude:

- "Help me integrate PushEngage into this app." — runs the full integration flow.
- "PushEngage notifications aren't arriving on my iPhone." — runs the diagnostic flow.
- "Audit my PushEngage setup." — runs the static audit.
- "Add PushEngage in-app messages to this app" / "My in-app message never shows." — walks the in-app messaging setup or its troubleshooting checklist (SDK iOS 1.1.0+, Android 1.0.0+, Flutter 1.1.0+, React Native 1.1.0+).

The plugin detects your platform automatically and hands off to the matching integration/debug skill.

With the MCP server connected, you can also operate your account: "Send a notification titled 'Sale ends tonight' to my US segment", "How did last week's campaign perform?"

## What's inside

This marketplace ships one plugin, `pushengage`, containing six skills:

| Skill | Purpose |
|---|---|
| `pushengage` | Entry skill — detects platform and routes to the right spoke; holds the cross-platform concepts, best practices, and in-app messaging semantics. |
| `pushengage-ios` | iOS integration and debugging. |
| `pushengage-android` | Android integration and debugging. |
| `pushengage-flutter` | Flutter integration and debugging. |
| `pushengage-react-native` | React Native integration and debugging. |
| `pushengage-debug` | Shared diagnostic skill for broken integrations. |

## License

[MIT](LICENSE) © 2026 PushEngage

## Feedback

Questions or feedback? [Open an issue](https://github.com/awesomemotive/pushengage-skills/issues) or reach the PushEngage team via [pushengage.com](https://www.pushengage.com/).
