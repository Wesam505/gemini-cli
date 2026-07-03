# Gemini CLI - Coding Agent Instructions

This document guides AI agents in becoming productive contributors to the Gemini CLI codebase.

## Project Overview

**Gemini CLI** is a terminal-based AI agent framework that brings Google's Gemini model into the command line. It's a monorepo with three core packages:

- **`packages/cli`**: React/Ink-based terminal UI handling user input, rendering, and configuration
- **`packages/core`**: Backend orchestrating the Gemini API, tool execution, and lifecycle management
- **`packages/a2a-server`**: Experimental Agent-to-Agent server for multi-agent orchestration

## Architecture & Data Flow

### Request → Tool Execution → Response Loop

```
User Input (CLI)
    ↓
packages/cli/gemini.tsx (Ink React component)
    ↓
packages/core (API orchestration & tool routing)
    ↓
Tool Execution (file ops, shell, web fetch, MCP)
    ↓
Display/Response (back to CLI)
```

**Key Principle**: CLI and Core are intentionally decoupled. Core is backend-agnostic and can be consumed by different frontends (IDE companion, API servers, etc.).

### Hook System (Extensibility Layer)

Located in `packages/core/src/hooks/`, the hook system allows projects to inject behavior at critical points:

- **`before-agent`**: Pre-prompt processing
- **`before-model`**: API call preparation
- **`after-model`**: Response post-processing
- **`before-tool-selection`**: Tool routing logic
- **`allow-tool` / `block-tool`**: Tool execution filtering

Hooks are loaded from `~/.gemini/hooks/` (shell scripts or binaries). See `HookSystem`, `HookRegistry`, `HookRunner` for implementation patterns.

### Confirmation Bus & Policy Engine

The **MessageBus** (`packages/core/src/confirmation-bus/message-bus.ts`) mediates:

- Tool execution confirmations (for write/shell operations)
- Policy-based approvals (via `PolicyEngine` in `packages/core/src/policy/`)
- Bidirectional communication between CLI and Core

This ensures read-only ops execute silently, but destructive operations require user consent.

## Tool System Architecture

Tools are registered in `packages/core/src/tools/tool-registry.ts` and extend `BaseDeclarativeTool`:

### Built-in Tools

- **File Operations**: `read-file`, `write-file`, `edit` (smart merge-based), `read-many-files`
- **Search**: `glob`, `grep` (ripgrep wrapper), `ls`
- **Execution**: `shell` (sandboxable), `write-todos` (memory persistence)
- **External**: `web-search`, `web-fetch`
- **MCP Integration**: `mcp-tool` (dynamic tool discovery via Model Context Protocol)

Tools declare parameters as JSON Schema (`@google/genai` `FunctionDeclaration` format). Tool results flow back to Gemini API for continued conversation.

## Code Patterns & Conventions

### TypeScript/JavaScript

- **Prefer plain objects + interfaces over classes** — aligns with React's data flow model. Use ES modules (`export`/`import`) for encapsulation, not class privacy.
- **Avoid `any`; use `unknown` with type narrowing** instead.
- **Use exhaustive `switch` statements** with `checkExhaustive()` helper (see `packages/cli/src/utils/checks.ts`).
- **Leverage array operators** (`.map()`, `.filter()`, `.reduce()`) for immutable transformations.
- **Hyphens in flag names** (e.g., `--output-format` not `--output_format`).

### React/Ink Components (CLI UI)

- **Functional components with Hooks only** — no class components.
- **Keep render logic pure** — side effects belong in `useEffect` with explicit dependencies.
- **Context for shared state**: See `packages/cli/src/ui/contexts/` (e.g., `SessionContext` for conversation state).
- **React DevTools**: Debug with `DEV=true npm start` + `npx react-devtools@4.28.5`.
- **Avoid unnecessary memoization** — React Compiler handles optimization; write clear code instead.

### Testing

**Framework**: Vitest for all tests (unit & integration).

**Conventions**:

- Test files co-located with source (`.test.ts` / `.test.tsx`).
- Mock ES modules at the **top of the file** before other imports, especially Node.js built-ins (`fs`, `os`, `child_process`).
- Use `vi.hoisted(() => vi.fn())` if a mock is needed in the `vi.mock()` factory.
- For React components: use `render()` from `ink-testing-library`, assert with `lastFrame()`.
- Async tests use `async/await`; fake timers with `vi.useFakeTimers()`.

**Example**:
```typescript
vi.mock('fs/promises', async (importOriginal) => {
  const actual = await importOriginal();
  return { ...actual, readFile: vi.fn() };
});
```

## Build & Development Workflows

### Critical npm Scripts

- `npm run preflight` — **Full validation**: format, lint, build, type-check, test. **Always run before PR**.
- `npm run build` — TypeScript → `dist/`, handles bundling and asset copying.
- `npm start` — Runs CLI from source (dev mode, watches not enabled by default).
- `npm run test` — Unit tests in `packages/`.
- `npm run test:e2e` — Integration tests (in `integration-tests/`); validates end-to-end flows.
- `npm run debug` — Attach Chrome DevTools via `chrome://inspect` for debugging.
- `npm run lint:fix` — Auto-fix linting + format.

