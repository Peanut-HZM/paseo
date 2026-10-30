# ZCode Provider Design

Date: 2026-10-30
Status: Draft for review
Scope: Add a ZCode provider to Paseo as a bundled plugin. The ZCode repository is not modified.

## Decision summary

| Question                                                                     | Decision                                                                                                                    |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Where does the adapter live?                                                 | A bundled plugin under `plugins/zcode-provider/`. Paseo core gains no provider entry, no factory, no manifest id.           |
| Which ZCode protocol surface?                                                | Two transports behind one interface. Legacy `session/*` first, V4 `v4/*` second.                                            |
| How does the provider find the ZCode binary?                                 | A user override first, then `zcode` on PATH, then the runtime bundled inside the ZCode desktop app, then `GLM_BINARY_PATH`. |
| Does the change touch Paseo core?                                            | One small general fix in `plugin-provider.ts` so a plugin can own binary discovery. It lands as its own PR.                 |
| Is the goal "Paseo can drive ZCode" or "ZCode becomes a standard ACP agent"? | The first. ZCode exporting ACP is a separate product decision and is out of scope here.                                     |

## 1. Goal and constraints

### 1.1 Goal

Paseo can open a ZCode session, stream the agent's replies, render tool calls, answer permission prompts, switch model and mode, list and restore sessions, and rewind — with ZCode unmodified.

Rewind and steer arrive with the V4 transport in phase P3. Everything else is reachable over the legacy transport in P1 and P2.

### 1.2 Constraints

- The ZCode repository stays untouched. The provider uses the protocol that `zcode app-server --stdio` already speaks.
- The provider ships as a bundled plugin. `docs/providers.md` routes new providers through the plugin SDK, and states that a plugin needs no core manifest entry or provider factory.
- Protocol differences between ZCode's legacy and V4 surfaces are contained in one interface. The mapping layer above it never learns which transport is active.
- The provider declares only the capabilities its active transport can actually deliver.

### 1.3 Non-goals

- ZCode exposing ACP to third-party editors.
- ZCode's desktop app speaking ACP. The desktop already drives the same CLI runtime and has a stronger remote-control surface.
- Structured output from ZCode. ZCode has no such concept, so `prompt.output_schema` stays unsupported.
- Backfilling sessions that ZCode cannot list.

## 2. Current state

### 2.1 ZCode

ZCode's agent runtime is reachable through exactly one seam: the `zcode app-server --stdio` command, which speaks a JSON-RPC-shaped protocol over NDJSON on stdio.

Two protocol generations coexist.

- Legacy. Method table in `packages/shared/src/zcode-protocol/index.ts:4109-4242`. Session lifecycle lives at `session/create`, `session/resume`, `session/list`, `session/send`, `session/stop`, `session/close`, `session/setModel`, `session/setMode`, `session/setThoughtLevel`. Streaming arrives as `session/event` with 25 event types (`packages/shared/src/zcode-protocol/index.ts:1459-1485`). The agent asks the host for permission through the reverse request `interaction/requestPermission` (`index.ts:4235`).
- V4. Method table in `packages/shared/src/zcode-protocol-v4/transport.ts:337-391`. Writes go through a single `v4/command` envelope; reads are topic subscriptions that deliver a snapshot followed by ordered deltas.

Three properties shape the design:

1. **Legacy is shrink-only.** `packages/shared/src/zcode-protocol/index.ts:32` states that the whole file goes away when the old client/server groups are removed. `session/rewind`, `session/steer`, `session/applyFileRewind` and `session/previewFileRewind` are already gone — they are listed as deleted words at `index.ts:25-28`. Legacy therefore cannot serve rewind or steer.
2. **Legacy frames carry no `jsonrpc` field.** The frame schemas are `{id, method, params?, trace?}` and strict (`index.ts:407-446`).
3. **ZCode answers a prompt immediately.** `session/send` acknowledges and lets the turn run in the background; the comment at `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server-operations.ts:2103-2106` says the request cannot wait for the model round to finish. Turn boundaries arrive as `turn.started`, `turn.completed` and `turn.failed` events.

