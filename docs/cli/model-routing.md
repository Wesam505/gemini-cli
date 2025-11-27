## Model Routing

Gemini CLI includes an intelligent model routing feature that optimizes performance and cost by selecting the most appropriate model based on request context. This is complemented by an automatic fallback mechanism for quota management.

## Routing Architecture

Model routing decisions are made by the `ModelRouterService` in `packages/core/src/routing/modelRouterService.ts`. The service evaluates context and delegates to a pluggable strategy pattern:

- **ClassifierStrategy** (currently active): Uses Gemini API to analyze prompt complexity and recommend flash vs pro model
- **DefaultStrategy**: Simple strategy that always uses the configured model (available as fallback)

## How Routing Works

### 1. Request Evaluation

When a new prompt arrives, the router evaluates:

- **Conversation history:** Recent turns provide context about complexity
- **Turn type:** Determines if routing should be bypassed (tool responses, next speaker requests use current model)
- **Quota status:** If fallback mode is active, routes to flash model automatically
- **Forced override:** User-specified model via CLI flag or settings always takes precedence

### 2. Classification (ClassifierStrategy)

The classifier prompt analyzes the user's request and categorizes it as:

- **`flash` (Complex Reasoning or Debugging):** Multi-file context, architectural design, ambiguous scope
- **`pro` (Simple Tasks):** Basic questions, boilerplate generation, single-tool operations, self-contained work

The classifier uses 4 recent conversation turns for context and examines a 20-turn search window to understand conversation flow.

### 3. Model Resolution

The router maps classifier output to concrete model names:

- `flash` → `gemini-2.5-flash`
- `pro` → `gemini-2.5-pro`

Then applies fallback rules: if quota limit hit, use `gemini-2.5-flash` regardless (except "lite" model requests are always honored).

### 4. Bypass Conditions

Routing is **skipped** for:

- **Tool responses:** Use the model that was initially selected for the prompt
- **Next speaker requests:** Continuation prompts use current model
- **Quota fallback mode:** Hardcoded downgrade to flash model
- **Forced model override:** User-specified model via CLI, env var, or settings takes precedence

## Model Selection Precedence

The model used by Gemini CLI is determined by the following order of precedence:

1. **Forced model override (internal/CLI):** Model specified via `forcedModel` in routing context always takes precedence
2. **`--model` command-line flag:** A model specified with the `--model` flag when launching the CLI
3. **`GEMINI_MODEL` environment variable:** Model specified in the `GEMINI_MODEL` environment variable
4. **`model.name` in `~/.gemini/config.json`:** Model specified in your settings file
5. **Default model:** If none of the above are set, the default is `auto` (resolves to `gemini-2.5-pro`)

### Model Resolution

User-friendly aliases are automatically resolved to concrete model names:

- `pro` or `auto` → `gemini-2.5-pro` (or `gemini-3-pro-preview` if preview features enabled)
- `flash` → `gemini-2.5-flash`
- `flash-lite` → `gemini-2.5-flash-lite`

## Quota Management & Fallback

When API quota limits are hit:

1. **Error Classification:** The router detects quota errors using `classifyGoogleError()`
2. **Fallback Activation:** Quota exhaustion triggers `config.setFallbackMode(true)`
3. **Model Downgrade:** Subsequent requests automatically downgrade to `gemini-2.5-flash`
4. **Lite Preservation:** Requests for "lite" models bypass downgrade (cost optimization exception)
5. **Manual Recovery:** Quota recovery happens on next successful API call; no manual intervention needed

### Error Types

- **TerminalQuotaError:** Hard limit (daily quota) → fallback activated
- **RetryableQuotaError:** Soft limit (per-minute rate) → fallback activated with configurable retry strategy

## Classifier Behavior

The ClassifierStrategy analyzes your prompt to make routing decisions. Understanding when it selects each model helps optimize performance:

### Flash Model Selected When

- **Simple Questions:** Syntax, definitions, API usage (e.g., "explain JavaScript closures")
- **Boilerplate Generation:** Standard code snippets, starter templates
- **Single-Tool Operations:** Direct file reads, simple shell commands, single-file edits
- **Self-Contained Tasks:** All information needed is in your prompt; no discovery required

### Pro Model Selected When

- **Complex Reasoning:** Planning, architecture design, multi-step problems
- **Debugging:** Analyzing errors in large contexts or non-obvious bugs
- **Multi-File Context:** Tasks requiring reading/understanding/modifying multiple files
- **Ambiguous Scope:** Broad requests requiring clarification or creative problem-solving
- **Architectural Design:** System design, refactoring, performance optimization

## Classification Context

The classifier considers:

- **Recent conversation history:** Last 4 turns to understand ongoing work
- **Search window:** 20-turn lookback to detect conversational patterns
- **Prompt content:** Explicit keywords and complexity indicators
- **Context requirements:** Whether multiple files or system components are involved

## Disabling or Overriding Routing

### Use a Specific Model

Override routing completely by specifying a model:

```bash
# Command line
gemini --model pro "your prompt"
gemini --model flash "your prompt"

# Environment variable
export GEMINI_MODEL=pro
gemini "your prompt"

# Settings file (~/.gemini/config.json)
{
  "model": {
    "name": "pro"
  }
}
```

## Performance Implications

| Model | Speed | Cost | Best For |
|-------|-------|------|----------|
| **flash** | Fast (~1s) | Low (1/5 cost of pro) | Quick answers, simple tasks |
| **pro** | Slower (~3-5s) | Higher | Complex reasoning, debugging |
| **flash-lite** | Fastest (<1s) | Minimal | Constrained environments |

The router balances these tradeoffs automatically by analyzing your prompt complexity. Force a model only if you know better than the classifier for your specific task.

## For Developers

Model routing implementation details and testing strategies are documented in `.github/copilot-instructions.md`:

- **Architecture:** Routing service, strategy pattern, context evaluation
- **Current Strategy:** ClassifierStrategy with complexity-based routing
- **Debugging:** How to inspect routing decisions, test strategies independently
- **Edge Cases:** Quota handling, bypass conditions, model resolution precedence
- **Testing:** Mocking strategies, integration test patterns, E2E validation

See `packages/core/src/routing/` for the implementation and `packages/core/src/routing/modelRouterService.test.ts` for examples.
