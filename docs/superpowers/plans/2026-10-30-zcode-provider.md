# ZCode Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a bundled Paseo plugin that drives ZCode over its existing `zcode app-server --stdio` protocol, so Paseo can open a ZCode session, stream replies, render tool calls, answer permissions, switch model and mode, and list and restore sessions — with the ZCode repository unmodified.

**Architecture:** The plugin owns its transport, session state, catalog and capabilities, following the `plugins/antigravity-provider/` pattern. ZCode's two protocol generations sit behind one `ZCodeTransport` interface; the legacy implementation lands first, and the V4 implementation follows in a separate plan. The provider never sends a prompt until a pending cancellation has settled, and it never lets a late stop reach the wrong turn.

**Tech Stack:** TypeScript, Zod, `@getpaseo/plugin/server` (`spawnProcess`, `execCommand`, `terminateProcess`), `@getpaseo/plugin/server/provider` (`ProviderRegistration`, `ProviderInput`, `ProviderEvent`), Vitest, Biome.

**Spec:** `docs/superpowers/specs/2026-10-30-zcode-provider-design.md`

## Global Constraints

These apply to every task. Copy them into the task brief you hand an implementer.

- The ZCode repository at `/Users/huazhongmin/IdeaProjects/ZCode` is read-only for this work. Nothing in this plan edits it.
- The provider ships as a bundled plugin under `plugins/zcode-provider/`. Paseo core gains no provider manifest entry, no provider factory, and no protocol-layer provider id.
- Provider protocol version stays `1`. `PROVIDER_PROTOCOL_VERSION` in `packages/plugin/src/server/provider.ts` is not changed.
- `paseo-plugin.json` reads exactly `{ "id": "zcode-provider", "requirements": { "paseo": ">=0.14.0-beta.2" } }`. The floor is the release that carries the Task 1 launch-resolution fix.
- The legacy transport declares exactly these capabilities: `prompt.message`, `prompt.command`, `prompt.image`, `permission`, `permission.tool_policy`, `session.configure`, `session.list`, `session.persistence`, `session.archive`, `session.unarchive`, `session.can_stop_runtime`, `session.subsession`, `timeline.plugin`. It does not declare `session.revert.conversation`, `session.revert.files`, `session.revert.both`, `prompt.steer`, or `prompt.output_schema`.
- Binary resolution order, first match wins: the user's `agents.providers.zcode.command` override, then `zcode` on PATH, then the runtime bundled in the ZCode desktop app under `resources/glm/`, then the `GLM_BINARY_PATH` environment variable.
- When the bundled JavaScript runtime is used, the child process runs as `{ command: process.execPath, args: [bundlePath, "app-server", "--stdio"], env: { ELECTRON_RUN_AS_NODE: "1" } }`.
- ZCode frames are `{ id, method, params? }` for requests, `{ id, result }` for responses, `{ id, error: { code, message, data? } }` for failures, and `{ eventId, sessionId, turnId?, seq, timestamp, type, payload? }` for events.
- Every frame decoder is tolerant. Unknown fields are dropped, unknown event types are ignored, and a structurally broken frame is counted in the diagnostic rather than ending the connection.
- ZCode delivers legacy events by cursor, not by push. Each session polls `session/events` with an `afterSeq` cursor.
- ZCode owns its own authentication. No test carries an auth check, an environment gate, or a conditional skip.
- Tests use the two categories in `docs/testing.md` and nothing else: unit tests with injected ports, and real end-to-end tests. No `vi.mock`, no JSDOM, no component mounting.
- Use the npm scripts for tooling. `npm run lint`, `npm run format:files -- <paths>`, `npm run typecheck`. Never call `oxlint`, `oxfmt` or `eslint` directly.
- Run only the test file you changed: `npx vitest run <file> --bail=1`. Never run a workspace-wide suite.
- The stop boundary in Task 7 is a hard requirement, not a refinement. A cancel that never settled must block new runs.

## Review Focus

Five input classes the spec implies that no task's happy path exercises. Each one has a test in the task that owns the code.

1. **A ZCode event carrying a field the provider does not know.** ZCode's schemas are strict and keep gaining fields, and one dropped field must not drop a whole turn. Expected: the event still reaches the timeline. Tested in Task 3.
2. **Stop pressed twice.** The second press must retry the cancel that is already in flight, not open a second one that kills the next turn. Expected: exactly one cancellation reaches ZCode. Tested in Task 7.
3. **A prompt sent while a cancel is still settling.** Expected: refused, with the refusal reported, and the run does not start. Tested in Task 7.
4. **The ZCode binary missing.** Expected: the provider is unavailable with a diagnostic naming all four resolution paths, and the daemon does not crash. Tested in Task 2.
5. **A permission option whose answer carries `permissionUpdates`.** Expected: the rule change reaches ZCode intact, so "always allow" survives the round trip. Tested in Task 8.

---

### Task 1: Core — launch resolution honors a user override and a plugin-supplied command

The plugin needs to own binary discovery, and the daemon needs to keep owning the environment. Today neither is possible: `resolveLaunch` returns `undefined` before it reads the user's override, and it throws before the plugin's `status` can run.

**Files:**

- Modify: `packages/plugin/src/server/provider.ts:29-43` — add the `resolveCommand` hook to `ProviderRegistration`
- Modify: `packages/server/src/server/agent/plugin-provider.ts:157-170` — one launch resolution order
- Test: `packages/server/src/server/agent/plugin-provider.test.ts`

**Interfaces:**

- Consumes: nothing from other tasks.
- Produces: `ProviderRegistration.resolveCommand?(): Promise<readonly [string, ...string[]] | undefined>` (SDK type), and a `resolveLaunch` that returns a complete `ProviderLaunch` whenever a command was resolved by any of the three sources.

- [ ] **Step 1: Write the failing test**

Append to `packages/server/src/server/agent/plugin-provider.test.ts`:

```ts
describe("plugin launch resolution", () => {
  function registrationWith(
    resolveCommand: () => Promise<readonly [string, ...string[]] | undefined>,
  ): ProviderRegistration {
    return {
      id: "launch-resolution-probe",
      label: "Launch Resolution Probe",
      async resolveCommand() {
        return resolveCommand();
      },
      async status({ launch }) {
        return { available: true, diagnostic: launch ? launch.command : "no-launch" };
      },
      async connect() {
        throw new Error("not used");
      },
    };
  }

  test("a user command override wins over the plugin's own resolution", async () => {
    const logger = createTestLogger();
    const plugins = new PluginAgentClientRegistry(logger);
    plugins.replace([registrationWith(async () => ["/plugin/owned/zcode"])]);
    buildProviderRegistry(logger, {
      pluginProviders: plugins.definitions(),
      providerOverrides: {
        "launch-resolution-probe": {
          command: { mode: "replace", argv: ["/user/chosen/zcode"] },
        },
      },
    });
    const client = plugins.clients()["launch-resolution-probe"]!;
    await expect(client.isAvailable()).resolves.toBe(true);
  });

  test("the plugin's own resolution supplies the command when the user set none", async () => {
    const logger = createTestLogger();
    const plugins = new PluginAgentClientRegistry(logger);
    plugins.replace([registrationWith(async () => ["/plugin/owned/zcode"])]);
    buildProviderRegistry(logger, { pluginProviders: plugins.definitions() });
    const client = plugins.clients()["launch-resolution-probe"]!;
    await expect(client.isAvailable()).resolves.toBe(true);
  });

  test("a missing user override target fails hard", async () => {
    const logger = createTestLogger();
    const plugins = new PluginAgentClientRegistry(logger);
    plugins.replace([registrationWith(async () => ["/plugin/owned/zcode"])]);
    buildProviderRegistry(logger, {
      pluginProviders: plugins.definitions(),
      providerOverrides: {
        "launch-resolution-probe": {
          command: { mode: "replace", argv: ["/definitely/missing/zcode"] },
        },
      },
    });
    const client = plugins.clients()["launch-resolution-probe"]!;
    await expect(client.isAvailable()).resolves.toBe(false);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run packages/server/src/server/agent/plugin-provider.test.ts --bail=1`
Expected: FAIL. The `resolveCommand` property does not exist on `ProviderRegistration`, so the test file fails to typecheck or the override is ignored.

- [ ] **Step 3: Add the hook to the SDK type**

In `packages/plugin/src/server/provider.ts`, extend `ProviderRegistration`:

```ts
export interface ProviderRegistration {
  /** Activate only while agents.providers contains this provider ID, independently of enabled. */
  requiresConfiguration?: boolean;
  /** Equal keys share discovery within this provider. Include effective configuration and execution environment. */
  getCatalogCacheKey?(options: ProviderCatalogOptions): Promise<string | undefined>;
  /** Default executable and arguments; the daemon resolves overrides before status/connect. */
  command?: readonly [string, ...string[]];
  /**
   * Resolve the default executable when the plugin owns binary discovery.
   * A user override in agents.providers.<id>.command still wins over this.
   */
  resolveCommand?(): Promise<readonly [string, ...string[]] | undefined>;
  status?(request: ProviderStatusRequest): Promise<ProviderStatus>;
  id: string;
  label: string;
  description?: string;
  /** Plugin-directory-relative path to a self-contained SVG file. */
  icon?: string;
  connect(request: ProviderConnectRequest): Promise<ProviderConnection>;
}
```

- [ ] **Step 4: Rewrite launch resolution**

In `packages/server/src/server/agent/plugin-provider.ts`, replace the body of `resolveLaunch` so it reads:

```ts
private async resolveLaunch(): Promise<ProviderLaunch | undefined> {
  const env = createProviderEnv({ runtimeSettings: this.runtimeSettings });
  const override = this.runtimeSettings?.command;
  if (override) {
    const launch = await resolveProviderLaunch({ commandConfig: override });
    launch.env = env;
    const availability = await checkProviderLaunchAvailable(launch);
    if (!availability.resolvedPath)
      throw new Error(`${launch.command} not found on PATH`);
    return { command: resolvePath(availability.resolvedPath), args: launch.args, env };
  }
  const command = this.registration.command ?? (await this.registration.resolveCommand?.());
  if (!command) return undefined;
  const launch = await resolveProviderLaunch({ defaultBinary: command[0] });
  launch.args.unshift(...command.slice(1));
  launch.env = env;
  const availability = await checkProviderLaunchAvailable(launch);
  if (!availability.resolvedPath) throw new Error(`${launch.command} not found on PATH`);
  return { command: resolvePath(availability.resolvedPath), args: launch.args, env };
}
```

`resolveProviderLaunch` throws when `defaultBinary` is undefined and the mode is not `replace`. The override branch passes only `commandConfig`, so it requires the override to be a `replace`. That is the documented shape of `agents.providers.<id>.command`.

- [ ] **Step 5: Run the test to verify it passes**

Run: `npx vitest run packages/server/src/server/agent/plugin-provider.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 6: Typecheck and lint**

Run: `npm run typecheck && npm run lint`
Expected: clean.

- [ ] **Step 7: Commit**

```bash
git add packages/plugin/src/server/provider.ts packages/server/src/server/agent/plugin-provider.ts packages/server/src/server/agent/plugin-provider.test.ts
git commit -m "feat(plugins): let a plugin resolve its own default executable

