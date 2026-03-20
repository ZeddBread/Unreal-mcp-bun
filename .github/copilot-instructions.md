# Copilot instructions for Unreal MCP

## Big picture

- Dual-process system: TypeScript MCP server in `src/` + Unreal Editor plugin in `plugins/McpAutomationBridge/` (+ optional Rust/WASM in `wasm/`).
- Data flow: MCP tool call → TS dispatch (`src/tools/consolidated-tool-handlers.ts`) → `executeAutomationRequest()` → WebSocket client (`src/automation/bridge.ts`) → UE subsystem dispatch (`UMcpAutomationBridgeSubsystem::ProcessAutomationRequest()` in `plugins/.../McpAutomationBridge_ProcessRequest.cpp`) → handler registered in `UMcpAutomationBridgeSubsystem::InitializeHandlers()`.

## Non-obvious constraints (don’t break MCP I/O)

- Keep stdout JSON-only: runtime logs must go through `Logger` (see `routeStdoutLogsToStderr()` in `src/index.ts`). Avoid `console.log` in runtime code.
- `.env` loading is silenced to protect stdout (see `src/config.ts`).

## Key dev commands (from `package.json`)

- Dev: `bun run dev`
- Build: `bun run build` (WASM build is best-effort) / `bun run build:core`
- Tests: `bun run test` (integration via `tests/integration.mjs`) and `bun run test:unit` (vitest)
- Plugin sync: `bun run automation:sync`

## Connection + ports

- The UE plugin listens by default on `127.0.0.1` ports `8090,8091` (see `ListenPorts` in `plugins/.../McpAutomationBridgeSettings.cpp`).
- The TS bridge connects as a WebSocket client; override with env like `MCP_AUTOMATION_CLIENT_PORT` or `MCP_AUTOMATION_WS_PORTS` (see `src/automation/bridge.ts`).
- Offline mode: set `MOCK_UNREAL_CONNECTION=true` to force `UnrealBridge.tryConnect()` to succeed.

## Tooling conventions (TS)

- Tool schemas live in `src/tools/consolidated-tool-definitions.ts` (action enums + input/output schemas). Output schemas are registered at startup in `src/index.ts` and validated by AJV in `src/utils/response-validator.ts`.
- Tool routing is via `toolRegistry.register()` in `src/tools/consolidated-tool-handlers.ts` and domain logic in `src/tools/handlers/*-handlers.ts`.
- Always call Unreal through `executeAutomationRequest()` (`src/tools/handlers/common-handlers.ts`) rather than raw WS.

## Safety + normalization

- Console commands are safety-filtered by `src/utils/command-validator.ts`; don’t bypass it.
- UE calls are throttled/retried via `src/utils/unreal-command-queue.ts`.
- Prefer `/Game/...` paths; some layers normalize `/Content` → `/Game` (e.g. `src/resources/assets.ts`, `src/tools/assets.ts`, and plugin helpers in `plugins/.../McpAutomationBridgeHelpers.h`).

## Adding a new action end-to-end

1) TS: extend the tool’s `action` enum + schemas in `src/tools/consolidated-tool-definitions.ts`.
2) TS: route the action in `src/tools/consolidated-tool-handlers.ts` (or the appropriate `src/tools/handlers/*-handlers.ts`).
3) C++: implement the action in an appropriate `plugins/.../*Handlers.cpp` and register it in `UMcpAutomationBridgeSubsystem::InitializeHandlers()`.
4) Tests: add/extend a case in `tests/integration.mjs`.
## Architecture
- This is a dual-process system: the TypeScript MCP server lives in `src/`, and the Unreal Editor bridge plugin lives in `Plugins/McpAutomationBridge/`.
- Typical flow is schema validation -> tool registry dispatch -> domain handler -> `executeAutomationRequest()` -> WebSocket bridge -> C++ subsystem -> game-thread handler.
- Keep workspace-wide guidance here and rely on the closer `AGENTS.md` files for area-specific detail under `src/tools/`, `src/tools/handlers/`, `src/automation/`, `tests/`, and `Plugins/McpAutomationBridge/`.