The app-server already hosts many sessions in one process: `sessions: Map<string, ZCodeProtocolSessionRecord>` at `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server-operations.ts` and `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server.ts:163`.

The agent binary ships two ways. The npm package `@zcode/cli` exposes the `zcode` binary. The desktop app bundles the runtime as `zcode.cjs` or a native `zcode-agent` under `resources/glm/` (`packages/shared/src/zcode-agent-runtime.ts:29-45`).

### 2.2 Paseo

Paseo already supports providers that do not speak ACP.

- Plugin providers own their transport, session state, catalog and capabilities (`docs/providers.md`). The bundled examples are `plugins/antigravity-provider/` and `plugins/muse-provider/`.
- The provider contract is `ProviderRegistration` in `packages/plugin/src/server/provider.ts:29-43`. A connection receives `ProviderInput` values and publishes `ProviderEvent` values (`provider.ts:895-960` and `provider.ts:1296-1435`).
- 21 capabilities exist (`provider.ts:6-25`).
- One connection owns many sessions (`packages/server/src/server/agent/plugin-provider.ts:130-136`).

One gap blocks the binary-discovery requirement. `packages/server/src/server/agent/plugin-provider.ts:157-170` resolves the launch command:

- When the plugin declares no default command, the method returns `undefined` before it reads the user's `agents.providers.<id>.command` override, so that override is ignored.
- When the default command is declared but the binary is missing, the method throws. The plugin's `status()` callback never runs, so a plugin cannot fall back to its own discovery.

## 3. Chosen approach

Three approaches were considered.

| Approach                                                  | Summary                                                                                        | Verdict                                                                                                                                               |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| ZCode exports ACP; Paseo consumes it via `extends: "acp"` | ZCode implements an ACP facade over its runtime.                                               | Rejected for now. It needs a product decision on ZCode's side, and ZCode removed its ACP code on purpose.                                             |
| Paseo writes a direct provider in core                    | New `packages/server/src/server/agent/providers/zcode-agent.ts`, core manifest entry, factory. | Rejected. `docs/providers.md` routes new providers through the plugin SDK, and a core provider makes every ZCode protocol change a Paseo core change. |
| Bundled plugin with a swappable transport                 | `plugins/zcode-provider/`, one transport interface, legacy then V4 implementations.            | Chosen. Keeps ZCode untouched, keeps Paseo core free of provider-specific code, and confines protocol churn to one directory.                         |

The mapping work is identical in the second and third approaches. Only its location differs, and the plugin location costs less governance.

## 4. Architecture

### 4.1 Deliverables

| #   | Path                                                  | Change                                |
| --- | ----------------------------------------------------- | ------------------------------------- |
| 1   | `packages/server/src/server/agent/plugin-provider.ts` | Launch resolution fix, in its own PR. |
| 2   | `plugins/zcode-provider/`                             | New bundled plugin.                   |

### 4.2 Module layout

```
plugins/zcode-provider/
  index.server.ts                 registerProvider(createProvider())
  paseo-plugin.json               { "id": "zcode-provider", "requirements": { "paseo": ">=0.14.0-beta.2" } }
  icon.svg                        single-color, currentColor, viewBox 0 0 24 24
  server/
    provider.ts                   ProviderRegistration: id, label, icon, status, connect
    connection.ts                 ProviderConnection: send / onEvent / close, session table
    internal/
      binary.ts                   binary discovery
      transport.ts                ZCodeTransport interface and shared types
      legacy-transport.ts         session/* over NDJSON
      v4-transport.ts             v4/command + v4/conversation/frame
      wire.ts                     NDJSON framing and ZCode method calls
      mapping.ts                  ZCode semantics to Paseo semantics, pure functions
      timeline.ts                 ZCode events to ProviderEvent
      signals.ts                  permission round trip
      catalog.ts                  model, mode and command discovery
```

`mapping.ts` holds the semantic translation as pure functions so it can be tested without a process.

### 4.3 Transport interface

