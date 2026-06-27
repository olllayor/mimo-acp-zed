# MiMo Code — ACP Registry Entry

This repository contains the [ACP Registry](https://github.com/agentclientprotocol/registry) entry for [MiMo Code](https://github.com/XiaomiMiMo/MiMo-Code).

## What is MiMo Code?

MiMo Code is a terminal-native AI coding assistant built as a fork of [OpenCode](https://github.com/anomalyco/opencode). It speaks the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) natively, so it works with Zed, JetBrains, and any ACP-compatible editor.

### Key Features
- Multiple agents (build, plan, compose)
- Persistent memory across sessions
- Intelligent context management
- Subagent orchestration
- MCP server support
- Voice input

## How to Use

### Install MiMo Code

```bash
curl -fsSL https://mimo.xiaomi.com/install | bash
# or
npm install -g @mimo-ai/cli
```

### Zed Setup

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

### Manual Protocol Test

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":1}}' | mimo acp
```

## Model and Effort Separation

MiMo Code uses ACP [Session Config Options](https://agentclientprotocol.com/protocol/v1/session-config-options) to separate model selection from reasoning effort. These are exposed as **distinct config options** during session setup, not combined into a single dropdown.

### Current Issue

The agent currently treats effort levels (low/medium/high) as **model variants**, embedding them in the model selector as `provider/model/variant` entries. This causes Zed to display combined entries like `MiMo Auto (free)/MiMo Auto (high)` instead of separate selectors.

### Required Fix (in MiMo-Code repo)

The `buildConfigOptions` function in `packages/opencode/src/acp/agent.ts` must be updated to:

1. **Exclude effort variants from model selector** — filter out `low`/`medium`/`high` variants from `buildAvailableModels`
2. **Add separate `thought_level` config option** — expose effort as its own selector

#### Before (broken):
```json
{
  "configOptions": [
    {
      "id": "model",
      "category": "model",
      "options": [
        { "value": "mimo/mimo-auto", "name": "MiMo Auto" },
        { "value": "mimo/mimo-auto/low", "name": "MiMo Auto (low)" },
        { "value": "mimo/mimo-auto/medium", "name": "MiMo Auto (medium)" },
        { "value": "mimo/mimo-auto/high", "name": "MiMo Auto (high)" }
      ]
    }
  ]
}
```

#### After (correct):
```json
{
  "configOptions": [
    {
      "id": "model",
      "name": "Model",
      "category": "model",
      "type": "select",
      "currentValue": "mimo/mimo-auto",
      "options": [
        { "value": "mimo/mimo-auto", "name": "MiMo Auto" },
        { "value": "mimo/mimo-auto-free", "name": "MiMo Auto (free)" },
        { "value": "nvidia/nemotron-safety", "name": "Nvidia Nemotron Safety" }
      ]
    },
    {
      "id": "effort",
      "name": "Reasoning Effort",
      "category": "thought_level",
      "type": "select",
      "currentValue": "high",
      "options": [
        { "value": "low", "name": "Low", "description": "Fast responses with minimal reasoning" },
        { "value": "medium", "name": "Medium", "description": "Balanced speed and reasoning depth" },
        { "value": "high", "name": "High", "description": "Deep reasoning for complex tasks" }
      ]
    }
  ]
}
```

### Code Changes Required

In `packages/opencode/src/acp/agent.ts`:

1. **Filter effort variants from model list** in `buildAvailableModels`:
```typescript
// Exclude effort variants (low/medium/high) from model selector
const EFFORT_VARIANTS = new Set(["low", "medium", "high"])
const variants = Object.keys(model.variants).filter(
  (v) => v !== DEFAULT_VARIANT_VALUE && !EFFORT_VARIANTS.has(v)
)
```

2. **Add `thought_level` config option** in `buildConfigOptions`:
```typescript
function buildConfigOptions(input: {
  currentModelId: string
  availableModels: ModelOption[]
  currentEffort?: string
  modes?: { availableModes: ModeOption[]; currentModeId: string } | undefined
}): SessionConfigOption[] {
  const options: SessionConfigOption[] = [
    {
      id: "model",
      name: "Model",
      category: "model",
      type: "select",
      currentValue: input.currentModelId,
      options: input.availableModels.map((m) => ({ value: m.modelId, name: m.name })),
    },
    {
      id: "effort",
      name: "Reasoning Effort",
      category: "thought_level",
      type: "select",
      currentValue: input.currentEffort ?? "high",
      options: [
        { value: "low", name: "Low", description: "Fast responses with minimal reasoning" },
        { value: "medium", name: "Medium", description: "Balanced speed and reasoning depth" },
        { value: "high", name: "High", description: "Deep reasoning for complex tasks" },
      ],
    },
  ]
  // ... modes handling
  return options
}
```

3. **Handle `setSessionConfigOption` for effort**:
```typescript
} else if (params.configId === "effort") {
  if (typeof params.value !== "string") throw RequestError.invalidParams("effort value must be a string")
  this.sessionManager.setVariant(session.id, params.value)
}
```

### How Clients Should Render This

- **Model selector** (`category: "model"`): Primary dropdown for choosing the LLM backend
- **Effort selector** (`category: "thought_level"`): Secondary control for reasoning depth, rendered near the model picker or as a separate toggle

Clients **SHOULD NOT** combine these into a single combined entry like `MiMo Auto (free)/MiMo Auto (high)`. Each option is independent — the user picks a model and an effort level separately.

## Registry Entry

The `agent.json` and `icon.svg` files in this repository are the ACP registry submission. To register:

1. Fork [agentclientprotocol/registry](https://github.com/agentclientprotocol/registry)
2. Copy `agent.json` → `mimocode/agent.json`
3. Copy `icon.svg` → `mimocode/icon.svg`
4. Submit a PR

## ACP Implementation

MiMo Code's ACP implementation lives in [`packages/opencode/src/acp/`](https://github.com/XiaomiMiMo/MiMo-Code/tree/main/packages/opencode/src/acp) in the MiMo-Code repository. It uses the official `@agentclientprotocol/sdk` TypeScript SDK.

## License

MIT
