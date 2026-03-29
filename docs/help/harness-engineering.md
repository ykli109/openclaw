---
title: "Harness Engineering"
summary: "How OpenClaw builds and uses test harnesses: principles, patterns, and best practices drawn from the codebase"
read_when:
  - Writing tests for a new module or extension
  - Adding a harness file to a new domain
  - Understanding why tests are structured the way they are
  - Onboarding to the test infrastructure
---

# Harness Engineering

**Harness Engineering** is the practice of deliberately designing the scaffolding around your production code so that it can be driven, observed, and isolated during testing—without changing the production code itself. The "harness" is infrastructure you build once and reuse across many tests: factories, fakes, environment managers, process wrappers, and output captures.

This guide explains the principles behind harness engineering, shows how OpenClaw applies them, and describes the best practices the codebase has converged on.

---

## Why harnesses matter

A test without a harness typically embeds setup logic inline. As modules grow, inline setup becomes copy-pasted across files and breaks silently when internals change. Harnesses invert this: the module owns its test infrastructure just as it owns its production code.

Concretely, a harness:

- Isolates the system under test from real I/O (file system, network, time, processes)
- Provides a stable, typed API for driving the module in tests
- Cleans up after itself so tests do not bleed state into each other
- Hides low-level details (temp-dir creation, mock wiring, env snapshots) behind a small, purposeful surface

---

## File naming convention

OpenClaw uses a consistent suffix convention so harness files are easy to locate and distinguish from production code:

| Suffix | Purpose |
|---|---|
| `*.test-harness.ts` | Unit/integration harness for a single module |
| `*.e2e-harness.ts` | End-to-end harness that spawns real processes |
| `*.mock-harness.ts` | Shared stubs/fakes for a subsystem |
| `test-helpers/` | A directory of supporting utilities for a domain |
| `test-utils/` | Repo-wide test utilities (env, fixtures, types) |

Examples from the codebase:

```
src/cron/service.test-harness.ts
src/auto-reply/reply.test-harness.ts
src/gateway/server-http.test-harness.ts
extensions/discord/src/send.test-harness.ts
test/helpers/gateway-e2e-harness.ts
scripts/e2e/mcp-channels-harness.ts
```

---

## Core principle: dependency injection

The foundational pattern that makes everything else possible is passing dependencies in rather than reaching for them. OpenClaw does this via a `createDefaultDeps()` factory (see `src/cli/deps.ts`):

```typescript
// Production code receives deps as a parameter
export function createDefaultDeps(): CliDeps {
  return {
    whatsapp: createLazySender("whatsapp", () => import("./channels/whatsapp.runtime.js")),
    telegram: createLazySender("telegram", () => import("./channels/telegram.runtime.js")),
    discord:  createLazySender("discord",  () => import("./channels/discord.runtime.js")),
    // ...
  };
}
```

Tests swap in fake implementations:

```typescript
const deps = {
  ...createDefaultDeps(),
  whatsapp: vi.fn().mockResolvedValue({ ok: true }),
};
await runCommand(deps, args);
```

**Why lazy loading matters here:** channels load heavy runtime modules on first use. The factory wraps each channel in a lazy loader with a module-level cache, so tests that never exercise a channel do not pay the import cost. The `createLazySender` wrapper is transparent to callers.

---

## The harness factory pattern

Most harness files export a factory function that creates all the fakes needed for a domain and optionally installs Vitest lifecycle hooks:

```typescript
// src/cron/service.test-harness.ts

export function setupCronServiceSuite(options?: { prefix?: string; baseTimeIso?: string }) {
  const logger = createNoopLogger();
  const { makeStorePath } = createCronStoreHarness({ prefix: options?.prefix });
  installCronTestHooks({ logger, baseTimeIso: options?.baseTimeIso });
  return { logger, makeStorePath };
}
```

A test file calls this once at suite scope and gets back typed handles:

```typescript
const { logger, makeStorePath } = setupCronServiceSuite();

it("runs a job", async () => {
  const { storePath } = await makeStorePath();
  await withCronServiceForTest({ makeStorePath: () => ({ storePath, cleanup: async () => {} }), logger, cronEnabled: true },
    async ({ cron }) => {
      await cron.start();
      // ...
    },
  );
});
```

The lifecycle hooks (`beforeAll`, `afterAll`, `beforeEach`, `afterEach`) live inside the harness, not in the test file. This keeps each test file to three lines of setup even for complex stateful services.

---

## Environment isolation

Tests that touch the file system or depend on `HOME`/`OPENCLAW_STATE_DIR` must never pollute the host environment. OpenClaw provides two levels of isolation:

### Scoped environment helpers (`src/test-utils/env.ts`)