```ts
interface ZCodeTransport {
  /** Capabilities this transport can actually deliver. */
  readonly capabilities: readonly ProviderCapability[];
  createSession(params: ZCodeCreateSessionParams): Promise<ZCodeSessionHandle>;
  resumeSession(params: ZCodeResumeSessionParams): Promise<ZCodeSessionHandle>;
  send(input: ZCodeSendInput): Promise<void>;
  stop(input: ZCodeStopInput): Promise<void>;
  listSessions(input: ZCodeListSessionsInput): Promise<ZCodeSessionSummary[]>;
  setModel(input: ZCodeSetModelInput): Promise<void>;
  setMode(input: ZCodeSetModeInput): Promise<void>;
  setThoughtLevel(input: ZCodeSetThoughtLevelInput): Promise<void>;
  onEvent(listener: (event: ZCodeEvent) => void): () => void;
  close(): Promise<void>;
}
```

### 4.4 Capability negotiation

The connection's capability set is the intersection of what the active transport declares and what the client requests.

The legacy transport omits `session.revert.conversation`, `session.revert.files`, `session.revert.both` and `prompt.steer`. Paseo therefore shows no rewind or steer affordance while legacy is active. Switching to the V4 transport makes them appear. The provider and the app need no change for that to happen.

### 4.5 Process model

One plugin connection owns one ZCode CLI process, and that process serves every Paseo session opened through the connection. This matches the app-server's own session map and the plugin connection contract.

The CLI reads user configuration from `~/.zcode/cli/config.json`, which is per user rather than per workspace, so sharing one process across workspaces does not mix configurations. A process crash affects every session it hosted; the provider reports the failure per session and the user can reopen.

### 4.6 Core change, in its own PR

`packages/server/src/server/agent/plugin-provider.ts:157-170` gets two behavior changes.

- When the plugin declares no default command but the user configured `agents.providers.<id>.command`, the override still applies.
- When the default command's binary is missing and the user did not override it, resolution returns `undefined` instead of throwing, so the plugin's `status()` performs its own discovery. An override that points at a missing binary still fails hard, because a wrong explicit setting must be visible.

The change is under ten lines and benefits any plugin that owns binary discovery. It is reviewed and merged on its own before the plugin work starts, so the plugin PR contains no core diff.

## 5. Protocol mapping

### 5.1 Inputs

| ProviderInput                          | ZCode method                                                      | Conversion                                                                                                                                                |
| -------------------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                              | `session/read`, `provider/refreshRegistry`                        | Models, modes and thought levels come from `configOptions`.                                                                                               |
| `sessions`                             | `session/list`                                                    | Optional `workspace` passes through.                                                                                                                      |
| `session.open`                         | `session/create`, or `session/resume` when persistence is present | `config.cwd` becomes `workspace.workspacePath`; `workspaceKey` is `workspaceIdentity?.trim() \|\| workspacePath`.                                         |
| `session.prompt`                       | `session/send`                                                    | Paseo's prompt parts become ZCode's `content` string plus `attachments`. The provider generates an `inputId` and sends it as the turn correlation anchor. |
| `session.interrupt`                    | `session/stop`                                                    | See section 5.6 for the stop boundary.                                                                                                                    |
| `session.permission`                   | answers the pending `interaction/requestPermission`               | See section 5.3.                                                                                                                                          |
| `session.configure`                    | `session/setModel`, `session/setMode`, `session/setThoughtLevel`  | ZCode modes `plan`, `build`, `edit`, `yolo`, `guarded`, `auto` pass through unchanged.                                                                    |
| `session.archive`, `session.unarchive` | `session/close` with persistence, `session/resume`                |                                                                                                                                                           |
| `session.can_stop_runtime`             | `session/read`                                                    | Answered from the session status.                                                                                                                         |
| `session.close`                        | `session/close`                                                   |                                                                                                                                                           |
| `session.revert`                       | none in legacy                                                    | The capability stays undeclared. V4 supplies `applyFileRewind` and `editUserQuery`.                                                                       |

The persistence handle is `{ version: 1, data: { nativeSessionId, workspaceKey, workspacePath } }`. `nativeSessionId` is the ZCode session id and feeds `session/resume` directly.

### 5.2 Outputs

