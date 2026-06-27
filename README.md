<p align="center">
  <img src="registry-entry/mimo-wordmark.svg" alt="MiMo Code" height="32">
</p>

<p align="center">
  <strong>Terminal-native AI coding assistant with persistent memory</strong>
</p>

<p align="center">
  <a href="https://agentclientprotocol.com/">ACP</a> •
  <a href="https://github.com/XiaomiMiMo/MiMo-Code">GitHub</a> •
  <a href="https://mimo.xiaomi.com/mimocode">Website</a>
</p>

---

## What is MiMo Code?

MiMo Code is a terminal-native AI coding assistant that speaks the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) natively. It works with Zed, JetBrains, and any ACP-compatible editor.

### Features

- **Multiple agents** — Build, plan, and compose with specialized agents
- **Persistent memory** — Context carries across sessions
- **Subagent orchestration** — Break complex tasks into parallel work
- **MCP server support** — Connect to external tools and data sources
- **Voice input** — Speak your prompts naturally

## Quick Start

### Install

```bash
curl -fsSL https://mimo.xiaomi.com/install | bash
```

or

```bash
npm install -g @mimo-ai/cli
```

### Connect to Zed

Add to your Zed `settings.json`:

```json
{
  "agent_servers": {
    "MiMo Code": {
      "command": "mimo",
      "args": ["acp"]
    }
  }
}
```

### Verify it works

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":1}}' | mimo acp
```

## How it works

MiMo Code implements ACP using the official `@agentclientprotocol/sdk`. When connected to an editor, it exposes:

- **Model selection** — Choose your LLM backend
- **Reasoning effort** — Control depth: low (fast), medium (balanced), high (deep)
- **Agent modes** — Switch between build, plan, and compose workflows

## Registry

This repository contains the [ACP Registry](https://github.com/agentclientprotocol/registry) entry for MiMo Code. The `registry-entry/` directory holds the submission files.

**Current PR:** [agentclientprotocol/registry#409](https://github.com/agentclientprotocol/registry/pull/409)

## Contributing

See the [MiMo-Code repository](https://github.com/XiaomiMiMo/MiMo-Code) for the ACP implementation and contribution guidelines.

## License

MIT