A plugin that owns binary discovery could not use it. Launch resolution
returned early when the plugin declared no command, so the user's
agents.providers.<id>.command override was never read, and it threw on a
missing default binary before the plugin's status callback could run.

Add an optional resolveCommand hook and one resolution order: a user
override wins, then the plugin's declared command, then the plugin's own
resolution. The daemon-owned environment reaches the launch in every
case."
```

This commit is its own pull request. It merges before the plugin work starts, so no later commit carries a core diff.

---

### Task 2: Plugin skeleton, binary discovery, and registration

**Files:**

- Create: `plugins/zcode-provider/paseo-plugin.json`
- Create: `plugins/zcode-provider/index.server.ts`
- Create: `plugins/zcode-provider/icon.svg`
- Create: `plugins/zcode-provider/server/provider.ts`
- Create: `plugins/zcode-provider/server/internal/binary.ts`
- Test: `plugins/zcode-provider/server/internal/binary.test.ts`

**Interfaces:**

- Consumes: `ProviderRegistration.resolveCommand?` from Task 1.
- Produces:
  - `resolveZCodeCommand(): Promise<readonly [string, ...string[]] | undefined>` — the discovery result, or `undefined` when nothing was found.
  - `describeZCodeBinary(command): string` — a one-line diagnostic naming the source.
  - `ZCODE_NOT_FOUND_DIAGNOSTIC: string` — the diagnostic listing all four resolution paths.

- [ ] **Step 1: Write the failing test**

Create `plugins/zcode-provider/server/internal/binary.test.ts`:

```ts
import { mkdtemp, mkdir, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { describe, expect, test } from "vitest";
import { ZCODE_NOT_FOUND_DIAGNOSTIC, resolveZCodeCommand } from "./binary.js";

describe("ZCode binary discovery", () => {
  test("reports every resolution path when nothing is found", async () => {
    const empty = await mkdtemp(join(tmpdir(), "zcode-bin-"));
    const result = await resolveZCodeCommand({
      pathEntries: [empty],
      env: {},
      desktopRoots: [join(empty, "no-such-app")],
    });
    expect(result).toBeUndefined();
    for (const hint of ["agents.providers.zcode.command", "PATH", "desktop", "GLM_BINARY_PATH"]) {
      expect(ZCODE_NOT_FOUND_DIAGNOSTIC).toContain(hint);
    }
  });

  test("prefers the desktop bundle over the environment variable", async () => {
    const root = await mkdtemp(join(tmpdir(), "zcode-bin-"));
    const bundleDir = join(root, "ZCode.app", "Contents", "Resources", "glm");
    await mkdir(bundleDir, { recursive: true });
    await writeFile(join(bundleDir, "zcode.cjs"), "// bundle\n");
    const fakeNode = join(root, "node");
    await writeFile(fakeNode, "#!/bin/sh\n");
    const result = await resolveZCodeCommand({
      pathEntries: [],
      env: { GLM_BINARY_PATH: join(root, "ignored") },
      desktopRoots: [join(root, "ZCode.app")],
      execPath: fakeNode,
    });
    expect(result?.[1]).toContain("zcode.cjs");
  });

  test("uses an executable named zcode found on PATH", async () => {
    const root = await mkdtemp(join(tmpdir(), "zcode-bin-"));
    const binary = join(root, "zcode");
    await writeFile(binary, "#!/bin/sh\n");
    const result = await resolveZCodeCommand({
      pathEntries: [root],
      env: {},
      desktopRoots: [],
    });
    expect(result?.[0]).toBe(binary);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/binary.test.ts --bail=1`
Expected: FAIL with "Cannot find module './binary.js'".

- [ ] **Step 3: Write the discovery module**

Create `plugins/zcode-provider/server/internal/binary.ts`:

```ts
import { access, constants, readdir } from "node:fs/promises";
import { join } from "node:path";

export const ZCODE_NOT_FOUND_DIAGNOSTIC = [
  "ZCode was not found.",
  "Install the ZCode CLI (@zcode/cli) so `zcode` is on PATH, or set",
  "agents.providers.zcode.command, or install the ZCode desktop app so its",
  "bundled runtime is discovered, or set GLM_BINARY_PATH.",
].join(" ");

export interface ZCodeDiscoveryOptions {
  /** PATH entries to search for a `zcode` executable. */
  pathEntries: readonly string[];
  /** Environment to consult, including GLM_BINARY_PATH. */
  env: Readonly<Record<string, string | undefined>>;
  /** Roots that may contain a ZCode desktop app bundle. */
  desktopRoots: readonly string[];
  /** Runtime used to execute a bundled JavaScript entry. Defaults to this process. */
  execPath?: string;
}

async function isExecutable(path: string): Promise<boolean> {
  try {
    await access(path, constants.X_OK);
    return true;
  } catch {
    return false;
  }
}

async function isFile(path: string): Promise<boolean> {
  try {
    await access(path, constants.F_OK);
    return true;
  } catch {
    return false;
  }
}

async function findOnPath(
  name: string,
  pathEntries: readonly string[],
): Promise<string | undefined> {
  for (const entry of pathEntries) {
    const candidate = join(entry, name);
    if (await isExecutable(candidate)) return candidate;
  }
  return undefined;
}

async function findDesktopBundle(
  desktopRoots: readonly string[],
): Promise<{ bundle: string; native: boolean } | undefined> {
  for (const root of desktopRoots) {
    const glmDir = join(root, "Contents", "Resources", "glm");
    const native = join(glmDir, "zcode-agent");
    if (await isExecutable(native)) return { bundle: native, native: true };
    const script = join(glmDir, "zcode.cjs");
    if (await isFile(script)) return { bundle: script, native: false };
  }
  return undefined;
}

export async function resolveZCodeCommand(
  options: ZCodeDiscoveryOptions,
): Promise<readonly [string, ...string[]] | undefined> {
  const onPath = await findOnPath("zcode", options.pathEntries);
  if (onPath) return [onPath, "app-server", "--stdio"];

  const bundled = await findDesktopBundle(options.desktopRoots);
  if (bundled) {
    if (bundled.native) return [bundled.bundle, "app-server", "--stdio"];
    return [options.execPath ?? process.execPath, bundled.bundle, "app-server", "--stdio"];
  }

  const fromEnv = options.env.GLM_BINARY_PATH?.trim();
  if (fromEnv && (await isExecutable(fromEnv))) return [fromEnv, "app-server", "--stdio"];

  return undefined;
}

export function defaultDesktopRoots(): readonly string[] {
  if (process.platform === "darwin") {
    return ["/Applications", join(process.env.HOME ?? "", "Applications")];
  }
  if (process.platform === "win32") {
    const localAppData = process.env.LOCALAPPDATA;
    return localAppData ? [join(localAppData, "Programs")] : [];
  }
  return ["/opt", "/usr/share"];
}

export async function listDesktopAppRoots(roots: readonly string[]): Promise<string[]> {
  const found: string[] = [];
  for (const root of roots) {
    let entries: string[];
    try {
      entries = await readdir(root);
    } catch {
      continue;
    }
    for (const entry of entries) {
      if (entry.toLowerCase().startsWith("zcode")) found.push(join(root, entry));
    }
  }
  return found;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run plugins/zcode-provider/server/internal/binary.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 5: Write the manifest, icon, and registration**

Create `plugins/zcode-provider/paseo-plugin.json`:

```json
{ "id": "zcode-provider", "requirements": { "paseo": ">=0.14.0-beta.2" } }
```

Create `plugins/zcode-provider/icon.svg` as a single-color, `currentColor` SVG with `viewBox="0 0 24 24"` and no external references.

Create `plugins/zcode-provider/server/capabilities.ts`:

```ts
import type { ProviderCapability } from "@getpaseo/plugin/server/provider";

/**
 * The capabilities the legacy ZCode transport can deliver.
 * The V4 transport adds session.revert.* and prompt.steer.
 */
export const ZCODE_CAPABILITIES = [
  "prompt.message",
  "prompt.command",
  "prompt.image",
  "permission",
  "permission.tool_policy",
  "session.configure",
  "session.list",
  "session.persistence",
  "session.archive",
  "session.unarchive",
  "session.can_stop_runtime",
  "session.subsession",
  "timeline.plugin",
] as const satisfies readonly ProviderCapability[];
```

The capabilities live in their own module because the transport imports them and the transport is imported by the provider. Keeping them in `provider.ts` would make the two modules import each other.

Create `plugins/zcode-provider/index.server.ts`:

```ts
import { registerProvider } from "@getpaseo/plugin/server";
import { createZCodeProvider } from "./server/provider.js";

export default registerProvider(createZCodeProvider());
```

Create `plugins/zcode-provider/server/provider.ts`:

```ts
import {
  negotiateProviderCapabilities,
  type ProviderConnection,
  type ProviderRegistration,
  type ProviderStatus,
} from "@getpaseo/plugin/server/provider";
import { ZCODE_CAPABILITIES } from "./capabilities.js";
import {
  ZCODE_NOT_FOUND_DIAGNOSTIC,
  defaultDesktopRoots,
  listDesktopAppRoots,
  resolveZCodeCommand,
} from "./internal/binary.js";

export function createZCodeProvider(): ProviderRegistration {
  return {
    id: "zcode",
    label: "ZCode",
    description: "ZCode agent on your host, driven over its app-server protocol",
    icon: "icon.svg",
    async resolveCommand() {
      return resolveZCodeCommand({
        pathEntries: (process.env.PATH ?? "").split(process.platform === "win32" ? ";" : ":"),
        env: process.env,
        desktopRoots: await listDesktopAppRoots(defaultDesktopRoots()),
      });
    },
    async status({ launch }): Promise<ProviderStatus> {
      if (!launch) {
        return { available: false, diagnostic: ZCODE_NOT_FOUND_DIAGNOSTIC };
      }
      return { available: true, diagnostic: `ZCode resolved to ${launch.command}` };
    },
    async connect(request) {
      const negotiated = negotiateProviderCapabilities(request.capabilities, ZCODE_CAPABILITIES);
      return createConnection(negotiated);
    },
  };
}
```

`createConnection` is added in Task 9. Until then, stub it:

```ts
function createConnection(_capabilities: readonly string[]): ProviderConnection {
  throw new Error("ZCode connection is not wired yet");
}
```

- [ ] **Step 6: Run typecheck and lint**

Run: `npm run typecheck && npm run lint`
Expected: clean.

- [ ] **Step 7: Commit**

```bash
git add plugins/zcode-provider
git commit -m "feat(plugins): add the ZCode provider skeleton and binary discovery

Resolve the ZCode executable from PATH, from the runtime bundled in the
ZCode desktop app, or from GLM_BINARY_PATH, and report every path in the
diagnostic when none of them resolve."
```

---

### Task 3: Tolerant frame decoding

ZCode's schemas are strict and keep gaining fields. A decoder that mirrors that strictness drops whole turns when ZCode adds one.

**Files:**

- Create: `plugins/zcode-provider/server/internal/wire.ts`
- Test: `plugins/zcode-provider/server/internal/wire.test.ts`

**Interfaces:**

- Consumes: nothing from other tasks.
- Produces:
  - `decodeFrame(line: string): ZCodeFrame | undefined`
  - `encodeRequest(id: string, method: string, params?: unknown): string`
  - `ZCodeError` — an error carrying a numeric protocol code.

- [ ] **Step 1: Write the failing test**

Create `plugins/zcode-provider/server/internal/wire.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { decodeFrame, encodeRequest } from "./wire.js";

describe("ZCode frame decoding", () => {
  test("keeps an event that carries an unknown field", () => {
    const frame = decodeFrame(
      JSON.stringify({
        eventId: "e1",
        sessionId: "s1",
        seq: 3,
        timestamp: 1_700_000_000_000,
        type: "message.upserted",
        payload: { content: "hello" },
        futureField: { anything: true },
      }),
    );
    expect(frame).toMatchObject({ type: "message.upserted", sessionId: "s1", seq: 3 });
  });

  test("ignores an event type it does not know", () => {
    const frame = decodeFrame(
      JSON.stringify({
        eventId: "e2",
        sessionId: "s1",
        seq: 4,
        timestamp: 1_700_000_000_000,
        type: "some.future.event",
        payload: {},
      }),
    );
    expect(frame).toBeUndefined();
  });

  test("returns undefined for a broken frame instead of throwing", () => {
    expect(decodeFrame("{not json")).toBeUndefined();
    expect(decodeFrame(JSON.stringify({ id: "1" }))).toBeUndefined();
  });

  test("decodes a response and a protocol error", () => {
    expect(decodeFrame(JSON.stringify({ id: "1", result: { ok: true } }))).toMatchObject({
      kind: "response",
      id: "1",
    });
    const failure = decodeFrame(
      JSON.stringify({ id: "2", error: { code: -32000, message: "boom" } }),
    );
    expect(failure).toMatchObject({ kind: "error", id: "2" });
  });

  test("encodes a request without a jsonrpc field", () => {
    const line = encodeRequest("7", "session/send", { sessionId: "s1", content: "hi" });
    expect(JSON.parse(line)).toEqual({
      id: "7",
      method: "session/send",
      params: { sessionId: "s1", content: "hi" },
    });
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/wire.test.ts --bail=1`
Expected: FAIL with "Cannot find module './wire.js'".

- [ ] **Step 3: Write the frame module**

Create `plugins/zcode-provider/server/internal/wire.ts`:

```ts
import { z } from "zod";

const eventFrameSchema = z
  .object({
    eventId: z.string().min(1),
    sessionId: z.string().min(1),
    turnId: z.string().min(1).optional(),
    seq: z.number().int().nonnegative(),
    timestamp: z.number(),
    type: z.string().min(1),
    payload: z.unknown().optional(),
  })
  .passthrough();

const responseFrameSchema = z
  .object({
    id: z.union([z.string(), z.number()]),
    result: z.unknown().optional(),
    error: z
      .object({ code: z.number().int(), message: z.string(), data: z.unknown().optional() })
      .optional(),
  })
  .passthrough();

export type ZCodeEventFrame = z.infer<typeof eventFrameSchema>;

export type ZCodeFrame =
  | { kind: "event"; event: ZCodeEventFrame }
  | { kind: "response"; id: string; result: unknown }
  | { kind: "error"; id: string; code: number; message: string; data?: unknown };

export class ZCodeError extends Error {
  constructor(
    message: string,
    readonly code = "ZCODE_ERROR",
    readonly protocolCode?: number,
  ) {
    super(message);
    this.name = "ZCodeError";
  }
}

export function decodeFrame(line: string): ZCodeFrame | undefined {
  let parsed: unknown;
  try {
    parsed = JSON.parse(line);
  } catch {
    return undefined;
  }
  const event = eventFrameSchema.safeParse(parsed);
  if (event.success) return { kind: "event", event: event.data };
  const response = responseFrameSchema.safeParse(parsed);
  if (!response.success) return undefined;
  if (response.data.error) {
    return {
      kind: "error",
      id: String(response.data.id),
      code: response.data.error.code,
      message: response.data.error.message,
      data: response.data.error.data,
    };
  }
  return { kind: "response", id: String(response.data.id), result: response.data.result };
}

export function encodeRequest(id: string, method: string, params?: unknown): string {
  const frame: Record<string, unknown> = { id, method };
  if (params !== undefined) frame.params = params;
  return `${JSON.stringify(frame)}\n`;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run plugins/zcode-provider/server/internal/wire.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add plugins/zcode-provider/server/internal/wire.ts plugins/zcode-provider/server/internal/wire.test.ts
git commit -m "feat(plugins): decode ZCode frames without mirroring their strictness"
```

---

### Task 4: JSON-RPC client over a writable stream

**Files:**

- Create: `plugins/zcode-provider/server/internal/rpc.ts`
- Test: `plugins/zcode-provider/server/internal/rpc.test.ts`

**Interfaces:**

- Consumes: `decodeFrame`, `encodeRequest`, `ZCodeError` from Task 3.
- Produces:
  - `class ZCodeRpcClient` with `call(method, params): Promise<unknown>`, `onEvent(listener): () => void`, `write(line: string): void`, `close(): void`.
  - `ZCodeRpcClient.isMethodNotFound(error): boolean`.

- [ ] **Step 1: Write the failing test**

Create `plugins/zcode-provider/server/internal/rpc.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { ZCodeRpcClient, isMethodNotFound } from "./rpc.js";

describe("ZCode RPC client", () => {
  test("resolves a call when the matching response arrives", async () => {
    const client = new ZCodeRpcClient();
    const pending = client.call("session/read", { sessionId: "s1" });
    client.receive(JSON.stringify({ id: "1", result: { ok: true } }));
    await expect(pending).resolves.toEqual({ ok: true });
  });

  test("rejects a call when the protocol answers with an error", async () => {
    const client = new ZCodeRpcClient();
    const pending = client.call("session/read", { sessionId: "s1" });
    client.receive(JSON.stringify({ id: "1", error: { code: -32601, message: "no method" } }));
    await expect(pending).rejects.toThrow("no method");
  });

  test("recognizes a method-not-found failure", async () => {
    const client = new ZCodeRpcClient();
    const pending = client.call("session/steer", { sessionId: "s1" });
    client.receive(JSON.stringify({ id: "1", error: { code: -32601, message: "no method" } }));
    const error = await pending.catch((value: unknown) => value);
    expect(isMethodNotFound(error)).toBe(true);
  });

  test("dispatches events to listeners and survives a broken frame", () => {
    const client = new ZCodeRpcClient();
    const seen: string[] = [];
    client.onEvent((event) => seen.push(event.type));
    client.receive("{broken");
    client.receive(
      JSON.stringify({
        eventId: "e1",
        sessionId: "s1",
        seq: 1,
        timestamp: 1,
        type: "turn.started",
      }),
    );
    expect(seen).toEqual(["turn.started"]);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/rpc.test.ts --bail=1`
Expected: FAIL with "Cannot find module './rpc.js'".

- [ ] **Step 3: Write the client**

Create `plugins/zcode-provider/server/internal/rpc.ts`:

```ts
import { decodeFrame, encodeRequest, type ZCodeEventFrame } from "./wire.js";

interface PendingCall {
  resolve(value: unknown): void;
  reject(error: unknown): void;
}

export class ZCodeMethodNotFoundError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "ZCodeMethodNotFoundError";
  }
}

export function isMethodNotFound(error: unknown): boolean {
  return error instanceof ZCodeMethodNotFoundError;
}

export class ZCodeRpcClient {
  private nextId = 1;
  private readonly pending = new Map<string, PendingCall>();
  private readonly listeners = new Set<(event: ZCodeEventFrame) => void>();
  private closed = false;

  call(method: string, params?: unknown): Promise<unknown> {
    if (this.closed) return Promise.reject(new Error("ZCode connection is closed"));
    const id = String(this.nextId++);
    return new Promise<unknown>((resolve, reject) => {
      this.pending.set(id, { resolve, reject });
      this.write(encodeRequest(id, method, params));
    });
  }

  onEvent(listener: (event: ZCodeEventFrame) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  receive(line: string): void {
    const frame = decodeFrame(line);
    if (!frame) return;
    if (frame.kind === "event") {
      for (const listener of this.listeners) listener(frame.event);
      return;
    }
    const pending = this.pending.get(frame.id);
    if (!pending) return;
    this.pending.delete(frame.id);
    if (frame.kind === "error") {
      pending.reject(
        frame.code === -32601
          ? new ZCodeMethodNotFoundError(frame.message)
          : Object.assign(new Error(frame.message), { protocolCode: frame.code }),
      );
      return;
    }
    pending.resolve(frame.result);
  }

  close(): void {
    this.closed = true;
    for (const [, pending] of this.pending) {
      pending.reject(new Error("ZCode connection closed before the response arrived"));
    }
    this.pending.clear();
    this.listeners.clear();
  }

  /** Overridden by the process layer to write a framed line to the child. */
  protected write(_line: string): void {
    throw new Error("ZCode RPC client has no output stream");
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run plugins/zcode-provider/server/internal/rpc.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add plugins/zcode-provider/server/internal/rpc.ts plugins/zcode-provider/server/internal/rpc.test.ts
git commit -m "feat(plugins): add the ZCode JSON-RPC client"
```

---

### Task 5: Process lifecycle

**Files:**

- Create: `plugins/zcode-provider/server/internal/process.ts`
- Create: `plugins/zcode-provider/test/fake-zcode-app-server.mjs`
- Test: `plugins/zcode-provider/server/internal/process.test.ts`

**Interfaces:**

- Consumes: `ZCodeRpcClient` from Task 4.
- Produces:
  - `startZCodeProcess(options): ZCodeProcess` where `ZCodeProcess` exposes `rpc: ZCodeRpcClient`, `stderr(): string`, `exit(): Promise<void>`.
  - `ZCodeProcess.startupError` — set when the child failed to start.

- [ ] **Step 1: Write the fake app-server**

Create `plugins/zcode-provider/test/fake-zcode-app-server.mjs`:

```js
import { createInterface } from "node:readline";

const lines = createInterface({ input: process.stdin });
const mode = process.env.FAKE_ZCODE_MODE ?? "ok";

for await (const line of lines) {
  if (!line.trim()) continue;
  const request = JSON.parse(line);
  if (mode === "exit") {
    process.exit(3);
  }
  if (mode === "stderr") {
    process.stderr.write("ZCode runtime missing\n");
    process.exit(1);
  }
  const id = request.id;
  if (request.method === "session/events") {
    process.stdout.write(`${JSON.stringify({ id, result: { events: [] } })}\n`);
    continue;
  }
  if (request.method === "session/send") {
    process.stdout.write(
      `${JSON.stringify({ id, result: { sessionId: request.params.sessionId, accepted: true, stateRevision: 1 } })}\n`,
    );
    continue;
  }
  process.stdout.write(`${JSON.stringify({ id, result: {} })}\n`);
}
```

- [ ] **Step 2: Write the failing test**

Create `plugins/zcode-provider/server/internal/process.test.ts`:

```ts
import { fileURLToPath } from "node:url";
import { describe, expect, test } from "vitest";
import { startZCodeProcess } from "./process.js";

const fakeServer = fileURLToPath(new URL("../../test/fake-zcode-app-server.mjs", import.meta.url));

describe("ZCode process lifecycle", () => {
  test("answers a call over the child's stdio", async () => {
    const process = startZCodeProcess({
      command: process.execPath,
      args: [fakeServer],
      env: { FAKE_ZCODE_MODE: "ok" },
    });
    await expect(process.rpc.call("session/read", { sessionId: "s1" })).resolves.toEqual({});
    await process.exit();
  });

  test("reports a startup failure with the child's stderr", async () => {
    const process = startZCodeProcess({
      command: process.execPath,
      args: [fakeServer],
      env: { FAKE_ZCODE_MODE: "stderr" },
    });
    await expect(process.ready).rejects.toThrow(/runtime missing/);
    await process.exit();
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/process.test.ts --bail=1`
Expected: FAIL with "Cannot find module './process.js'".

- [ ] **Step 4: Write the process module**

Create `plugins/zcode-provider/server/internal/process.ts`:

```ts
import { spawnProcess } from "@getpaseo/plugin/server";
import { createInterface } from "node:readline";
import { ZCodeRpcClient } from "./rpc.js";

export interface ZCodeProcessOptions {
  command: string;
  args: readonly string[];
  env: Readonly<Record<string, string>>;
  cwd?: string;
}

export interface ZCodeProcess {
  rpc: ZCodeRpcClient;
  /** Resolves once the child answered its first request. Rejects on startup failure. */
  ready: Promise<void>;
  stderr(): string;
  exit(): Promise<void>;
}

class ProcessRpcClient extends ZCodeRpcClient {
  constructor(private readonly writeLine: (line: string) => void) {
    super();
  }

  protected override write(line: string): void {
    this.writeLine(line);
  }
}

export function startZCodeProcess(options: ZCodeProcessOptions): ZCodeProcess {
  const child = spawnProcess(options.command, [...options.args], {
    cwd: options.cwd,
    env: { ...options.env },
    stdio: "pipe",
    detached: process.platform !== "win32",
  });

  let stderrText = "";
  child.stderr?.setEncoding("utf8");
  child.stderr?.on("data", (chunk: string) => {
    stderrText += chunk;
  });

  const rpc = new ProcessRpcClient((line) => {
    child.stdin?.write(line);
  });

  const reader = createInterface({ input: child.stdout });
  reader.on("line", (line) => {
    rpc.receive(line);
  });

  let resolveReady: () => void;
  let rejectReady: (error: Error) => void;
  const ready = new Promise<void>((resolve, reject) => {
    resolveReady = resolve;
    rejectReady = reject;
  });
  ready.catch(() => undefined);

  child.on("exit", (code) => {
    const error = new Error(
      `ZCode exited with code ${code ?? "unknown"}.${stderrText ? ` ${stderrText.trim()}` : ""}`,
    );
    rejectReady(error);
    rpc.close();
  });
  child.on("error", (error) => {
    rejectReady(error instanceof Error ? error : new Error(String(error)));
  });

  // The child is ready once it answers a request.
  void rpc
    .call("runtime/capabilities", {})
    .then(() => resolveReady())
    .catch((error) => rejectReady(error instanceof Error ? error : new Error(String(error))));

  return {
    rpc,
    ready,
    stderr: () => stderrText,
    async exit() {
      rpc.close();
      child.stdin?.end();
      child.kill();
    },
  };
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `npx vitest run plugins/zcode-provider/server/internal/process.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add plugins/zcode-provider/server/internal/process.ts plugins/zcode-provider/server/internal/process.test.ts plugins/zcode-provider/test/fake-zcode-app-server.mjs
git commit -m "feat(plugins): run the ZCode app-server as a child process"
```

---

### Task 6: Legacy transport and the event cursor

**Files:**

- Create: `plugins/zcode-provider/server/internal/transport.ts`
- Create: `plugins/zcode-provider/server/internal/legacy-transport.ts`
- Test: `plugins/zcode-provider/server/internal/legacy-transport.test.ts`

**Interfaces:**

- Consumes: `ZCodeProcess` from Task 5, `ZCodeRpcClient` from Task 4.
- Produces:
  - `interface ZCodeTransport` with `capabilities`, `createSession`, `resumeSession`, `send`, `stop`, `listSessions`, `setModel`, `setMode`, `setThoughtLevel`, `onEvent`, `close`.
  - `createLegacyTransport(process): ZCodeTransport`.

- [ ] **Step 1: Write the failing test**

Create `plugins/zcode-provider/server/internal/legacy-transport.test.ts`:

```ts
import { fileURLToPath } from "node:url";
import { describe, expect, test } from "vitest";
import { startZCodeProcess } from "./process.js";
import { createLegacyTransport } from "./legacy-transport.js";

const fakeServer = fileURLToPath(new URL("../../test/fake-zcode-app-server.mjs", import.meta.url));

describe("legacy ZCode transport", () => {
  test("sends a prompt and acknowledges it", async () => {
    const process = startZCodeProcess({
      command: process.execPath,
      args: [fakeServer],
      env: { FAKE_ZCODE_MODE: "ok" },
    });
    const transport = createLegacyTransport(process);
    await transport.createSession({
      workspacePath: "/tmp/project",
      workspaceKey: "/tmp/project",
    });
    await expect(
      transport.send({ sessionId: "s1", content: "hello", inputId: "input-1" }),
    ).resolves.toBeUndefined();
    await transport.close();
  });

  test("does not advertise rewind or steer", () => {
    const process = startZCodeProcess({
      command: process.execPath,
      args: [fakeServer],
      env: { FAKE_ZCODE_MODE: "ok" },
    });
    const transport = createLegacyTransport(process);
    expect(transport.capabilities).not.toContain("session.revert.conversation");
    expect(transport.capabilities).not.toContain("prompt.steer");
    expect(transport.capabilities).toContain("prompt.message");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/legacy-transport.test.ts --bail=1`
Expected: FAIL with "Cannot find module './transport.js'".

- [ ] **Step 3: Write the transport interface**

Create `plugins/zcode-provider/server/internal/transport.ts`:

```ts
export interface ZCodeWorkspaceRef {
  workspacePath: string;
  workspaceKey: string;
}

export interface ZCodeSessionHandle {
  sessionId: string;
  workspace: ZCodeWorkspaceRef;
}

export interface ZCodeTransport {
  readonly capabilities: readonly string[];
  createSession(input: {
    workspace: ZCodeWorkspaceRef;
    mode?: string;
    model?: string;
    thoughtLevel?: string;
  }): Promise<ZCodeSessionHandle>;
  resumeSession(input: {
    sessionId: string;
    workspace?: ZCodeWorkspaceRef;
  }): Promise<ZCodeSessionHandle>;
  send(input: { sessionId: string; content: string; inputId: string }): Promise<void>;
  stop(input: { sessionId: string }): Promise<void>;
  listSessions(input: { workspace?: ZCodeWorkspaceRef; limit?: number }): Promise<unknown[]>;
  setModel(input: { sessionId: string; model: string }): Promise<void>;
  setMode(input: { sessionId: string; mode: string }): Promise<void>;
  setThoughtLevel(input: { sessionId: string; thoughtLevel: string }): Promise<void>;
  onEvent(listener: (event: ZCodeEvent) => void): () => void;
  close(): Promise<void>;
}

export type ZCodeEvent = {
  type: string;
  sessionId: string;
  turnId?: string;
  seq: number;
  payload: unknown;
};
```

- [ ] **Step 4: Write the legacy transport**

Create `plugins/zcode-provider/server/internal/legacy-transport.ts`:

```ts
import { ZCODE_CAPABILITIES } from "../capabilities.js";
import type { ZCodeProcess } from "./process.js";
import type {
  ZCodeEvent,
  ZCodeSessionHandle,
  ZCodeTransport,
  ZCodeWorkspaceRef,
} from "./transport.js";

export const LEGACY_POLL_INTERVAL_MS = 200;

export function createLegacyTransport(process: ZCodeProcess): ZCodeTransport {
  const cursors = new Map<string, number>();
  const listeners = new Set<(event: ZCodeEvent) => void>();
  let timer: NodeJS.Timeout | undefined;
  let stopped = false;

  async function poll(): Promise<void> {
    for (const [sessionId, cursor] of cursors) {
      try {
        const result = (await process.rpc.call("session/events", {
          sessionId,
          afterSeq: cursor,
        })) as { events?: unknown[] };
        for (const raw of result?.events ?? []) {
          const event = raw as {
            type?: string;
            sessionId?: string;
            turnId?: string;
            seq?: number;
            payload?: unknown;
          };
          if (!event.type || !event.sessionId || typeof event.seq !== "number") continue;
          cursors.set(sessionId, event.seq);
          const mapped: ZCodeEvent = {
            type: event.type,
            sessionId: event.sessionId,
            turnId: event.turnId,
            seq: event.seq,
            payload: event.payload,
          };
          for (const listener of listeners) listener(mapped);
        }
      } catch {
        // A failed poll is retried on the next tick; the session's own
        // error path reports the failure if the process is gone.
      }
    }
  }

  function startPolling(): void {
    if (timer || stopped) return;
    timer = setInterval(() => {
      void poll();
    }, LEGACY_POLL_INTERVAL_MS);
    timer.unref?.();
  }

  return {
    capabilities: ZCODE_CAPABILITIES,
    async createSession(input) {
      const result = (await process.rpc.call("session/create", {
        workspace: {
          workspacePath: input.workspace.workspacePath,
          workspaceKey: input.workspace.workspaceKey,
        },
        ...(input.mode ? { mode: input.mode } : {}),
        ...(input.model ? { model: input.model } : {}),
        ...(input.thoughtLevel ? { thoughtLevel: input.thoughtLevel } : {}),
      })) as { sessionId?: string };
      if (!result?.sessionId) throw new Error("ZCode did not return a session id");
      cursors.set(result.sessionId, 0);
      startPolling();
      return { sessionId: result.sessionId, workspace: input.workspace };
    },
    async resumeSession(input) {
      const workspace = input.workspace ?? {
        workspacePath: input.sessionId,
        workspaceKey: input.sessionId,
      };
      await process.rpc.call("session/resume", {
        sessionId: input.sessionId,
        workspace,
      });
      cursors.set(input.sessionId, 0);
      startPolling();
      return { sessionId: input.sessionId, workspace };
    },
    async send(input) {
      await process.rpc.call("session/send", {
        sessionId: input.sessionId,
        content: input.content,
        inputId: input.inputId,
      });
    },
    async stop(input) {
      await process.rpc.call("session/stop", { sessionId: input.sessionId });
    },
    async listSessions(input) {
      const result = (await process.rpc.call("session/list", {
        ...(input.workspace ? { workspace: input.workspace } : {}),
        ...(input.limit ? { limit: input.limit } : {}),
      })) as { sessions?: unknown[] };
      return result?.sessions ?? [];
    },
    async setModel(input) {
      await process.rpc.call("session/setModel", {
        sessionId: input.sessionId,
        model: input.model,
      });
    },
    async setMode(input) {
      await process.rpc.call("session/setMode", {
        sessionId: input.sessionId,
        mode: input.mode,
      });
    },
    async setThoughtLevel(input) {
      await process.rpc.call("session/setThoughtLevel", {
        sessionId: input.sessionId,
        thoughtLevel: input.thoughtLevel,
      });
    },
    onEvent(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
    async close() {
      stopped = true;
      if (timer) clearInterval(timer);
      cursors.clear();
      listeners.clear();
    },
  };
}
```

**Method-not-found downgrade.** ZCode's protocol answers a removed method with `-32601`. That is the only failure that may be downgraded: the capability goes away and the rest of the provider keeps working. Any other failure propagates.

Add to the legacy transport:

```ts
const unavailable = new Set<string>();

async function call(method: string, params?: unknown): Promise<unknown> {
  try {
    return await process.rpc.call(method, params);
  } catch (error) {
    if (isMethodNotFound(error)) {
      unavailable.add(method);
      return undefined;
    }
    throw error;
  }
}
```

Replace every `process.rpc.call(...)` in the transport with `call(...)`, and expose the unavailable set so the connection can drop the matching capability:

```ts
// In ZCodeTransport:
readonly unavailableMethods: ReadonlySet<string>;
```

The connection removes a capability when its method is unavailable. The mapping lives next to the transport:

```ts
const CAPABILITY_METHODS: Record<string, string> = {
  "prompt.command": "session/command",
  "session.subsession": "session/subagents",
  "session.can_stop_runtime": "session/read",
};
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `npx vitest run plugins/zcode-provider/server/internal/legacy-transport.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add plugins/zcode-provider/server/internal/transport.ts plugins/zcode-provider/server/internal/legacy-transport.ts plugins/zcode-provider/server/internal/legacy-transport.test.ts
git commit -m "feat(plugins): add the legacy ZCode transport and its event cursor"
```

---

### Task 7: Turn state machine and the stop boundary

This task owns the rule that a cancel which has not settled blocks new runs.

**Files:**

- Create: `plugins/zcode-provider/server/internal/session.ts`
- Test: `plugins/zcode-provider/server/internal/session.test.ts`

**Interfaces:**

- Consumes: `ZCodeTransport`, `ZCodeEvent` from Task 6.
- Produces:
  - `class ZCodeSession` with `open`, `prompt`, `interrupt`, `configure`, `close`, and an `onEvent` hook.
  - `type TurnPhase = "idle" | "running" | "stopping" | "settled"`.

- [ ] **Step 1: Write the failing test**

Create `plugins/zcode-provider/server/internal/session.test.ts`:

```ts
import { describe, expect, test, vi } from "vitest";
import { ZCodeSession } from "./session.js";
import type { ZCodeEvent, ZCodeTransport } from "./transport.js";

function fakeTransport() {
  const sent: Array<{ method: string; params: unknown }> = [];
  const listeners = new Set<(event: ZCodeEvent) => void>();
  const transport: ZCodeTransport = {
    capabilities: ["prompt.message", "permission"],
    createSession: async () => ({
      sessionId: "zcode-1",
      workspace: { workspacePath: "/w", workspaceKey: "/w" },
    }),
    resumeSession: async () => ({
      sessionId: "zcode-1",
      workspace: { workspacePath: "/w", workspaceKey: "/w" },
    }),
    send: async (input) => {
      sent.push({ method: "send", params: input });
    },
    stop: async (input) => {
      sent.push({ method: "stop", params: input });
    },
    listSessions: async () => [],
    setModel: async () => undefined,
    setMode: async () => undefined,
    setThoughtLevel: async () => undefined,
    onEvent: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
    close: async () => undefined,
  };
  return {
    transport,
    sent,
    emit(event: ZCodeEvent) {
      for (const listener of listeners) listener(event);
    },
  };
}

describe("ZCode session turn state", () => {
  test("acknowledges a prompt with the turn id it generated", async () => {
    const harness = fakeTransport();
    const events: unknown[] = [];
    const session = new ZCodeSession({
      id: "paseo-1",
      transport: harness.transport,
      emit: (event) => events.push(event),
    });
    await session.open({ cwd: "/w" });
    await session.prompt({ clientMessageId: "m1", text: "hello" });
    expect(events).toContainEqual(
      expect.objectContaining({
        type: "session.prompt_result",
        result: expect.objectContaining({ type: "turn" }),
      }),
    );
    expect(harness.sent[0]?.params).toMatchObject({ inputId: expect.any(String) });
  });

  test("a second interrupt retries the cancel instead of opening a second one", async () => {
    const harness = fakeTransport();
    const session = new ZCodeSession({
      id: "paseo-1",
      transport: harness.transport,
      emit: () => undefined,
    });
    await session.open({ cwd: "/w" });
    await session.prompt({ clientMessageId: "m1", text: "hello" });
    const first = session.interrupt();
    const second = session.interrupt();
    await Promise.all([first, second]);
    expect(harness.sent.filter((entry) => entry.method === "stop")).toHaveLength(1);
  });

  test("refuses a new prompt while a cancel is unsettled", async () => {
    const harness = fakeTransport();
    const session = new ZCodeSession({
      id: "paseo-1",
      transport: harness.transport,
      emit: () => undefined,
    });
    await session.open({ cwd: "/w" });
    await session.prompt({ clientMessageId: "m1", text: "hello" });
    void session.interrupt();
    await expect(session.prompt({ clientMessageId: "m2", text: "again" })).rejects.toThrow(
      /cancel/i,
    );
  });

  test("a failed cancel keeps refusing new prompts", async () => {
    const harness = fakeTransport();
    harness.transport.stop = async () => {
      throw new Error("stop failed");
    };
    const session = new ZCodeSession({
      id: "paseo-1",
      transport: harness.transport,
      emit: () => undefined,
    });
    await session.open({ cwd: "/w" });
    await session.prompt({ clientMessageId: "m1", text: "hello" });
    await expect(session.interrupt()).rejects.toThrow("stop failed");
    await expect(session.prompt({ clientMessageId: "m2", text: "again" })).rejects.toThrow(
      /cancel/i,
    );
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/session.test.ts --bail=1`
Expected: FAIL with "Cannot find module './session.js'".

- [ ] **Step 3: Write the session**

Create `plugins/zcode-provider/server/internal/session.ts`:

```ts
import type { ProviderEvent, ProviderPrompt } from "@getpaseo/plugin/server/provider";
import type { ZCodeEvent, ZCodeTransport } from "./transport.js";

export type TurnPhase = "idle" | "running" | "stopping" | "settled";

interface ActiveTurn {
  turnId: string;
  inputId: string;
  clientMessageId: string;
}

export interface ZCodeSessionOptions {
  id: string;
  transport: ZCodeTransport;
  emit(event: ProviderEvent): void;
}

export class ZCodeSession {
  private phase: TurnPhase = "idle";
  private active: ActiveTurn | undefined;
  private currentTurnId = "zcode-turn-0";
  private nativeSessionId = "";
  private cancelInFlight: Promise<void> | undefined;
  private readonly unsubscribe: () => void;

  constructor(private readonly options: ZCodeSessionOptions) {
    this.unsubscribe = options.transport.onEvent((event) => this.handleEvent(event));
  }

  async open(input: { cwd: string }): Promise<string> {
    const handle = await this.options.transport.createSession({
      workspace: { workspacePath: input.cwd, workspaceKey: input.cwd },
    });
    this.nativeSessionId = handle.sessionId;
    this.options.emit({
      type: "session.opened",
      sessionId: this.options.id,
      capabilities: this.options.transport.capabilities,
      restoration: "core",
      cwd: input.cwd,
      persistence: { version: 1, data: { nativeSessionId: handle.sessionId } },
    });
    this.options.emit({ type: "session.ready", sessionId: this.options.id });
    return handle.sessionId;
  }

  async prompt(prompt: ProviderPrompt): Promise<void> {
    if (this.cancelInFlight) {
      throw new Error("A cancel is still settling; wait for it before starting a new run");
    }
    if (this.phase === "stopping") {
      throw new Error("A cancel is still settling; wait for it before starting a new run");
    }
    if (prompt.delivery === "steer") {
      throw new Error("Steering is not supported on the legacy ZCode transport");
    }
    if (prompt.input.type === "command") {
      throw new Error("Slash commands are not supported on the legacy ZCode transport");
    }
    const text = prompt.input.content
      .filter((part): part is { type: "text"; text: string } => part.type === "text")
      .map((part) => part.text)
      .join("\n");
    const turnId = `zcode-turn-${this.options.id}-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
    const inputId = `paseo-input-${prompt.clientMessageId}`;
    this.phase = "running";
    this.currentTurnId = turnId;
    this.active = { turnId, inputId, clientMessageId: prompt.clientMessageId };
    await this.options.transport.send({
      sessionId: this.nativeSessionId,
      content: text,
      inputId,
    });
    this.options.emit({
      type: "session.prompt_result",
      sessionId: this.options.id,
      clientMessageId: prompt.clientMessageId,
      result: { type: "turn", turnId },
    });
  }

  interrupt(): Promise<void> {
    if (this.cancelInFlight) return this.cancelInFlight;
    this.phase = "stopping";
    this.cancelInFlight = this.options.transport
      .stop({ sessionId: this.nativeSessionId })
      .then(() => {
        this.cancelInFlight = undefined;
      })
      .catch((error) => {
        this.cancelInFlight = undefined;
        this.phase = "settled";
        this.options.emit({
          type: "session.turn",
          sessionId: this.options.id,
          turnId: this.currentTurnId,
          state: "failed",
          error: { message: error instanceof Error ? error.message : String(error) },
        });
        throw error;
      });
    return this.cancelInFlight;
  }

  private handleEvent(event: ZCodeEvent): void {
    if (event.type === "turn.started") {
      const payload = event.payload as { inputId?: string } | undefined;
      if (payload?.inputId && payload.inputId === this.active?.inputId) {
        this.options.emit({
          type: "session.turn",
          sessionId: this.options.id,
          turnId: this.currentTurnId,
          state: "started",
        });
      }
      return;
    }
    if (event.type === "turn.completed" || event.type === "turn.failed") {
      if (this.phase === "stopping") return;
      this.phase = "idle";
      this.options.emit({
        type: "session.turn",
        sessionId: this.options.id,
        turnId: this.currentTurnId,
        state: event.type === "turn.completed" ? "completed" : "failed",
      });
      this.active = undefined;
    }
  }

  async close(): Promise<void> {
    this.unsubscribe();
    // The transport is shared by every session on this connection, so the
    // connection closes it, not the session.
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run plugins/zcode-provider/server/internal/session.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add plugins/zcode-provider/server/internal/session.ts plugins/zcode-provider/server/internal/session.test.ts
git commit -m "feat(plugins): own the ZCode stop boundary in the session

A cancel that has not settled must not be followed by a new run, a
second interrupt must retry the cancel already in flight rather than
open a second one, and a cancel that failed must keep refusing new
runs."
```

---

### Task 8: Timeline and permission mapping

**Files:**

- Create: `plugins/zcode-provider/server/internal/timeline.ts`
- Create: `plugins/zcode-provider/server/internal/signals.ts`
- Test: `plugins/zcode-provider/server/internal/timeline.test.ts`
- Test: `plugins/zcode-provider/server/internal/signals.test.ts`

**Interfaces:**

- Consumes: `ZCodeEvent` from Task 6.
- Produces:
  - `mapEventToTimelineItems(event: ZCodeEvent): ProviderTimelineItem[]`
  - `mapPermissionRequest(payload: unknown): ProviderPermissionRequest`
  - `permissionResponseFor(request: ProviderPermissionRequest, response: ProviderPermissionResponse): Record<string, JsonValue>`

- [ ] **Step 1: Write the failing tests**

Create `plugins/zcode-provider/server/internal/timeline.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { mapEventToTimelineItems } from "./timeline.js";

describe("ZCode timeline mapping", () => {
  test("maps an assistant message", () => {
    const items = mapEventToTimelineItems({
      type: "message.upserted",
      sessionId: "s1",
      seq: 1,
      payload: { messageId: "m1", content: "hello", role: "assistant" },
    });
    expect(items).toEqual([
      { id: "m1", type: "assistant_message", text: "hello", messageId: "m1" },
    ]);
  });

  test("maps a reasoning delta", () => {
    const items = mapEventToTimelineItems({
      type: "part.delta",
      sessionId: "s1",
      seq: 2,
      payload: { messageId: "m2", partId: "p2", field: "reasoning", delta: "thinking" },
    });
    expect(items[0]).toMatchObject({ type: "reasoning", text: "thinking" });
  });

  test("maps a tool call", () => {
    const items = mapEventToTimelineItems({
      type: "tool.updated",
      sessionId: "s1",
      seq: 3,
      payload: { toolCallId: "t1", toolName: "read_file", kind: "read", input: { path: "/a" } },
    });
    expect(items[0]).toMatchObject({ type: "tool_call", callId: "t1", name: "read_file" });
  });

  test("returns nothing for an event it does not map", () => {
    expect(
      mapEventToTimelineItems({
        type: "session.updated",
        sessionId: "s1",
        seq: 4,
        payload: {},
      }),
    ).toEqual([]);
  });
});
```

Create `plugins/zcode-provider/server/internal/signals.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { mapPermissionRequest, permissionResponseFor } from "./signals.js";

describe("ZCode permission mapping", () => {
  const payload = {
    requestId: "p1",
    sessionId: "s1",
    toolCallId: "t1",
    toolName: "bash",
    reason: "run a command",
    riskLevel: "medium",
    input: { command: "ls" },
    options: [
      {
        optionId: "allow-once",
        kind: "allow",
        name: "Allow once",
        response: { decision: "allow" },
      },
      {
        optionId: "always",
        kind: "allow",
        name: "Always allow",
        response: {
          decision: "allow",
          permissionUpdates: { addRules: [{ behavior: "allow", toolName: "bash" }] },
        },
      },
      {
        optionId: "deny",
        kind: "deny",
        name: "Deny",
        response: { decision: "deny", reason: "not allowed" },
      },
    ],
  };

  test("maps the options to permission actions", () => {
    const request = mapPermissionRequest(payload);
    expect(request.id).toBe("p1");
    expect(request.name).toBe("bash");
    expect(request.kind).toBe("tool");
    expect(request.actions?.map((action) => action.id)).toEqual(["allow-once", "always", "deny"]);
    expect(request.actions?.map((action) => action.behavior)).toEqual(["allow", "allow", "deny"]);
  });

  test("returns the selected option's ZCode response unchanged", () => {
    const request = mapPermissionRequest(payload);
    expect(
      permissionResponseFor(request, { behavior: "allow", selectedActionId: "always" }),
    ).toEqual({
      decision: "allow",
      permissionUpdates: { addRules: [{ behavior: "allow", toolName: "bash" }] },
    });
  });

  test("rejects an unknown action id", () => {
    const request = mapPermissionRequest(payload);
    expect(() =>
      permissionResponseFor(request, { behavior: "allow", selectedActionId: "nope" }),
    ).toThrow(/nope/);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run plugins/zcode-provider/server/internal/timeline.test.ts plugins/zcode-provider/server/internal/signals.test.ts --bail=1`
Expected: FAIL with "Cannot find module".

- [ ] **Step 3: Write the timeline mapper**

Create `plugins/zcode-provider/server/internal/timeline.ts`:

```ts
import type { ProviderTimelineItem } from "@getpaseo/plugin/server/provider";
import type { ZCodeEvent } from "./transport.js";

interface EventPayload {
  messageId?: string;
  partId?: string;
  field?: string;
  delta?: string;
  content?: string;
  role?: string;
  toolCallId?: string;
  toolName?: string;
  kind?: string;
  input?: unknown;
  status?: string;
}

export function mapEventToTimelineItems(event: ZCodeEvent): ProviderTimelineItem[] {
  const payload = (event.payload ?? {}) as EventPayload;
  if (event.type === "message.upserted") {
    const id = payload.messageId ?? `${event.sessionId}-${event.seq}`;
    if (payload.role === "user") {
      return [
        { id, type: "user_message", text: payload.content ?? "", messageId: payload.messageId },
      ];
    }
    return [
      {
        id,
        type: "assistant_message",
        text: payload.content ?? "",
        messageId: payload.messageId,
      },
    ];
  }
  if (event.type === "part.delta") {
    const id = payload.partId ?? payload.messageId ?? `${event.sessionId}-${event.seq}`;
    const text = payload.delta ?? "";
    if (payload.field === "reasoning") return [{ id, type: "reasoning", text }];
    return [{ id, type: "assistant_message", text }];
  }
  if (event.type === "tool.updated") {
    const callId = payload.toolCallId ?? `${event.sessionId}-${event.seq}`;
    const status =
      payload.status === "failed"
        ? "failed"
        : payload.status === "canceled"
          ? "canceled"
          : "running";
    return [
      {
        id: callId,
        type: "tool_call",
        callId,
        name: payload.toolName ?? "tool",
        status,
        error: status === "failed" ? { message: "tool failed" } : null,
        detail: { type: "shell", command: describeToolInput(payload.input) },
      },
    ];
  }
  return [];
}

function describeToolInput(input: unknown): string {
  if (input && typeof input === "object") {
    const record = input as Record<string, unknown>;
    for (const key of ["command", "filePath", "path", "query"]) {
      const value = record[key];
      if (typeof value === "string") return value;
    }
  }
  return "";
}
```

- [ ] **Step 4: Write the permission mapper**

Create `plugins/zcode-provider/server/internal/signals.ts`:

```ts
import type { JsonValue } from "@getpaseo/protocol/agent-types";
import type {
  ProviderPermissionRequest,
  ProviderPermissionResponse,
} from "@getpaseo/plugin/server/provider";

interface ZCodePermissionOption {
  optionId: string;
  kind: string;
  name: string;
  description?: string;
  response: Record<string, JsonValue>;
}

const ZCODE_OPTIONS_METADATA_KEY = "zcodePermissionOptions";

export function mapPermissionRequest(payload: unknown): ProviderPermissionRequest {
  const request = payload as {
    requestId: string;
    sessionId: string;
    toolCallId: string;
    toolName: string;
    reason: string;
    riskLevel: string;
    input: unknown;
    options: ZCodePermissionOption[];
  };
  return {
    id: request.requestId,
    name: request.toolName,
    kind: "tool",
    title: request.toolName,
    description: request.reason,
    input:
      request.input && typeof request.input === "object"
        ? (request.input as Record<string, JsonValue>)
        : undefined,
    actions: request.options.map((option) => ({
      id: option.optionId,
      label: option.name,
      behavior: option.response.decision === "allow" ? "allow" : "deny",
    })),
    metadata: {
      [ZCODE_OPTIONS_METADATA_KEY]: request.options as unknown as JsonValue,
    },
  };
}

export function permissionResponseFor(
  request: ProviderPermissionRequest,
  response: ProviderPermissionResponse,
): Record<string, JsonValue> {
  const actionId = "selectedActionId" in response ? response.selectedActionId : undefined;
  if (!actionId) {
    throw new Error("The permission response must name the action the user chose");
  }
  const options = request.metadata?.[ZCODE_OPTIONS_METADATA_KEY];
  const list = Array.isArray(options) ? (options as unknown as ZCodePermissionOption[]) : [];
  const chosen = list.find((option) => option.optionId === actionId);
  if (!chosen) throw new Error(`Unknown permission action: ${actionId}`);
  return chosen.response;
}
```

The raw ZCode options travel in `metadata`, which is a `Record<string, JsonValue>` on the request. That keeps the SDK type intact and means the selected action's own `response` reaches ZCode unchanged, including any `permissionUpdates` that make "always allow" stick.

`answerPermission` is added to `ZCodeTransport` in this task:

```ts
// In ZCodeTransport:
answerPermission(input: { sessionId: string; requestId: string; response: unknown }): Promise<void>;
```

And to the legacy transport:

```ts
async answerPermission(input) {
  await process.rpc.call("interaction/respond", {
    sessionId: input.sessionId,
    requestId: input.requestId,
    ...(input.response as Record<string, unknown>),
  });
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `npx vitest run plugins/zcode-provider/server/internal/timeline.test.ts plugins/zcode-provider/server/internal/signals.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add plugins/zcode-provider/server/internal/timeline.ts plugins/zcode-provider/server/internal/timeline.test.ts plugins/zcode-provider/server/internal/signals.ts plugins/zcode-provider/server/internal/signals.test.ts plugins/zcode-provider/server/internal/transport.ts plugins/zcode-provider/server/internal/legacy-transport.ts
git commit -m "feat(plugins): map ZCode events to timeline items and permissions"
```

---

### Task 9: Wire the provider connection

**Files:**

- Modify: `plugins/zcode-provider/server/provider.ts` — replace the stub with a real connection
- Create: `plugins/zcode-provider/server/internal/catalog.ts`
- Test: `plugins/zcode-provider/server/internal/catalog.test.ts`
- Test: `plugins/zcode-provider/server/provider.test.ts`

**Interfaces:**

- Consumes: everything from Tasks 2, 6, 7, 8.
- Produces: `createZCodeProvider()` returning a working `ProviderRegistration`.

- [ ] **Step 1: Write the failing catalog test**

Create `plugins/zcode-provider/server/internal/catalog.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { mapCatalog } from "./catalog.js";

describe("ZCode catalog mapping", () => {
  test("maps models and modes from the session snapshot", () => {
    const catalog = mapCatalog({
      configOptions: {
        model: {
          current: "glm-5",
          options: [
            { id: "glm-5", name: "GLM 5" },
            { id: "glm-4", name: "GLM 4" },
          ],
        },
        mode: {
          current: "plan",
          options: [
            { id: "plan", name: "Plan" },
            { id: "build", name: "Build" },
          ],
        },
      },
    });
    expect(catalog.models.map((model) => model.id)).toEqual(["glm-5", "glm-4"]);
    expect(catalog.modes.map((mode) => mode.id)).toEqual(["plan", "build"]);
  });

  test("returns an empty catalog for a snapshot without options", () => {
    expect(mapCatalog({}).models).toEqual([]);
    expect(mapCatalog({}).modes).toEqual([]);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/internal/catalog.test.ts --bail=1`
Expected: FAIL with "Cannot find module './catalog.js'".

- [ ] **Step 3: Write the catalog mapper**

Create `plugins/zcode-provider/server/internal/catalog.ts`:

```ts
import type { ProviderCatalog } from "@getpaseo/plugin/server/provider";

interface OptionGroup {
  current?: string;
  options?: Array<{ id: string; name?: string; label?: string }>;
}

interface Snapshot {
  configOptions?: {
    model?: OptionGroup;
    mode?: OptionGroup;
    thoughtLevel?: OptionGroup;
  };
}

export function mapCatalog(snapshot: Snapshot): ProviderCatalog {
  const options = snapshot.configOptions;
  const models = (options?.model?.options ?? []).map((option) => ({
    id: option.id,
    label: option.name ?? option.label ?? option.id,
  }));
  const modes = (options?.mode?.options ?? []).map((option) => ({
    id: option.id,
    label: option.name ?? option.label ?? option.id,
  }));
  return {
    models,
    modes,
    thinkingOptions: (options?.thoughtLevel?.options ?? []).map((option) => ({
      id: option.id,
      label: option.name ?? option.label ?? option.id,
    })),
    defaultModel: options?.model?.current,
    defaultMode: options?.mode?.current,
    defaultThinkingOption: options?.thoughtLevel?.current,
  };
}
```

- [ ] **Step 4: Write the connection**

Replace the stub `createConnection` in `plugins/zcode-provider/server/provider.ts` with this. It follows the `plugins/antigravity-provider/server/provider.ts` shape: a session map, a listener set, a serialized send queue, and a `dispatch` switch over `ProviderInput`.

```ts
function createConnection(
  launch: ProviderLaunch,
  negotiated: readonly string[],
): ProviderConnection {
  const sessions = new Map<string, ZCodeSession>();
  const listeners = new Set<(event: ProviderEvent) => void>();
  let closed = false;
  let pending = Promise.resolve();
  let transport: ZCodeTransport | undefined;

  function emit(event: ProviderEvent): void {
    if (closed) return;
    for (const listener of listeners) listener(event);
  }

  function session(id: string): ZCodeSession {
    const found = sessions.get(id);
    if (!found) throw new ZCodeError(`Unknown ZCode session: ${id}`);
    return found;
  }

  /**
   * Paseo and ZCode name sessions differently. The persistence handle carries
   * the ZCode id, and the archive and unarchive inputs carry only that handle,
   * so the connection indexes both directions.
   */
  const nativeIndex = new Map<string, string>();

  function paseoSessionForNative(nativeId: string): ZCodeSession {
    const paseoId = nativeIndex.get(nativeId);
    if (!paseoId) throw new ZCodeError(`Unknown ZCode session: ${nativeId}`);
    return session(paseoId);
  }

  function ensureTransport(): ZCodeTransport {
    if (!transport) {
      transport = createLegacyTransport(
        startZCodeProcess({ command: launch.command, args: launch.args, env: launch.env }),
      );
    }
    return transport;
  }

  async function dispatch(input: ProviderInput): Promise<void> {
    switch (input.type) {
      case "catalog": {
        const snapshot = (await ensureTransport().rpc.call("session/read", {})) as unknown;
        emit({ type: "catalog", requestId: input.requestId, catalog: mapCatalog(snapshot) });
        return;
      }
      case "sessions": {
        const list = await ensureTransport().listSessions({
          ...(input.cwd
            ? { workspace: { workspacePath: input.cwd, workspaceKey: input.cwd } }
            : {}),
          ...(input.limit ? { limit: input.limit } : {}),
        });
        emit({
          type: "sessions",
          requestId: input.requestId,
          sessions: list.map((entry) => toSessionSummary(entry)),
        });
        return;
      }
      case "session.open": {
        if (sessions.has(input.sessionId)) {
          throw new ZCodeError(`Session already exists: ${input.sessionId}`);
        }
        const opened = new ZCodeSession({
          id: input.sessionId,
          transport: ensureTransport(),
          emit,
        });
        sessions.set(input.sessionId, opened);
        try {
          const native = input.persistence
            ? await opened.resume({
                cwd: input.config.cwd,
                nativeSessionId: nativeSessionIdOf(input.persistence),
                replay: input.history === "replay",
              })
            : await opened.open({ cwd: input.config.cwd });
          nativeIndex.set(native, input.sessionId);
        } catch (error) {
          sessions.delete(input.sessionId);
          throw error;
        }
        return;
      }
      case "session.prompt": {
        await session(input.sessionId).prompt(input.prompt);
        return;
      }
      case "session.configure": {
        await session(input.sessionId).configure(input.changes);
        return;
      }
      case "session.interrupt": {
        await session(input.sessionId).interrupt();
        return;
      }
      case "session.permission": {
        await session(input.sessionId).answerPermission(input.permissionId, input.response);
        return;
      }
      case "session.archive": {
        const nativeId = nativeSessionIdOf(input.persistence);
        await paseoSessionForNative(nativeId).archive();
        emit({
          type: "session.persistence",
          sessionId: nativeId,
          persistence: input.persistence,
        });
        return;
      }
      case "session.unarchive": {
        const nativeId = nativeSessionIdOf(input.persistence);
        await paseoSessionForNative(nativeId).unarchive(input.persistence);
        return;
      }
      case "session.can_stop_runtime": {
        emit({
          type: "session.can_stop_runtime",
          requestId: input.requestId,
          sessionId: input.sessionId,
          canStop: true,
        });
        return;
      }
      case "session.close": {
        await session(input.sessionId).close();
        sessions.delete(input.sessionId);
        emit({ type: "session.closed", sessionId: input.sessionId });
        return;
      }
      default:
        throw new ZCodeError(`Unsupported ZCode operation: ${input.type}`);
    }
  }

  function failed(input: ProviderInput, error: unknown): void {
    const message = error instanceof Error ? error.message : String(error);
    const code = error instanceof ZCodeError ? error.code : "ZCODE_ERROR";
    if (input.type === "session.prompt") {
      emit({
        type: "session.prompt_result",
        sessionId: input.sessionId,
        clientMessageId: input.prompt.clientMessageId,
        result: { type: "failed", error: { message, code } },
      });
      return;
    }
    if ("requestId" in input) {
      emit({ type: "request.failed", requestId: input.requestId, error: { message, code } });
    }
  }

  return {
    version: 1,
    capabilities: negotiated,
    async send(input) {
      if (closed) throw new ZCodeError("ZCode connection is closed");
      requireProviderCapabilities(negotiated, input);
      pending = pending
        .then(async () => {
          if (!closed) await dispatch(input);
          return undefined;
        })
        .catch((error) => failed(input, error));
    },
    onEvent(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
    async close() {
      if (closed) return;
      closed = true;
      await pending;
      await Promise.all([...sessions.values()].map((opened) => opened.close()));
      sessions.clear();
      listeners.clear();
      await transport?.close();
      await child?.exit();
    },
  };
}
```

The `connect` implementation becomes:

```ts
async connect(request) {
  if (!request.versions.includes(1)) {
    throw new ZCodeError("Provider protocol version 1 is required");
  }
  if (!request.launch) {
    throw new ZCodeError("The daemon must supply the ZCode launch");
  }
  const negotiated = negotiateProviderCapabilities(request.capabilities, ZCODE_CAPABILITIES);
  return createConnection(request.launch, negotiated);
}
```

`ZCodeSession` grows the three methods the dispatch calls, in `plugins/zcode-provider/server/internal/session.ts`:

```ts
async resume(input: {
  cwd: string;
  nativeSessionId: string;
  replay: boolean;
}): Promise<string> {
  const handle = await this.options.transport.resumeSession({
    sessionId: input.nativeSessionId,
    workspace: { workspacePath: input.cwd, workspaceKey: input.cwd },
  });
  this.nativeSessionId = handle.sessionId;
  this.options.emit({
    type: "session.opened",
    sessionId: this.options.id,
    capabilities: this.options.transport.capabilities,
    restoration: "core",
    cwd: input.cwd,
    persistence: { version: 1, data: { nativeSessionId: handle.sessionId } },
  });
  this.options.emit({ type: "session.ready", sessionId: this.options.id });
  return handle.sessionId;
}

async configure(changes: ProviderConfigChanges): Promise<void> {
  if (changes.model) await this.options.transport.setModel({ sessionId: this.nativeSessionId, model: changes.model });
  if (changes.mode) await this.options.transport.setMode({ sessionId: this.nativeSessionId, mode: changes.mode });
  if (changes.thinkingOption) {
    await this.options.transport.setThoughtLevel({
      sessionId: this.nativeSessionId,
      thoughtLevel: changes.thinkingOption,
    });
  }
}

async answerPermission(permissionId: string, response: ProviderPermissionResponse): Promise<void> {
  const request = this.permissionRequests.get(permissionId);
  if (!request) throw new ZCodeError(`Unknown permission request: ${permissionId}`);
  const zcodeResponse = permissionResponseFor(request, response);
  await this.options.transport.answerPermission({
    sessionId: this.nativeSessionId,
    requestId: permissionId,
    response: zcodeResponse,
  });
  this.permissionRequests.delete(permissionId);
  this.options.emit({ type: "session.permission_resolved", sessionId: this.options.id, permissionId });
}

async archive(): Promise<void> {
  await this.options.transport.archive({ sessionId: this.nativeSessionId });
}

async unarchive(_persistence: ProviderPersistence): Promise<void> {
  await this.options.transport.resumeSession({ sessionId: this.nativeSessionId });
}
```

`permissionRequests` is a `Map<string, ProviderPermissionRequest>` on the session. The session's event handler fills it when a `permission.requested` event arrives:

```ts
if (event.type === "permission.requested") {
  const request = mapPermissionRequest(event.payload);
  this.permissionRequests.set(request.id, request);
  this.options.emit({ type: "session.permission", sessionId: this.options.id, request });
  return;
}
```

`archive` is added to `ZCodeTransport` and to the legacy transport:

```ts
// In ZCodeTransport:
archive(input: { sessionId: string }): Promise<void>;

// In the legacy transport:
async archive(input) {
  await process.rpc.call("session/close", { sessionId: input.sessionId });
},
```

`toSessionSummary` converts a ZCode session entry into a `ProviderSessionSummary`, which is `{ persistence, cwd, title?, description?, updatedAt? }`:

```ts
function nativeSessionIdOf(persistence: ProviderPersistence): string {
  const data = persistence.data as { nativeSessionId?: string };
  if (!data?.nativeSessionId) {
    throw new ZCodeError("The persistence handle carries no ZCode session id");
  }
  return data.nativeSessionId;
}

function toSessionSummary(entry: unknown): ProviderSessionSummary {
  const record = entry as {
    sessionId?: string;
    title?: string;
    workspacePath?: string;
    updatedAt?: string;
  };
  return {
    persistence: {
      version: 1,
      data: { nativeSessionId: record.sessionId ?? "" },
    },
    cwd: record.workspacePath ?? "",
    ...(record.title ? { title: record.title } : {}),
    ...(record.updatedAt ? { updatedAt: record.updatedAt } : {}),
  };
}
```

**Process death.** When the child exits, every session it hosted reports the failure and closes. Add this to the connection, right after `ensureTransport` is defined:

```ts
function onProcessExit(error: Error): void {
  for (const opened of sessions.values()) {
    emit({
      type: "session.runtime_failed",
      sessionId: opened.id,
      error: { message: error.message, code: "ZCODE_EXITED" },
    });
    emit({ type: "session.closed", sessionId: opened.id, error: { message: error.message } });
  }
  sessions.clear();
}
```

`ZCodeProcess` exposes the failure through its `ready` promise, which rejects when the child exits. The connection awaits it once and calls `onProcessExit`:

```ts
void ensureTransportProcess().ready.catch((error) => {
  onProcessExit(error instanceof Error ? error : new Error(String(error)));
});
```

`ensureTransport` therefore keeps the `ZCodeProcess` next to the transport, not just the transport:

```ts
let child: ZCodeProcess | undefined;

function ensureTransportProcess(): ZCodeProcess {
  if (!child) {
    child = startZCodeProcess({ command: launch.command, args: launch.args, env: launch.env });
    transport = createLegacyTransport(child);
  }
  return child;
}

function ensureTransport(): ZCodeTransport {
  ensureTransportProcess();
  return transport!;
}
```

- [ ] **Step 5: Write the provider test**

Create `plugins/zcode-provider/server/provider.test.ts`:

```ts
import { fileURLToPath } from "node:url";
import { describe, expect, test } from "vitest";
import type { ProviderEvent } from "@getpaseo/plugin/server/provider";
import { createZCodeProvider } from "./provider.js";

const fakeServer = fileURLToPath(new URL("../test/fake-zcode-app-server.mjs", import.meta.url));

describe("ZCode provider", () => {
  test("negotiates capabilities and does not advertise rewind", async () => {
    const provider = createZCodeProvider();
    const connection = await provider.connect({
      launch: {
        command: process.execPath,
        args: [fakeServer],
        env: { FAKE_ZCODE_MODE: "ok" },
      },
      versions: [1],
      capabilities: ["prompt.message", "session.persistence"],
    });
    expect(connection.capabilities).toContain("prompt.message");
    expect(connection.capabilities).not.toContain("session.revert.conversation");
    await connection.close();
  });

  test("opens a session and reports it ready", async () => {
    const provider = createZCodeProvider();
    const connection = await provider.connect({
      launch: {
        command: process.execPath,
        args: [fakeServer],
        env: { FAKE_ZCODE_MODE: "ok" },
      },
      versions: [1],
      capabilities: ["prompt.message"],
    });
    const events: ProviderEvent[] = [];
    connection.onEvent((event) => events.push(event));
    connection.send({
      type: "session.open",
      requestId: "r1",
      sessionId: "p1",
      config: {
        cwd: "/tmp",
        env: {},
        mcpServers: {},
        settings: {},
        persist: false,
      },
      history: "skip",
    });
    await vi.waitFor(() => {
      expect(events.some((event) => event.type === "session.ready")).toBe(true);
    });
    await connection.close();
  });
});
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `npx vitest run plugins/zcode-provider/server/internal/catalog.test.ts plugins/zcode-provider/server/provider.test.ts --bail=1`
Expected: PASS.

- [ ] **Step 7: Typecheck, lint, format**

Run: `npm run typecheck && npm run lint && npm run format:files -- plugins/zcode-provider`
Expected: clean.

- [ ] **Step 8: Commit**

```bash
git add plugins/zcode-provider
git commit -m "feat(plugins): wire the ZCode provider connection"
```

---

### Task 10: End-to-end test against the fake app-server

**Files:**

- Modify: `plugins/zcode-provider/test/fake-zcode-app-server.mjs`
- Create: `plugins/zcode-provider/server/provider.e2e.test.ts`

**Interfaces:**

- Consumes: everything from Tasks 2 through 9.
- Produces: a test that drives a real child process over the real protocol.

- [ ] **Step 1: Extend the fake app-server**

Replace `plugins/zcode-provider/test/fake-zcode-app-server.mjs` with a version that scripts a full turn. `session/create` returns a session id, `session/send` queues three events, and `session/events` returns the queued events the client has not seen yet, advancing by `seq`.

```js
import { createInterface } from "node:readline";

const lines = createInterface({ input: process.stdin });
const mode = process.env.FAKE_ZCODE_MODE ?? "ok";
const queue = [];
let seq = 0;

function push(sessionId, type, payload, extra = {}) {
  seq += 1;
  queue.push({
    eventId: `e${seq}`,
    sessionId,
    turnId: extra.turnId,
    seq,
    timestamp: Date.now(),
    type,
    payload,
  });
}

for await (const line of lines) {
  if (!line.trim()) continue;
  const request = JSON.parse(line);
  const id = request.id;

  if (mode === "exit") {
    process.exit(3);
  }
  if (mode === "stderr") {
    process.stderr.write("ZCode runtime missing\n");
    process.exit(1);
  }

  if (request.method === "session/create") {
    process.stdout.write(`${JSON.stringify({ id, result: { sessionId: "zcode-1" } })}\n`);
    continue;
  }
  if (request.method === "session/send") {
    const sessionId = request.params.sessionId;
    const turnId = "zcode-turn-1";
    push(
      sessionId,
      "turn.started",
      { turnNumber: 1, input: request.params.content, inputId: request.params.inputId },
      { turnId },
    );
    push(
      sessionId,
      "message.upserted",
      { messageId: "m1", role: "assistant", content: "hello from zcode" },
      { turnId },
    );
    push(
      sessionId,
      "turn.completed",
      { response: "hello from zcode", tokenCount: 12, toolCallCount: 0, duration: 5 },
      { turnId },
    );
    process.stdout.write(
      `${JSON.stringify({ id, result: { sessionId, accepted: true, stateRevision: 1 } })}\n`,
    );
    continue;
  }
  if (request.method === "session/events") {
    const afterSeq = request.params.afterSeq ?? 0;
    const events = queue.filter((event) => event.seq > afterSeq);
    process.stdout.write(`${JSON.stringify({ id, result: { events } })}\n`);
    continue;
  }
  if (request.method === "runtime/capabilities") {
    process.stdout.write(`${JSON.stringify({ id, result: { ok: true } })}\n`);
    continue;
  }
  process.stdout.write(`${JSON.stringify({ id, result: {} })}\n`);
}
```

- [ ] **Step 2: Write the failing test**

Create `plugins/zcode-provider/server/provider.e2e.test.ts`:

```ts
import { fileURLToPath } from "node:url";
import { describe, expect, test } from "vitest";
import type { ProviderEvent, ProviderInput } from "@getpaseo/plugin/server/provider";
import { createZCodeProvider } from "./provider.js";

const fakeServer = fileURLToPath(new URL("../test/fake-zcode-app-server.mjs", import.meta.url));

describe("ZCode provider end to end", () => {
  test("opens a session, streams a reply and completes the turn", async () => {
    const provider = createZCodeProvider();
    const connection = await provider.connect({
      launch: {
        command: process.execPath,
        args: [fakeServer],
        env: { FAKE_ZCODE_MODE: "ok" },
      },
      versions: [1],
      capabilities: ["prompt.message", "session.persistence"],
    });
    const events: ProviderEvent[] = [];
    connection.onEvent((event) => events.push(event));

    connection.send({
      type: "session.open",
      requestId: "r1",
      sessionId: "p1",
      config: { cwd: "/tmp", env: {}, mcpServers: {}, settings: {}, persist: false },
      history: "skip",
    } as ProviderInput);
    await vi.waitFor(() => {
      expect(events.some((event) => event.type === "session.ready")).toBe(true);
    });

    connection.send({
      type: "session.prompt",
      sessionId: "p1",
      prompt: {
        clientMessageId: "m1",
        delivery: "auto",
        input: { type: "message", content: [{ type: "text", text: "hello" }] },
      },
    } as ProviderInput);

    await vi.waitFor(() => {
      expect(
        events.some(
          (event) => event.type === "timeline.item" && event.item.type === "assistant_message",
        ),
      ).toBe(true);
    });
    await vi.waitFor(() => {
      expect(
        events.some((event) => event.type === "session.turn" && event.state === "completed"),
      ).toBe(true);
    });

    await connection.close();
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `npx vitest run plugins/zcode-provider/server/provider.e2e.test.ts --bail=1`
Expected: FAIL until the fake app-server scripts the turn and the connection maps the events.

- [ ] **Step 4: Make it pass**

The session's event handler turns a ZCode event into timeline items and publishes them. Add this to `ZCodeSession.handleEvent`, before the turn handling:

```ts
if (
  event.type !== "turn.started" &&
  event.type !== "turn.completed" &&
  event.type !== "turn.failed"
) {
  for (const item of mapEventToTimelineItems(event)) {
    this.options.emit({ type: "timeline.item", sessionId: this.options.id, item });
  }
}
```

`turn.completed` carries `tokenCount`, which becomes usage. Add to the same handler:

```ts
if (event.type === "turn.completed") {
  const payload = event.payload as { tokenCount?: number } | undefined;
  this.options.emit({
    type: "session.usage",
    sessionId: this.options.id,
    turnId: this.currentTurnId,
    usage: { inputTokens: payload?.tokenCount, outputTokens: 0 },
  });
}
```

The `timeline.item` event carries `item: ProviderTimelineItem`, and the assistant message comes from the `message.upserted` event the fake emits. The turn ends when `turn.completed` arrives, which the fake queues at the same `seq` order the transport reads.

- [ ] **Step 5: Run the full plugin test set**

Run: `npx vitest run plugins/zcode-provider --bail=1`
Expected: PASS.

- [ ] **Step 6: Typecheck, lint, format**

Run: `npm run typecheck && npm run lint && npm run format:files -- plugins/zcode-provider`
Expected: clean.

- [ ] **Step 7: Commit**

```bash
git add plugins/zcode-provider
git commit -m "test(plugins): drive the ZCode provider against a real child process"
```

---

## Deferred: the V4 transport

The V4 transport is a separate plan and is deliberately not part of this one. It needs a grounding pass over ZCode's V4 wire format that this plan did not do, and guessing at it would produce a plan full of invented detail.

Its entry criteria are met once this plan lands:

- The `ZCodeTransport` interface in `plugins/zcode-provider/server/internal/transport.ts` is the seam the V4 transport implements. It already exists, so the V4 work does not touch the legacy transport, the session state machine, the timeline mapper, the permission mapper, or the connection wiring.
- The capability list changes in exactly one place. `ZCODE_CAPABILITIES` in `plugins/zcode-provider/server/provider.ts` gains `session.revert.conversation`, `session.revert.files`, `session.revert.both` and `prompt.steer`, and the transport's own `capabilities` becomes the authority so the two transports differ without touching the connection.
- `PASEO_ZCODE_PROTOCOL` selects the transport. It reads `auto`, `legacy` or `v4` from the environment, with `auto` meaning "use the V4 transport when the CLI reports support for it, otherwise the legacy transport".

The V4 plan has to cover: topic subscription and the snapshot-plus-delta read model, the sequence and log-epoch watermarks, gap detection and resync, the CAS command envelope and its `baseRevision`, the six acknowledgement states, `applyFileRewind` and `editUserQuery` for rewind, the queue commands for steer, and the transport selection probe.

Its acceptance gate is the same as the spec states: `auto` resolves to V4 only after the V4 transport passes the real-ZCode local end-to-end smoke test. Until then `auto` resolves to legacy.