| ZCode event                              | Payload highlights                                                                  | ProviderEvent                                                                         |
| ---------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `turn.started`                           | `turnNumber`, `input`, `inputId`                                                    | `session.turn` with state `started`                                                   |
| `turn.completed`                         | `response`, `tokenCount`, `usage`, `toolCallCount`, `duration`, `cacheStats`        | `session.turn` with state `completed`, plus `session.usage`                           |
| `turn.failed`                            |                                                                                     | `session.turn` with state `failed` and an error                                       |
| `message.upserted`                       | `content`, `type`, `attachments`                                                    | `timeline.item` for a user or assistant message                                       |
| `part.delta`                             | `messageId`, `partId`, `field` of `text`, `reasoning`, `input` or `output`, `delta` | `timeline.item`; `text` maps to an assistant message, `reasoning` to a reasoning item |
| `part.upserted`                          | `part`                                                                              | `timeline.item`                                                                       |
| `tool.updated`                           | `kind`, `toolCallId`, `toolName`, `input`, `dependencies`, `parallelGroupIndex`     | `timeline.item` for a tool call                                                       |
| `permission.requested`                   | `requestId`, `toolCallId`, `toolName`, `reason`, `riskLevel`, `input`, `options`    | `session.permission`                                                                  |
| `permission.resolved`                    |                                                                                     | `session.permission_resolved`                                                         |
| `userInput.requested`                    |                                                                                     | `session.notice`                                                                      |
| `session.titleUpdated`                   |                                                                                     | session title update                                                                  |
| `checkpoint.created`, `rewind.triggered` |                                                                                     | `timeline.item` marking the rewind                                                    |
| `streamRecovery.updated`                 |                                                                                     | rebuild the timeline snapshot                                                         |

### 5.3 Permission round trip

ZCode's permission options already carry the answer to send back:

```
zcodePermissionOptionSchema = { optionId, kind, name, description?, response }
zcodePermissionResponseSchema = { decision: allow|deny|escalate|modify,
                                  reason?, modifiedInput?, permissionUpdates? }
```

The provider maps ZCode's `options` into `ProviderPermissionRequest.options`, uses ZCode's `requestId` as the permission id, and returns the selected option's `response` unchanged.

`permissionUpdates` carries `addRules` with an `allow`, `deny` or `ask` behavior, which is how ZCode persists "always allow" and "always deny". The `permission.tool_policy` capability therefore needs no extra work.

### 5.4 Turn timing and turn correlation

ZCode acknowledges a prompt immediately, while Paseo expects a turn that starts, ends and reports a result.

```
Paseo UI                 provider                ZCode CLI
   |  session.prompt         |                       |
   |------------------------>|  session/send         |
   |                         |---------------------->|   acknowledges immediately
   |  prompt_result {turn}   |<----------------------|
   |  session.turn started   |<-- turn.started ------|
   |  timeline.item x N      |<-- part.delta --------|
   |  timeline.item (tool)   |<-- tool.updated -------|
   |  session.permission     |<-- permission.requested
   |      (user decides)     |                       |
   |  session.permission     |---------------------->|   selected option's response
   |  session.turn completed |<-- turn.completed ----|
   |  session.usage          |<-- tokenCount --------|
```

`turn.started` carries `turnNumber` and `inputId` but no turn id, so the provider generates the turn id and sends the same `inputId` in `session/send`. The anchor is what keeps a queued or concurrent turn from being attributed to the wrong run.

### 5.5 Capability matrix

| Capability                                       | Legacy | V4  | Note                                                            |
| ------------------------------------------------ | ------ | --- | --------------------------------------------------------------- |
| `prompt.message`                                 | yes    | yes |                                                                 |
| `prompt.command`                                 | yes    | yes | `available_commands_update` to `session.commands`               |
| `prompt.image`                                   | yes    | yes | attachments                                                     |
| `prompt.steer`                                   | no     | yes | `session/steer` is a deleted word; V4 queue commands replace it |
| `prompt.output_schema`                           | no     | no  | ZCode has no structured output concept                          |
| `session.configure`                              | yes    | yes | model, mode, thought level                                      |
| `session.list`                                   | yes    | yes |                                                                 |
| `session.persistence`                            | yes    | yes |                                                                 |
| `session.archive`, `session.unarchive`           | yes    | yes |                                                                 |
| `session.can_stop_runtime`                       | yes    | yes |                                                                 |
| `session.subsession`                             | yes    | yes | `session/subagents`                                             |
| `session.revert.conversation`, `.files`, `.both` | no     | yes | rewind words are deleted in legacy                              |
| `permission`                                     | yes    | yes |                                                                 |
| `permission.tool_policy`                         | yes    | yes | via `permissionUpdates`                                         |
| `timeline.plugin`                                | yes    | yes |                                                                 |