```typescript
// Restore exactly the keys you changed
export function withEnv<T>(env: Record<string, string | undefined>, fn: () => T): T {
  const snapshot = captureEnv(Object.keys(env));
  try {
    applyEnvValues(env);
    return fn();
  } finally {
    snapshot.restore();
  }
}

// Cross-platform home directory override
export function createPathResolutionEnv(homeDir: string): NodeJS.ProcessEnv {
  // Sets HOME, USERPROFILE, HOMEDRIVE, HOMEPATH, OPENCLAW_HOME, OPENCLAW_STATE_DIR
  // Clears OPENCLAW_BUNDLED_PLUGINS_DIR
}
```

### Per-test temp-home harness (`src/auto-reply/reply.test-harness.ts`)

```typescript
export function createTempHomeHarness(options: { prefix: string }) {
  let fixtureRoot = "";
  let caseId = 0;

  beforeAll(async () => {
    fixtureRoot = await fs.mkdtemp(path.join(os.tmpdir(), options.prefix));
  });

  afterAll(async () => {
    await fs.rm(fixtureRoot, { recursive: true, force: true });
  });

  async function withTempHome<T>(fn: (home: string) => Promise<T>): Promise<T> {
    const home = path.join(fixtureRoot, `case-${++caseId}`);
    await fs.mkdir(path.join(home, ".openclaw", "agents", "main", "sessions"), { recursive: true });
    const envSnapshot = snapshotHomeEnv();
    process.env.HOME = home;
    // ...
    try {
      return await fn(home);
    } finally {
      restoreHomeEnv(envSnapshot);
    }
  }

  return { withTempHome };
}
```

The shared `createFixtureSuite` utility (`src/test-utils/fixture-suite.ts`) provides the same numbered-directory pattern for any module that needs isolated file-system paths without needing to replicate the lifecycle logic.

---

## Module mocking via `vi.mock`

When a test cannot inject a fake through function parameters, OpenClaw uses Vitest module mocking. The pattern is:

1. **Hoist** shared test state with `vi.hoisted()` so it is available before imports run.
2. **Mock** the module with `vi.mock()`, referencing the hoisted state.
3. **Configure** per-test behavior in `beforeEach` or per-case setup.

```typescript
// src/gateway/test-helpers.mocks.ts (excerpt)

const hoisted = vi.hoisted(() => ({
  getReplyFromConfig: vi.fn(),
  agentCommand: vi.fn(),
  embeddedRunMock: {
    activeIds: new Set<string>(),
    abortCalls: [] as string[],
    waitResults: new Map<string, unknown>(),
  },
}));

vi.mock("../auto-reply/reply.js", () => ({
  getReplyFromConfig: (...args: unknown[]) => hoisted.getReplyFromConfig(...args),
}));

vi.mock("../agents/pi-embedded.js", async () => {
  return await importEmbeddedRunMockModule(hoisted.embeddedRunMock);
});
```

**Why `vi.hoisted`?** Vitest hoists `vi.mock()` calls to the top of the file during transformation, but the factory closures run later. `vi.hoisted()` creates objects that are initialized before those closures run, so the mock factories can close over stable references instead of `undefined`.

---

## Stub factories for plugin/channel types

Integration tests across the gateway, routing, and auto-reply layers need a realistic `PluginRegistry` without real I/O. OpenClaw defines a `createStubPluginRegistry()` factory in `src/gateway/test-helpers.mocks.ts` that returns in-memory implementations of all built-in channels:

```typescript
const createStubChannelPlugin = (params: StubChannelOptions): ChannelPlugin => ({
  id: params.id,
  meta: { id, label, selectionLabel, docsPath, blurb },
  capabilities: { chatTypes: ["direct"] },
  config: {
    listAccountIds: async () => [],
    resolveAccount: async () => undefined,
    isConfigured: async () => false,
  },
  outbound: createStubOutboundAdapter(params.id),
  // ...
});

const createStubPluginRegistry = (): PluginRegistry => ({
  plugins: [],
  channels: [whatsapp, telegram, discord, slack, signal, imessage, msteams, matrix, /* ... */],
  // ...
});
```

All stubs are typed to the exact production interface, so if a channel gains a new required capability the TypeScript compiler flags every stub that needs updating.

---

## Output capture for CLI commands

CLI commands write to stdout/stderr via an injected `OutputRuntimeEnv`. The `createCliRuntimeCapture()` factory (`src/cli/test-runtime-capture.ts`) wraps all output methods in `vi.fn()` and accumulates output into plain arrays that tests can assert against:

```typescript
export function createCliRuntimeCapture(): CliRuntimeCapture {
  const runtimeLogs: string[] = [];
  const runtimeErrors: string[] = [];

  const defaultRuntime: CliMockOutputRuntime = {
    log: vi.fn((...args) => runtimeLogs.push(args.map(String).join(" "))),
    error: vi.fn((...args) => runtimeErrors.push(args.map(String).join(" "))),
    writeJson: vi.fn((value, space = 2) => defaultRuntime.log(JSON.stringify(value, null, space))),
    writeStdout: vi.fn((value) => defaultRuntime.log(value.replace(/\n$/, ""))),
    exit: vi.fn((code) => { throw new Error(`__exit__:${code}`); }),
  };

  return { runtimeLogs, runtimeErrors, defaultRuntime, resetRuntimeCapture: () => {
    runtimeLogs.length = 0;
    runtimeErrors.length = 0;
  }};
}
```

Tests use `expect(runtimeLogs).toContain(...)` or `expect(runtimeErrors[0]).toMatch(...)` without touching `process.stdout`.

---

## Time control

Tests for cron scheduling, retry backoff, and session expiry need deterministic time. OpenClaw's cron harness wraps the standard `vi.useFakeTimers()` call with shared reset logic:

```typescript
// src/cron/service.test-harness.ts
export function installCronTestHooks(options: {
  logger: ReturnType<typeof createNoopLogger>;
  baseTimeIso?: string;
}) {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.clearAllTimers();  // clear any leaked timers from previous file
    vi.setSystemTime(new Date(options.baseTimeIso ?? "2025-12-13T00:00:00.000Z"));
    options.logger.debug.mockClear();
    // ...
  });

  afterEach(() => {
    vi.clearAllTimers();
    vi.useRealTimers();
  });
}
```

The explicit `vi.clearAllTimers()` guard in `beforeEach` is important when tests run with `--isolate=false`: leaked timers from a previous file can otherwise fire mid-test.

---

## Process-level harnesses (E2E)

End-to-end tests spawn the real gateway binary and interact with it over HTTP/WebSockets. The `spawnGatewayInstance` function in `test/helpers/gateway-e2e-harness.ts` manages the full lifecycle:

```typescript
export type GatewayInstance = {
  name: string;
  port: number;
  hookToken: string;
  gatewayToken: string;
  homeDir: string;
  stateDir: string;
  configPath: string;
  child: ChildProcessWithoutNullStreams;
  stdout: string[];
  stderr: string[];
};

export async function spawnGatewayInstance(name: string): Promise<GatewayInstance> {
  const port = await getFreePort();          // ephemeral port to avoid conflicts
  const homeDir = await fs.mkdtemp(...);     // isolated home for this instance
  await fs.writeFile(configPath, yaml(...)); // minimal config

  const child = spawn("node", ["dist/cli.js", "gateway", "run", "--port", String(port)], {
    env: { ...process.env, HOME: homeDir, ... },
  });

  child.stdout.on("data", (chunk) => stdout.push(String(chunk)));
  child.stderr.on("data", (chunk) => stderr.push(String(chunk)));

  await waitForPortOpen(child, stdout, stderr, port, GATEWAY_START_TIMEOUT_MS);
  return { name, port, homeDir, /* ... */ child, stdout, stderr };
}
```

Key design decisions:
- **Ephemeral port allocation** avoids conflicts between parallel test runs.
- **Stdout/stderr capture** is collected in arrays, not streamed to the console, so test output stays clean while diagnostics remain available when a test fails.
- **Port readiness polling** at 10 ms intervals gives reliable startup detection without sleeping for a fixed duration.
- **Early-exit detection** in the polling loop surfaces the full captured output as the error message, making failures self-diagnosing.

---

## Async synchronization primitives

Tests for event-driven services (cron jobs firing, agent runs completing) need a way to wait for a specific event without busy-polling. The cron harness provides two utilities:

### `createFinishedBarrier`

```typescript
export function createFinishedBarrier() {
  const resolvers = new Map<string, (evt: CronEvent) => void>();
  return {
    waitForOk: (jobId: string) =>
      new Promise<CronEvent>((resolve) => { resolvers.set(jobId, resolve); }),
    onEvent: (evt: CronEvent) => {
      if (evt.action !== "finished" || evt.status !== "ok") return;
      resolvers.get(evt.jobId)?.(evt);
      resolvers.delete(evt.jobId);
    },
  };
}
```

Tests call `await finished.waitForOk("my-job")` and it resolves the moment the service fires the matching event—no `sleep()` needed.

### `createDeferred`

```typescript
export function createDeferred<T>() {
  let resolve!: (value: T) => void;
  let reject!: (reason?: unknown) => void;
  const promise = new Promise<T>((res, rej) => { resolve = res; reject = rej; });
  return { promise, resolve, reject };
}
```

Used anywhere a test needs to manually resolve a promise at a controlled point—for example, to gate an async mock until the test has verified intermediate state.

---