### Monorepo Structure

Root `package.json` uses **npm workspaces** (`"workspaces": ["packages/*"]`).

- Changes to one package must be tested in isolation: `npm run build --workspace @google/gemini-cli`.
- Shared scripts in `scripts/` (e.g., `build.js`, `clean.js`).
- Cross-package imports use published package names (e.g., `@google/gemini-cli-core`).

### Build Artifacts

- CLI builds to `packages/cli/dist/` → bundled into `bundle/gemini.js` (esbuild).
- Core builds to `packages/core/dist/`.
- Sandbox image (optional): `npm run build:all` includes Docker image build.

## Node.js & Environment

- **Required**: Node.js ≥ 20.19.0 (for dev); ≥ 20.0.0 for production.
- **Environment variables**: Loaded from `.env` (user project) and `~/.gemini/.env` (global settings).
- **Authentication**: Three paths: OAuth (Google account), API key, or Vertex AI. See `packages/core/src/config/` for resolution logic.

## Special Features & Integration Points

### MCP (Model Context Protocol) Integration

MCP servers provide dynamic tool discovery. See `packages/core/src/tools/mcp-client-manager.ts` for lifecycle:

1. **Config discovery**: `~/.gemini/settings.json` lists MCP servers.
2. **Connection**: `connectAndDiscover()` opens stdio/HTTP connections.
3. **Tool registration**: Discovered tools auto-added to registry.
4. **Execution**: MCP tool invocations routed through `DiscoveredToolInvocation`.

### Agent Routing & Sub-agents

Proof-of-concept **sub-agent system** in `packages/core/src/agents/`:

- `CodebaseInvestigatorAgent` — Specialized agent for deep codebase analysis before main agent acts.
- Agents have system prompts, tool access, and context windows.
- Routed via commands (e.g., `/investigate-codebase`).

### Sandboxing

Optional security layer for tool execution:

- **macOS Seatbelt**: `SEATBELT_PROFILE` configures sandbox (see `packages/cli/src/utils/sandbox-macos-*.sb`).
- **Docker/Podman**: `GEMINI_SANDBOX=docker|podman` in `.env`.
- **Custom proxies**: `GEMINI_SANDBOX_PROXY_COMMAND` restricts network access.

Set `GEMINI_SANDBOX=true` + run `npm run build:all` to enable.

## Common Development Tasks

### Adding a New Tool

1. Create `packages/core/src/tools/my-tool.ts` extending `BaseDeclarativeTool`.
2. Define parameters as JSON Schema (`FunctionDeclaration`).
3. Register in `tool-registry.ts`.
4. Add unit tests (`my-tool.test.ts`).
5. Test integration: `npm run test:e2e`.

### Modifying CLI Components

1. Edit React component in `packages/cli/src/ui/` or `packages/cli/src/commands/`.
2. Test with `npm start` (interactive) or unit tests with `render()` from ink-testing-library.
3. Update snapshot tests if UI changes.

### Debugging Gemini API Interactions

- Set `DEBUG=1 gemini` in shell or `.gemini/.env`.
- API requests/responses logged to console.
- Trace files in `~/.gemini/logs/`.

## Repository Standards

- **Main branch**: `main` (default).
- **Commits**: Follow Conventional Commits (e.g., `feat(cli): Add --json flag`, `fix(core): Tool timeout`).
- **PRs**: Link to issues, keep focused (one feature/fix per PR).
- **Linting**: ESLint + Prettier enforced in CI. Run `npm run preflight` locally.
- **Docs**: Markdown in `/docs`, referenced in `docs/sidebar.json`. Update docs with user-facing changes.

## Key Files to Know

| File/Directory | Purpose |
|---|---|
| `packages/cli/src/gemini.tsx` | Entry point, main Ink component |
| `packages/core/src/index.ts` | Core API exports |
| `packages/core/src/tools/tools.ts` | Tool base classes & types |
| `packages/core/src/hooks/hookSystem.ts` | Extensibility/hooks orchestration |
| `packages/core/src/confirmation-bus/message-bus.ts` | CLI ↔ Core communication |
| `GEMINI.md` | Project-specific AI development guidelines |
| `CONTRIBUTING.md` | Full contribution workflow |
| `integration-tests/` | End-to-end test suite |

## Quick Troubleshooting

**Tests fail with mock errors**: Ensure `vi.mock()` calls are at the top of the file, before imports.

**Build fails on TypeScript errors**: Run `npm run typecheck` locally; type errors block builds.

**CLI won't start**: Check Node.js version (`node --version`), confirm `npm run build` succeeded.

**Tool execution blocked**: Verify policy rules in `~/.gemini/` and check `packages/core/src/policy/` for policy logic.

---

*For more details, see `CONTRIBUTING.md`, `GEMINI.md`, and `docs/architecture.md`.*