### 5.6 Stop boundary

`docs/providers.md` requires a provider that cancels at session scope to own the stop boundary, and Paseo's Gotchas spell out the rules: model the stop as an explicit `stopping` turn state, retry the in-flight stop when the user presses Stop again instead of opening a second one, gate daemon-issued operations on both the terminal and the cancel settlement, and keep permission and question responses outside that gate so auto-approve cannot deadlock the stop. If the cancel never succeeded, new runs are refused until the next Stop issues a fresh cancel.

ZCode's two stop surfaces differ in exactly this respect.

- Legacy `session/stop` takes only `sessionId` (`packages/shared/src/zcode-protocol/index.ts:2270-2274`). A late stop can cancel an unrelated later turn.
- V4 `stop` accepts `expectedForegroundExecutionId` (`packages/shared/src/zcode-protocol-v4/command.ts:161-164`), and the CLI rejects a stop whose identifier no longer matches the active execution.

The provider implements the boundary on its own for the legacy transport: it tracks the foreground execution id it observed for the canceled turn, passes it to V4 when available, refuses new runs while a cancel is unsettled, and treats a second Stop as a retry of the first.

## 6. Binary discovery

Resolution order, first match wins.

1. The user's `agents.providers.zcode.command` override, applied by the daemon.
2. `zcode` on PATH.
3. The runtime bundled with the ZCode desktop app: `zcode.cjs` or the native `zcode-agent` under the app's `resources/glm/` directory.
4. The `GLM_BINARY_PATH` environment variable.

`status()` reports which path was used in its diagnostic. When nothing resolves, it reports `available: false` with a diagnostic that lists all four options so the user can fix the provider without reading the source.

## 7. Error handling and lifecycle

| Situation                                  | Behavior                                                                                            | What the user sees                                    |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| No binary found                            | `status()` returns `available: false` with a diagnostic naming all four paths                       | The provider is dimmed with actionable diagnostics    |
| A user override points at a missing binary | Hard failure                                                                                        | The same, naming the configured path                  |
| The CLI exits at startup                   | Capture the exit code and stderr, publish `session.runtime_failed`                                  | An error banner inside the session, retryable         |
| A frame carries unknown fields             | The decoder drops the unknown fields and keeps the event                                            | Nothing; a debug log only                             |
| A frame is structurally broken             | Count it in the diagnostic, do not kill the connection                                              | Diagnostic only                                       |
| ZCode answers `method-not-found`           | The only downgradeable error; mark that capability unavailable and continue                         | The feature disappears, everything else keeps working |
| Any other synchronous failure              | Propagate as `request.failed`                                                                       | The error surfaces in the requesting context          |
| The CLI process dies                       | Publish `session.runtime_failed` and `session.closed` for every hosted session                      | Sessions turn red and can be reopened                 |
| A turn never ends                          | Paseo sets no turn timeout; `session.interrupt` and `session.can_stop_runtime` let the user stop it | The user can interrupt                                |
| A permission request stays unanswered      | The pending request is kept; an interrupt answers it with deny so the tool is released              | The tool does not hang forever                        |
| stderr noise                               | Collected into the diagnostic and never mixed into stdout, which carries frames only                | Visible in diagnostics                                |

ZCode's frame schemas are strict and keep gaining fields, so the decoder stays more tolerant than ZCode: unknown fields are dropped and unknown event types are ignored, but a broken frame is never allowed to end the connection.

## 8. Phased plan

### P0 — Core fix and skeleton (2 days)