## Type-safe mock types

Vitest's `vi.fn()` returns an inferred type that can trigger TS2742 ("inferred type cannot be named") when used across module boundaries. OpenClaw centralizes the mock type in `src/test-utils/vitest-mock-fn.ts`:

```typescript
// oxlint-disable-next-line typescript/no-explicit-any
export type MockFn<T extends (...args: any[]) => any = (...args: any[]) => any> =
  import("vitest").Mock<T>;
```

Harness types use `MockFn` bound to the production signature:

```typescript
export type NoopLogger = {
  debug: MockFn;
  info: MockFn;
  warn: MockFn;
  error: MockFn;
};

export type CronServiceHarness = {
  cron: CronService;
  enqueueSystemEvent: MockFn<CronServiceDeps["enqueueSystemEvent"]>;
  requestHeartbeatNow: MockFn<CronServiceDeps["requestHeartbeatNow"]>;
};
```

Binding to the production signature means TypeScript checks call-site arguments in test assertions, not just `expect(mock).toHaveBeenCalled()`.

---

## Where harnesses live

| Location | Contains |
|---|---|
| `src/test-utils/` | Repo-wide utilities: `MockFn`, `fixture-suite`, `env`, `channel-plugin-test-fixtures`, `temp-home` |
| `src/test-helpers/` | Lower-level helpers: temp dirs, HTTP utils, SSRF utilities |
| `src/<domain>/test-helpers/` | Domain-specific helpers (agents, CLI, auto-reply) |
| `src/<domain>/<module>.test-harness.ts` | Module-level harness colocated with production code |
| `extensions/<id>/test-helpers/` | Extension-scoped helpers |
| `test/helpers/` | Top-level harnesses for full-process E2E tests |
| `scripts/e2e/` | Script-level E2E harnesses (MCP channel harness, Docker runners) |

---

## Best practices

### Keep the harness colocated with the module

A harness file belongs next to the module it supports, not in a central `test/` directory. Colocation makes it obvious what infrastructure a module owns and keeps deletions safe.

### Expose typed handles, not raw `vi.fn()`

Return a typed struct from your factory function:

```typescript
// Good: typed handles
export function createCronStoreHarness() {
  return { makeStorePath };
}

// Avoid: leaking raw vi.fn() into test scope
export const makeStorePath = vi.fn();
```

Named, typed handles make test assertions readable and prevent accidental reuse of state between suites.

### Install lifecycle hooks inside the harness

Put `beforeAll`, `afterAll`, `beforeEach`, `afterEach` inside the harness factory, not in the test file. This keeps test files focused on the behavior under test:

```typescript
// Inside harness factory:
beforeAll(async () => { fixtureRoot = await fs.mkdtemp(...); });
afterAll(async ()  => { await fs.rm(fixtureRoot, { recursive: true, force: true }); });
```

### Provide reset functions alongside setup

For harnesses used across multiple tests in a file, expose a `reset*` function:

```typescript
export function resetReplyRuntimeMocks(mocks: ReplyRuntimeMocks) {
  mocks.runEmbeddedPiAgent.mockClear();
  mocks.loadModelCatalog.mockClear();
  mocks.loadModelCatalog.mockResolvedValue([/* sensible default */]);
}
```

Reset in `beforeEach` rather than recreating the entire harness in each test. Reuse is cheaper and avoids re-running `vi.mock()` setup.

### Never leak timers

If you call `vi.useFakeTimers()`, always pair it with `vi.useRealTimers()` in `afterEach`. Add `vi.clearAllTimers()` in `beforeEach` as a defensive guard against leaked timers from other files when running with `--isolate=false`.

### Use ephemeral ports in process harnesses

Never hardcode a port in an E2E harness. Bind to port `0` to let the OS allocate an available port, then capture the assigned address:

```typescript
const srv = net.createServer();
await new Promise<void>((resolve) => srv.listen(0, "127.0.0.1", resolve));
const port = (srv.address() as net.AddressInfo).port;
await new Promise<void>((resolve) => srv.close(() => resolve()));
```

### Capture stdout/stderr in arrays, not on the console

E2E harnesses that spawn child processes should pipe stdout/stderr into string arrays. The output is silent during passing runs and available in full when a test fails—the best of both worlds.

### Use barriers over `sleep`

Replace `await sleep(N)` with a purpose-built `createFinishedBarrier()` or `createDeferred()`. Sleep-based synchronization is fragile under load and inflates test duration on slow CI hosts.

---

## Related documentation

- [Testing](/help/testing) — test suites, commands, live tests, and Docker runners
- [Plugin architecture](/plugins/architecture) — boundary rules that shape what harnesses can safely mock
- [Extension guide](/plugins/building-plugins) — how to add harnesses for new plugin types
