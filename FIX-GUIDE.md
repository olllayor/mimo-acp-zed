# Fix: Separate Model and Effort in MiMo Code ACP

## Problem

Zed shows combined dropdown entries like `MiMo Auto (free)/MiMo Auto (high)` instead of separate model and effort selectors. This is because effort levels (low/medium/high) are treated as model variants in the ACP config options.

## File to Modify

`packages/opencode/src/acp/agent.ts`

## Step 1: Filter Effort Variants from Model List

Find the `buildAvailableModels` function (around line 1674). Change the variant filtering logic to exclude effort levels:

```typescript
// BEFORE (line ~1688):
const variants = Object.keys(model.variants).filter((variant) => variant !== DEFAULT_VARIANT_VALUE)

// AFTER:
const EFFORT_VARIANTS = new Set(["low", "medium", "high"])
const variants = Object.keys(model.variants).filter(
  (variant) => variant !== DEFAULT_VARIANT_VALUE && !EFFORT_VARIANTS.has(variant)
)
```

## Step 2: Update `buildConfigOptions` Signature

Find the `buildConfigOptions` function (around line 1755). Add `currentEffort` parameter:

```typescript
// BEFORE:
function buildConfigOptions(input: {
  currentModelId: string
  availableModels: ModelOption[]
  modes?: { availableModes: ModeOption[]; currentModeId: string } | undefined
}): SessionConfigOption[] {

// AFTER:
function buildConfigOptions(input: {
  currentModelId: string
  availableModels: ModelOption[]
  currentEffort?: string
  modes?: { availableModes: ModeOption[]; currentModeId: string } | undefined
}): SessionConfigOption[] {
```

## Step 3: Add `thought_level` Config Option

Inside `buildConfigOptions`, add the effort selector after the model option:

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
  if (input.modes) {
    options.push({
      id: "mode",
      name: "Session Mode",
      category: "mode",
      type: "select",
      currentValue: input.modes.currentModeId,
      options: input.modes.availableModes.map((m) => ({
        value: m.id,
        name: m.name,
        ...(m.description ? { description: m.description } : {}),
      })),
    })
  }
  return options
}
```

## Step 4: Pass `currentEffort` to `buildConfigOptions`

Update all call sites of `buildConfigOptions` to pass the current effort value.

### In `loadSessionMode` (around line 1202):

```typescript
// BEFORE:
configOptions: buildConfigOptions({
  currentModelId: formatModelIdWithVariant(model, currentVariant, availableVariants, true),
  availableModels,
  modes,
}),

// AFTER:
configOptions: buildConfigOptions({
  currentModelId: formatModelIdWithVariant(model, currentVariant, availableVariants, true),
  availableModels,
  currentEffort: this.sessionManager.getVariant(sessionId),
  modes,
}),
```

### In `setSessionConfigOption` (around line 1280):

```typescript
// BEFORE:
return {
  configOptions: buildConfigOptions({ currentModelId, availableModels, modes }),
}

// AFTER:
const updatedSession = this.sessionManager.get(session.id)
return {
  configOptions: buildConfigOptions({
    currentModelId,
    availableModels,
    currentEffort: updatedSession.variant,
    modes,
  }),
}
```

### In `loadSession` (around line 626):

```typescript
// BEFORE:
result.configOptions = buildConfigOptions({
  currentModelId: result.models.currentModelId,
  availableModels: result.models.availableModels,
  modes: result.modes,
})

// AFTER:
result.configOptions = buildConfigOptions({
  currentModelId: result.models.currentModelId,
  availableModels: result.models.availableModels,
  currentEffort: this.sessionManager.getVariant(sessionId),
  modes: result.modes,
})
```

## Step 5: Handle `setSessionConfigOption` for Effort

In the `setSessionConfigOption` method (around line 1250), add handling for the `effort` config ID:

```typescript
// BEFORE:
if (params.configId === "model") {
  // ... model handling
} else if (params.configId === "mode") {
  // ... mode handling
} else {
  throw RequestError.invalidParams(JSON.stringify({ error: `Unknown config option: ${params.configId}` }))
}

// AFTER:
if (params.configId === "model") {
  // ... model handling (unchanged)
} else if (params.configId === "mode") {
  // ... mode handling (unchanged)
} else if (params.configId === "effort") {
  if (typeof params.value !== "string") throw RequestError.invalidParams("effort value must be a string")
  const validEfforts = ["low", "medium", "high"]
  if (!validEfforts.includes(params.value)) {
    throw RequestError.invalidParams(JSON.stringify({ error: `Invalid effort value: ${params.value}` }))
  }
  this.sessionManager.setVariant(session.id, params.value)
} else {
  throw RequestError.invalidParams(JSON.stringify({ error: `Unknown config option: ${params.configId}` }))
}
```

## Step 6: Update `unstable_setSessionModel`

In `unstable_setSessionModel` (around line 1220), ensure the effort is preserved when changing models:

```typescript
// The existing code already handles this via setVariant, but verify:
async unstable_setSessionModel(params: SetSessionModelRequest) {
  const session = this.sessionManager.get(params.sessionId)
  // ... existing provider/model parsing ...

  // When switching models, preserve the current effort level
  // (already handled by setVariant in parseModelSelection)

  return {
    _meta: buildVariantMeta({
      model: selection.model,
      variant: selection.variant,
      availableVariants,
    }),
  }
}
```

## Expected Result

After these changes, Zed will show:

1. **Model dropdown** (`category: "model"`): Clean list of models without effort suffixes
   - MiMo Auto
   - MiMo Auto (free)
   - Nvidia Nemotron Safety

2. **Effort dropdown** (`category: "thought_level"`): Separate reasoning level selector
   - Low
   - Medium
   - High

Each is independent — changing the model doesn't reset the effort level, and vice versa.

## Testing

1. Build MiMo Code: `npm run build`
2. Run ACP manually:
   ```bash
   echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":1}}' | ./dist/mimocode acp
   ```
3. Create a session:
   ```bash
   echo '{"jsonrpc":"2.0","id":2,"method":"session/new","params":{"cwd":"/tmp","mcpServers":[]}}' | ./dist/mimocode acp
   ```
4. Verify the response has separate `model` and `effort` configOptions
5. Open Zed, select MiMo Code as agent, confirm two separate dropdowns appear