| Change                                                                                 | Path                                                          |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Launch resolution fix                                                                  | `packages/server/src/server/agent/plugin-provider.ts:157-170` |
| Regression tests for that fix                                                          | `packages/server/src/server/agent/plugin-provider.test.ts`    |
| Plugin skeleton, icon, registration, `status()`, binary discovery, transport interface | `plugins/zcode-provider/**`                                   |

Lands as its own PR.

### P1 — Legacy transport and the core loop (1.5 weeks)

`LegacyTransport` covering NDJSON framing, requests, responses, event dispatch and process lifecycle; the mapping layer covering turn state, timeline projection and turn correlation; capabilities `prompt.message`, `prompt.command`, `prompt.image`, `permission`, `permission.tool_policy`, `session.configure`, `session.list`, `session.persistence`, `session.archive`, `session.unarchive`, `session.can_stop_runtime`, `session.subsession`, `timeline.plugin`.

### P2 — Remaining parity work (1 week)

Catalog discovery for models, modes and thought levels; usage reporting from `tokenCount`; slash commands from `available_commands_update`; session listing and restore through `session/list`, `session/resume` and the persistence handle.

### P3 — V4 transport (2 weeks)

Topic subscription, snapshot and delta handling, sequence and log-epoch watermarks, gap detection with resync, CAS commands and the six acknowledgement states. Enables `session.revert.*` and `prompt.steer`.

Transport selection is controlled by the `PASEO_ZCODE_PROTOCOL` environment variable, with values `auto`, `legacy` and `v4`. `auto` prefers V4 and falls back to legacy.

The environment variable is the channel because a plugin provider's runtime settings carry only `command`, `env` and `disallowedTools` (`packages/protocol/src/provider-config.ts:25-29`). There is no provider-options channel for plugins, so a configured `agents.providers.zcode.options` record would never reach the plugin. The provider also accepts the same value through the session-level `providerOptions` record, which callers already pass when creating an agent.

**The default flip is an acceptance gate, not part of P3.** `auto` resolves to legacy until the V4 transport passes the real-ZCode local end-to-end smoke test. After that it resolves to V4, because V4 is the surface ZCode keeps and legacy is scheduled for removal. Legacy stays available as an explicit choice for as long as ZCode ships it.

The plugin's `requirements.paseo` floor is the release that carries the section 4.6 launch-resolution fix. The fix and the plugin therefore ship in the same release train.

Total is about five weeks, of which the first three produce a usable provider.

## 9. Test strategy

`docs/testing.md` allows two categories: unit tests with ports and adapters, and real end-to-end tests. No mocks of the module under test, no JSDOM, no component mounting.

### 9.1 Unit tests (`*.test.ts`)

| Module                 | Cases                                                                                                                                                                                                    |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `binary.ts`            | PATH hit; PATH miss with the desktop bundle present; nothing found yields `available: false` with all four paths in the diagnostic; a user override wins over the other paths                            |
| `wire.ts`              | Frames split on LF; a non-JSON frame is skipped without killing the connection; an unknown method is ignored; an unknown field is dropped but the event is still delivered; a large frame does not block |
| `mapping.ts`           | A snapshot assertion for every ZCode event to `ProviderEvent` mapping; turn correlation; mode and model value mapping; permission option mapping                                                         |
| Turn state machine     | `started` to `completed`; `started` to `failed`; `started` to `canceled`; an interrupt with no terminal state                                                                                            |
| Capability negotiation | The legacy transport omits `revert.*` and `steer`; the V4 transport declares them; the intersection with the client's request                                                                            |
| Permission round trip  | The selected option's `response` is returned unchanged; `permissionUpdates` pass through                                                                                                                 |
| `plugin-provider.ts`   | A user override applies when no default command is declared; a missing default binary does not fail hard; a missing user override target still fails hard                                                |
| Stop boundary          | A second Stop retries the in-flight cancel instead of opening a second one; new runs are refused while a cancel is unsettled; permission responses stay outside the gate                                 |

### 9.2 End-to-end tests