## Critical constraints
- Keep stdout JSON-only. Runtime logs must go through the project logger; do not use `console.log` in runtime code. See `routeStdoutLogsToStderr()` in `src/index.ts`.
- `.env` loading is intentionally quiet to avoid corrupting MCP I/O. Keep it that way in `src/config.ts`.
- Do not bypass the tool routing stack. Register tool behavior via `toolRegistry.register()` and send Unreal work through `executeAutomationRequest()`, not raw WebSocket calls.
- Preserve path normalization. Prefer `/Game/...` asset paths and do not add new code that depends on `/Content/...` input staying unnormalized.

## UE 5.7 safety
- Do not use `UPackage::SavePackage()` in plugin code. Use the safe helper wrappers in `Plugins/McpAutomationBridge/Source/McpAutomationBridge/Public/McpAutomationBridgeHelpers.h`.
- For Blueprint component templates, let SCS own nodes and templates through `CreateNode()` and `AddNode()` patterns.
- Do not introduce new `ANY_PACKAGE` usage; use modern lookup patterns such as `nullptr` where required by newer UE versions.

## Build and test
- `bun run dev`: run the server from TypeScript.
- `bun run build:core`: compile the TypeScript server.
- `bun run automation:sync`: sync the Unreal plugin into a target project.
- `bun run test:unit`: run Vitest unit tests without Unreal.
- `bun run test:smoke`: run the offline smoke path using mock connection mode.
- `bun run test`: run the MCP integration suite. This requires Unreal Editor with the bridge plugin available.
- `bun run test:all`: run the integration entrypoints; treat this as Unreal-dependent unless you intentionally configured mock coverage.

## Connection and runtime behavior
- The Unreal plugin listens on loopback by default and commonly uses ports `8090` and `8091`.
- The TypeScript bridge connects as a WebSocket client and can be steered with env such as `MCP_AUTOMATION_CLIENT_PORT`, `MCP_AUTOMATION_PORT`, or `MCP_AUTOMATION_WS_PORTS`.
- Mock mode is an explicit development path: set `MOCK_UNREAL_CONNECTION=true` when you need the bridge layer to succeed without a live editor.
- Connection setup includes handshake and capability negotiation in `src/automation/`; keep protocol changes aligned across TypeScript and plugin code.

## Tooling conventions
- Tool schemas and action enums live in `src/tools/consolidated-tool-definitions.ts`. Treat that file as the source of truth for inputs and outputs.
- Output schemas are registered during startup and validated before responses leave the server. Keep new tool outputs schema-backed.
- Console commands are safety-filtered in `src/utils/command-validator.ts`; do not add bypasses.
- Unreal requests are queued and retried through the existing command queue utilities; reuse that path instead of inventing parallel transport logic.

## Making changes
- New MCP action flow:
	1. Add the action enum and schemas in `src/tools/consolidated-tool-definitions.ts`.
	2. Route the action in `src/tools/consolidated-tool-handlers.ts` or the relevant handler module under `src/tools/handlers/`.
	3. Implement the Unreal side in the appropriate handler under `Plugins/McpAutomationBridge/Source/` and register it in `UMcpAutomationBridgeSubsystem::InitializeHandlers()`.
	4. Add or update tests in `tests/integration.mjs`, `tests/mcp-tools/`, or colocated unit tests as appropriate.
- Keep TypeScript strict. Avoid `as any` in runtime code.
- If you change versions, update all version-bearing files together, including `package.json`, `server.json`, `src/index.ts`, and `Plugins/McpAutomationBridge/McpAutomationBridge.uplugin`.

## Reference points
- Use `src/tools/handlers/` for examples of handler structure and `executeAutomationRequest()` usage.
- Use `tests/AGENTS.md` for integration test conventions, expected result strings, and timeout tiers.
- Use `Plugins/McpAutomationBridge/AGENTS.md` for plugin-side patterns and Unreal-specific caveats.