| Case                          | Mechanism                                                                                                                                             | Suffix                               |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| Real subprocess protocol loop | A fake ZCode app-server: a Node script that speaks the real NDJSON protocol, driven by fixtures. Follows `plugins/antigravity-provider/test/agy.mjs`. | `*.test.ts`                          |
| Real ZCode CLI loop           | The `zcode app-server --stdio` binary from a local checkout                                                                                           | `*.local.e2e.test.ts`                |
| Live provider smoke           | A real ZCode account                                                                                                                                  | `*.real.e2e.test.ts`                 |
| App behavior                  | Paseo app Playwright: a failed permission prompt stays visible and can be retried                                                                     | `packages/app/e2e/browser/*.spec.ts` |

ZCode handles its own authentication, so tests carry no auth checks, no environment gates and no conditional skips. The default unit suite uses deterministic fake adapters so CI never depends on a real account.

## 10. Acceptance criteria

### P0

- [ ] A user override applies when the plugin declares no default command, covered by a core regression test.
- [ ] A missing default binary does not fail the launch resolution hard.
- [ ] The provider appears in Paseo's provider list.
- [ ] With no ZCode installed, the provider is unavailable and the diagnostic names all four resolution paths.

### P1

- [ ] The first assistant text appears within three seconds of sending a prompt and then keeps growing incrementally.
- [ ] Tool calls render as tool calls with a name and a status, not as raw JSON.
- [ ] A permission prompt appears, the user's decision releases the tool, and the turn completes.
- [ ] Interrupting a running turn stops it and the turn ends in the `canceled` state.
- [ ] Killing the CLI produces a visible error and a reopenable session, with no uncaught exception.

### P2

- [ ] Model, mode and thought level can be switched and take effect.
- [ ] The session list shows ZCode's own sessions.
- [ ] A session restores after an app restart, including its history.
- [ ] Usage shows token counts.
- [ ] Slash commands appear in the `/` menu.

### P3

- [ ] Conversation rewind and file rewind are available and appear in Paseo.
- [ ] Steering a running turn works.
- [ ] Ordered deltas arrive without gaps under the V4 transport, and an injected gap triggers a resync that keeps the body text.
- [ ] `PASEO_ZCODE_PROTOCOL` behaves as `auto`, `legacy` and `v4` are documented to behave.
- [ ] The V4 transport passes the real-ZCode local end-to-end smoke test, which is the condition for flipping the `auto` default.

### Whole change

- [ ] `npm run typecheck`, `npm run lint` and `npm run format` are clean.
- [ ] The ZCode repository has no diff.

## 11. Risks and open questions

- **Legacy removal.** ZCode will delete the legacy protocol. The provider follows, so the V4 transport is the long-term answer. The transport interface is what keeps this a migration rather than a rewrite.
- **V4 churn.** V4 is ZCode's active surface and still moving. The `auto` default stays on legacy until the V4 transport passes a real smoke test, which limits the blast radius of upstream churn.
- **Shared process blast radius.** One CLI process serves every session, so a crash affects all of them. The alternative, one process per session, multiplies CLI startup cost and diverges from ZCode's own model. Open question worth revisiting if crashes prove common in practice.
- **No ZCode-side changes.** Anything the provider cannot reach through the current protocol surface stays out of scope. If a future need requires a protocol addition, the ZCode-side change becomes its own project.
- **Desktop runtime discovery.** Path layout of the ZCode desktop bundle differs per platform and per installation. The discovery logic needs real-world paths for each supported platform before it can be called done.

## 12. References

- `docs/providers.md` — adding a provider, the plugin SDK route, provider Gotchas
- `public-docs/plugins/providers.md` — the provider plugin contract
- `packages/plugin/src/server/provider.ts` — `ProviderRegistration`, `ProviderInput`, `ProviderEvent`, capabilities
- `packages/server/src/server/agent/plugin-provider.ts` — plugin provider wrapper, launch resolution
- `docs/testing.md` — test categories, file naming, CI routing
- ZCode `packages/shared/src/zcode-protocol/index.ts` — legacy method table, event types, frame schemas
- ZCode `packages/shared/src/zcode-protocol-v4/transport.ts` — V4 method table
- ZCode `packages/shared/src/zcode-protocol-v4/command.ts` — V4 command payloads
- ZCode `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server.ts` — session map
- ZCode `packages/shared/src/zcode-agent-runtime.ts` — bundled runtime descriptor
