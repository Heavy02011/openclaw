# OpenClaw — Reverse Engineering & Coding Cheat Sheet

> **Repository:** [Heavy02011/openclaw](https://github.com/Heavy02011/openclaw)
> **Primary Language:** TypeScript (ESM, strict mode)
> **Runtime:** Node.js 22+ (Bun also supported for dev/scripts)
> **Category:** Multi-channel AI gateway with extensible messaging integrations

---

## Phase 1: Forensic Reconstruction — How Was This Built?

### 1.1 Creation Timeline & Process

#### Likely Build Order

1. **Core runtime & config** — `src/runtime.ts`, `src/config/`, `src/infra/` utilities (env, ports, file-lock). The foundation needed before anything else.
2. **CLI skeleton** — `src/cli/`, Commander-based program builder (`build-program.ts`), dependency injection factory (`deps.ts`).
3. **Single channel (WhatsApp/Web)** — `src/web/` (Baileys-based WhatsApp Web) was likely the first messaging channel, given its depth and maturity.
4. **Agent core** — `src/agents/` with model integration, system prompts, tool execution, session management.
5. **Gateway server** — `src/gateway/` HTTP server, method handlers, channel routing, plugin loading.
6. **Additional channels** — Telegram (`grammy`), Discord, Slack, Signal, iMessage, LINE added iteratively.
7. **Plugin SDK & extensions** — `src/plugin-sdk/`, `extensions/` workspace packages for 40+ integrations (Matrix, MS Teams, Zalo, Google Chat, etc.).
8. **Platform apps** — `apps/macos/`, `apps/ios/`, `apps/android/` — native wrappers came after the core was stable.
9. **UI layer** — `ui/` (Canvas/web interface) for browser-based interaction.
10. **Docs & i18n** — `docs/` (Mintlify), `docs/zh-CN/`, `docs/ja-JP/` localization pipeline.

#### Signs of AI-Assisted vs. Hand-Written Code

- **Hand-written indicators:** Varied comment density across files, pragmatic naming (e.g., `wired-hooks-llm.ts` — developer shorthand, not AI-clean naming), aggressive use of dynamic imports for lazy loading (performance optimization an AI rarely suggests unprompted).
- **Iterative refinement signals:** Migration support in config (`src/config/`), session transcript repair (`session-transcript-repair.ts`), model fallback chains (`model-fallback.ts`), and `FailoverError` with status-code mapping all indicate production battle-testing.
- **Refactoring evidence:** `createDefaultDeps()` factory with adapter pattern (`CliDeps` → `OutboundSendDeps`) suggests an interface was extracted after multiple channels were added. The plugin SDK's massive export surface (454 lines in `index.ts`) shows organic growth.

#### Build Approach

**Bottom-up, feature-driven** — the project grew channel by channel, with shared abstractions extracted after patterns emerged. Evidence: adapter interfaces for channels were likely extracted after 3+ channels existed (the `ChannelPlugin` type has 20+ optional adapter slots, suggesting incremental addition).

### 1.2 Bootstrapping Blueprint

If rebuilding from scratch:

```bash
# 1. Initialize project
mkdir openclaw && cd openclaw
pnpm init
# Set type: "module" in package.json, engines: { node: ">=22.12.0" }

# 2. Install core tooling FIRST
pnpm add -D typescript vitest oxlint @oxc/oxfmt tsdown
pnpm add commander @clack/prompts zod tslog

# 3. Create skeleton
mkdir -p src/{cli,config,infra,agents,channels,gateway,plugin-sdk}
mkdir -p src/{logging,media,security,terminal}
mkdir -p extensions docs scripts

# 4. Bootstrap config files
touch tsconfig.json vitest.config.ts .oxlintrc.json .oxfmtrc.jsonc
touch src/runtime.ts src/index.ts src/cli/deps.ts

# 5. Minimum viable skeleton
# - src/runtime.ts (RuntimeEnv type + defaultRuntime)
# - src/cli/deps.ts (createDefaultDeps factory)
# - src/cli/program/build-program.ts (Commander program builder)
# - src/config/config.ts (Zod-validated JSON5 config reader)
# - src/index.ts (entry point wiring)

# 6. First channel integration
mkdir -p src/web  # WhatsApp Web via Baileys

# 7. Build and verify
pnpm build && pnpm test
```

**Dependency installation order rationale:**

1. TypeScript + build tools (tsdown) — need to compile before anything runs
2. CLI framework (Commander + clack) — CLI is the primary interface
3. Validation (Zod) — config validation is critical early
4. Logging (tslog) — observability from day one
5. Channel SDKs — added per-channel as needed

---

## Phase 2: Architecture & Design Decisions

### 2.1 Structural Analysis

#### Directory Structure Rationale

```
src/
├── agents/        # Agent orchestration, model selection, tools, sessions
├── channels/      # Multi-channel routing and delivery
├── cli/           # CLI commands, DI factory, program builder
│   └── commands/  # Individual CLI command implementations
├── config/        # Zod-validated config management (JSON5)
├── gateway/       # HTTP gateway server, methods, plugins, auth
├── infra/         # Low-level utilities (ports, env, file-lock, format)
├── logging/       # tslog-based structured logging
├── media/         # Media processing (images, audio, MIME)
├── plugin-sdk/    # Public plugin API surface
├── providers/     # AI provider integrations (Copilot, Google, Qwen)
├── security/      # Audit, ACLs, dangerous tool policies
├── terminal/      # TTY output (tables, palette, theme)
├── tts/           # Text-to-speech engine
├── telegram/      # Telegram channel (grammy)
├── discord/       # Discord channel
├── slack/         # Slack channel (@slack/bolt)
├── signal/        # Signal channel
├── imessage/      # iMessage channel
├── web/           # WhatsApp Web (Baileys)
├── line/          # LINE channel
└── tui/           # Terminal UI
extensions/        # 40+ plugin packages (workspace packages)
apps/              # Platform apps (macOS, iOS, Android)
ui/                # Web UI (Canvas/browser interface)
docs/              # Mintlify documentation
scripts/           # Build, test, deployment scripts
```

**Pattern:** Hybrid **feature-based + layer-based** architecture. Core infrastructure (`infra/`, `logging/`, `security/`) is layered, while channels are feature-sliced. This balances discoverability (find a channel's code in one place) with shared infrastructure reuse.

#### Dependency Graph

```
CLI (src/cli/) ──→ Agents (src/agents/) ──→ Providers (src/providers/)
      │                    │
      ▼                    ▼
Gateway (src/gateway/) ──→ Channels (src/channels/)
      │                    │
      ▼                    ▼
Plugin SDK ←────── Extensions (extensions/*)
      │
      ▼
Infra (src/infra/) ←── Config (src/config/) ←── Security (src/security/)
```

**Boundaries:**

- Plugin SDK defines the public API surface — extensions depend only on `openclaw/plugin-sdk`
- Channels are independently loadable (lazy dynamic imports)
- Gateway orchestrates everything but doesn't contain business logic

#### Entry Points

1. **CLI entry:** `src/index.ts` → `src/cli/program/build-program.ts` → Commander program
2. **Gateway entry:** Gateway command → `src/gateway/server.impl.ts` → Express HTTP server
3. **Request flow:** HTTP request → gateway auth → channel routing → agent orchestration → model provider → response streaming → channel delivery

#### Configuration Strategy

- **Format:** JSON5 files with Zod schema validation
- **Location:** `~/.openclaw/` directory
- **Secrets:** Credentials stored at `~/.openclaw/credentials/`
- **Env vars:** `OPENCLAW_*` prefix convention (see `.env.example`)
- **Environment separation:** Docker env vars override file-based config
- **Migration:** Built-in config migration for legacy formats

### 2.2 Key Design Patterns

#### ⭐ Pattern: Factory + Lazy Loading (Dependency Injection)

**Where:** `src/cli/deps.ts`
**Problem solved:** Avoid loading all channel SDKs at startup — only load what's actually used.

```typescript
// PATTERN: Factory + Lazy Loading DI
// USE WHEN: You have many optional modules and want fast startup
// STOLEN FROM: src/cli/deps.ts

export type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  // ... per-channel send functions
};

export function createDefaultDeps(): CliDeps {
  return {
    sendMessageWhatsApp: async (...args) => {
      // Dynamic import — only loads when first called
      const { sendMessageWhatsApp } = await import("../channels/web/index.js");
      return await sendMessageWhatsApp(...args);
    },
    sendMessageTelegram: async (...args) => {
      const { sendMessageTelegram } = await import("../telegram/index.js");
      return await sendMessageTelegram(...args);
    },
  };
}
```

**When to steal this:** Any CLI tool or server with many optional integrations. Reduces cold start time from seconds to milliseconds.

#### ⭐ Pattern: Adapter Interface Mapping

**Where:** `src/cli/deps.ts` — `createOutboundSendDeps()`
**Problem solved:** Different layers need different interfaces for the same underlying functionality.

```typescript
// PATTERN: Adapter Interface Mapping
// USE WHEN: Two subsystems use different shapes for the same data
// STOLEN FROM: src/cli/deps.ts

export function createOutboundSendDeps(deps: CliDeps): OutboundSendDeps {
  return {
    sendWhatsApp: deps.sendMessageWhatsApp,
    sendTelegram: deps.sendMessageTelegram,
    sendDiscord: deps.sendMessageDiscord,
    // Maps CliDeps shape → OutboundSendDeps shape
  };
}
```

**When to steal this:** When your CLI layer and server layer need the same capabilities but with different naming conventions.

#### ⭐ Pattern: Plugin Adapter Composition

**Where:** `src/plugin-sdk/index.ts`, plugin types
**Problem solved:** Allow plugins to implement only the capabilities they need.

```typescript
// PATTERN: Optional Adapter Composition
// USE WHEN: Building a plugin system where plugins have varied capabilities
// STOLEN FROM: src/plugins/types.ts

export type ChannelPlugin = {
  auth?: ChannelAuthAdapter; // Optional: authentication
  messaging?: ChannelMessagingAdapter; // Optional: message handling
  outbound?: ChannelOutboundAdapter; // Optional: sending messages
  status?: ChannelStatusAdapter; // Optional: health/status
  // ... 20+ optional adapter slots
};
```

**When to steal this:** Any extensible system where plugins shouldn't be forced to implement everything.

#### Pattern: Builder + Composition (CLI Construction)

**Where:** `src/cli/program/build-program.ts`
**Problem solved:** Complex CLI programs need step-by-step configuration.

```typescript
// PATTERN: Builder + Composition
// USE WHEN: Building CLI programs with many commands and hooks
// STOLEN FROM: src/cli/program/build-program.ts

export function buildProgram() {
  const program = new Command();
  const ctx = createProgramContext();

  setProgramContext(program, ctx);
  configureProgramHelp(program, ctx);
  registerPreActionHooks(program, ctx.programVersion);
  registerProgramCommands(program, ctx, argv);

  return program;
}
```

#### Pattern: Type-Driven Runtime Configuration

**Where:** `src/runtime.ts`
**Problem solved:** Different environments (production, testing) need different runtime behaviors.

```typescript
// PATTERN: Type-Driven Runtime
// USE WHEN: You need swappable runtime behavior (prod vs test)
// STOLEN FROM: src/runtime.ts

export type RuntimeEnv = {
  log: typeof console.log;
  error: typeof console.error;
  exit: (code: number) => never;
};

export const defaultRuntime: RuntimeEnv = {
  ...createRuntimeIo(),
  exit: (code) => {
    restoreTerminalState("runtime exit");
    process.exit(code);
  },
};

// Test-friendly: doesn't call process.exit
export function createNonExitingRuntime(): RuntimeEnv {
  /* ... */
}
```

#### Pattern: Structured Error Hierarchy with Failover

**Where:** `src/agents/failover-error.ts`
**Problem solved:** AI provider errors need semantic meaning for automatic retry/failover.

```typescript
// PATTERN: Semantic Error Classification
// USE WHEN: Errors from external services need automated handling
// STOLEN FROM: src/agents/failover-error.ts

type FailoverReason = "billing" | "rate_limit" | "auth" | "timeout" | "format";

class FailoverError extends Error {
  reason: FailoverReason;
  provider?: string;
  model?: string;
  status?: number;
}

// Status code → reason mapping
// 402 → "billing", 429 → "rate_limit", 401/403 → "auth", 408 → "timeout"
function resolveFailoverReasonFromError(err: unknown): FailoverReason {
  /* ... */
}
```

#### Pattern: Child Logger Strategy

**Where:** `src/gateway/server.impl.ts`, `src/logging/logger.ts`
**Problem solved:** Complex systems need per-subsystem log filtering.

```typescript
// PATTERN: Child Logger Strategy
// USE WHEN: You need per-module log filtering in a large application
// STOLEN FROM: src/gateway/server.impl.ts

const log = createSubsystemLogger("gateway");
const logCanvas = log.child("canvas");
const logDiscovery = log.child("discovery");
const logAuth = log.child("auth");

// Each child logger inherits parent config but can be filtered independently
logCanvas.info("Canvas initialized"); // → [gateway:canvas] Canvas initialized
```

### 2.3 API & Interface Design

#### Gateway HTTP API

- **Structure:** Method-based RPC over HTTP (not REST) — `POST /method/:methodName`
- **Auth:** Token-based (`OPENCLAW_GATEWAY_TOKEN`) with rate limiting (`AuthRateLimiter`)
- **Rate limiting:** In-memory, configurable `maxAttempts`, `windowMs`, `lockoutMs` with `429 + Retry-After` headers
- **Error handling:** Structured error responses with status code mapping
- **Caching:** In-memory maps with TTL pruning (session titles, cost usage, agent runs, health snapshots)

#### Hook System

```typescript
// Hook actions: "wake" (activate agent) or "agent" (deliver message)
// Wake modes: "now" or "next-heartbeat"
// Body size limit: 256KB default
// Auth: per-hook token validation
// Transform: custom JS modules per hook mapping
```

---

## Phase 3: Code Craft & Techniques

### 3.1 TypeScript-Specific Techniques

#### Language Features

| Technique                      | Usage                                                 | Example Location               |
| ------------------------------ | ----------------------------------------------------- | ------------------------------ |
| **Dynamic imports**            | Lazy-load channels/providers on demand                | `src/cli/deps.ts`              |
| **Discriminated unions**       | `FailoverReason` type for error classification        | `src/agents/failover-error.ts` |
| **Type-only imports**          | `import type { X }` for zero-runtime-cost types       | Throughout codebase            |
| **Zod schemas**                | Runtime validation + TypeScript type inference        | `src/config/`                  |
| **ESM with .js extensions**    | Cross-package imports use `.js` even for `.ts` files  | All imports                    |
| **Spread composition**         | Build objects by spreading base + overrides           | `src/runtime.ts`               |
| **Optional chaining patterns** | 20+ optional adapter slots in plugin types            | `src/plugins/types.ts`         |
| **Async factories**            | `createDefaultDeps()` returns async function wrappers | `src/cli/deps.ts`              |

#### Standard Library Usage

- **`process.exit`** wrapped in `RuntimeEnv` for testability
- **`crypto`** for session tokens and checksums (`src/security/audit.ts`)
- **`fs/promises`** with file locks (`src/infra/file-lock.ts`) for concurrent access safety
- **`path`** with platform-aware handling (Windows ACLs in `src/security/windows-acl.ts`)

#### Third-Party Library Patterns

| Library         | Pattern                 | Purpose                       |
| --------------- | ----------------------- | ----------------------------- |
| **Commander**   | Builder composition     | CLI program construction      |
| **Zod**         | Schema-first validation | Config + API input validation |
| **tslog**       | Child logger strategy   | Per-subsystem logging         |
| **Express**     | Method-based RPC        | Gateway HTTP server           |
| **grammy**      | Bot framework adapter   | Telegram integration          |
| **@slack/bolt** | Event-driven adapter    | Slack integration             |
| **Baileys**     | WhatsApp Web protocol   | WhatsApp channel              |
| **Sharp**       | Image pipeline          | Media processing              |
| **Vitest**      | Forks pool isolation    | Test execution                |

#### ⭐ Performance Optimizations

- **Lazy dynamic imports:** Channels loaded only when first message arrives — startup stays fast even with 40+ channels
- **In-memory caching with TTL:** `DedupeCache`, `DirectoryCache<T>` with configurable max size and TTL
- **Forks pool testing:** Vitest uses process isolation to prevent env pollution between tests
- **Pruning strategies:** Agent run cache, cost usage cache pruned periodically (not on every access)

#### Error Handling Philosophy

- **Custom error hierarchy:** `FailoverError` with semantic `reason` field for automated retry decisions
- **Status code mapping:** HTTP status → failover reason (402→billing, 429→rate_limit, 401/403→auth)
- **Graceful degradation:** Model fallback chains (`model-fallback.ts`) try alternative providers on failure
- **Coercion helpers:** `coerceToFailoverError()` normalizes unknown errors into structured format

### 3.2 Testing Strategy

#### Configuration

```typescript
// vitest.config.ts highlights
export default defineConfig({
  test: {
    pool: "forks", // Process isolation
    poolOptions: { forks: { maxForks: 16 } }, // Parallel (2-3 in CI)
    testTimeout: 120_000, // 2 min default
    hookTimeout: isWindows ? 180_000 : undefined,
    coverage: {
      provider: "v8",
      thresholds: {
        lines: 70,
        functions: 70,
        branches: 55,
        statements: 70,
      },
    },
  },
});
```

#### Test Structure & Naming

- **Colocated tests:** `*.test.ts` next to source files
- **E2E tests:** `*.e2e.test.ts` (excluded from unit suite)
- **Live tests:** `*.live.test.ts` (require `CLAWDBOT_LIVE_TEST=1`)
- **Pattern:** `describe` → `it` → `expect` (Vitest)

#### Fixture & Factory Patterns

```typescript
// PATTERN: Mock Factory Helper
// USE WHEN: Creating test doubles for complex plugin interfaces
// STOLEN FROM: src/plugins/wired-hooks-llm.test.ts

function createMockPluginRegistry() {
  return {
    getHookHandlers: vi.fn(),
    registerPlugin: vi.fn(),
    // ... minimal mock surface
  };
}
```

#### Mocking Strategy

- **`vi.fn()`** for function mocks with call tracking
- **`vi.mock()`** for module-level mocking
- **`expect.objectContaining()`** for partial payload assertions
- **Dynamic imports mocked** for channel isolation in tests
- **No external services in unit tests** — all provider calls mocked

#### Test Coverage

| Type              | Scope                                 | Trigger                     |
| ----------------- | ------------------------------------- | --------------------------- |
| **Unit**          | `pnpm test`                           | Every PR, colocated tests   |
| **E2E**           | `pnpm test:e2e`                       | Specific workflows          |
| **Live**          | `CLAWDBOT_LIVE_TEST=1 pnpm test:live` | Manual, real API keys       |
| **Docker**        | `pnpm test:docker:live-models`        | Integration with containers |
| **Install smoke** | `pnpm test:install:smoke`             | Pre-release validation      |

### 3.3 LLM/AI Integration Patterns

#### System Prompt Construction

```typescript
// PATTERN: Multi-mode System Prompt
// USE WHEN: Different agent contexts need different prompt depths
// STOLEN FROM: src/agents/system-prompt.ts

// Three modes:
// "full"    — main agent: skills, memory, tooling, workspace, runtime
// "minimal" — subagents: only essential context
// "none"   — basic only: no dynamic sections

// Dynamic sections injected:
// - Skills guidance (read-then-follow pattern)
// - Memory retrieval with citations mode
// - Workspace context
// - Runtime environment details
```

#### Model Abstraction Layer

- **Model catalog:** `src/agents/model-catalog.ts` — registry of available models across providers
- **Model selection:** `src/agents/model-selection.ts` — picks best model based on task + availability
- **Model fallback:** `src/agents/model-fallback.ts` — chains of alternative providers
- **Auth profiles:** `src/agents/auth-profiles/` — per-provider authentication

#### Streaming

- **Raw stream processing:** `pi-embedded-subscribe.raw-stream.ts`
- **Soft chunking:** Paragraph-based chunk aggregation for smooth delivery
- **Block-reply flushing:** Aggregates chunks before sending to channels

#### Token/Cost Management

- **Cost usage cache:** Tracks spend per session with TTL
- **Token management:** Per-provider token refresh and rotation
- **OAuth subscriptions:** Claude Pro/Max, OpenAI subscription support

#### Tool Execution & Safety

```typescript
// PATTERN: Dangerous Tool Policy
// USE WHEN: LLM agents need restricted tool access
// STOLEN FROM: src/security/dangerous-tools.ts

// Default deny list: sessions_spawn, gateway, fs_write
// ACP (Agent Control Protocol) restrictions
// Per-tool policy enforcement
// Sandbox isolation for code execution
```

---

## Phase 4: The Cheat Sheet

### 🏗️ Project Scaffolding Recipe

```bash
# Bootstrap a TypeScript ESM project with OpenClaw's patterns
mkdir my-project && cd my-project

# 1. Initialize
pnpm init
# Edit package.json: "type": "module", "engines": { "node": ">=22.12.0" }

# 2. Core tooling
pnpm add -D typescript vitest @vitest/coverage-v8 tsdown
pnpm add -D oxlint @oxc/oxfmt

# 3. Runtime deps
pnpm add commander @clack/prompts zod tslog

# 4. Config files
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
EOF

# 5. Create source skeleton
mkdir -p src/{cli,config,infra,plugins}
touch src/runtime.ts src/index.ts src/cli/deps.ts

# 6. Verify
pnpm build && pnpm test
```

### 📐 Architecture Template

```
src/
├── index.ts              # Entry point, wiring
├── runtime.ts            # RuntimeEnv type + factories (prod/test)
├── cli/
│   ├── deps.ts           # DI factory: createDefaultDeps()
│   ├── program/
│   │   └── build-program.ts  # Commander builder
│   └── commands/         # Individual CLI commands
├── config/
│   ├── config.ts         # Zod-validated config reader/writer
│   └── zod-schema.ts     # Config schema definitions
├── infra/
│   ├── env.ts            # Environment utilities
│   ├── ports.ts          # Port management
│   ├── file-lock.ts      # File locking for concurrent access
│   └── format-time.ts    # Centralized time formatters (DO NOT duplicate)
├── gateway/
│   ├── server.impl.ts    # HTTP server setup
│   ├── methods/          # RPC method handlers
│   └── auth-rate-limit.ts  # Rate limiting
├── logging/
│   └── logger.ts         # tslog with child loggers + transports
├── security/
│   ├── audit.ts          # Security findings collection
│   └── dangerous-tools.ts  # Tool access policies
├── terminal/
│   ├── palette.ts        # Color palette (shared CLI theme)
│   ├── theme.ts          # Theme utilities
│   └── table.ts          # Table rendering
├── plugins/
│   └── types.ts          # Plugin adapter interfaces
└── plugin-sdk/
    └── index.ts          # Public API surface for extensions
extensions/               # Workspace plugin packages
```

### 🔧 Configuration Patterns

```typescript
// PATTERN: Zod-Validated Config
// Stolen from: src/config/config.ts

import { z } from "zod";

const ConfigSchema = z.object({
  gateway: z.object({
    mode: z.enum(["local", "remote"]).default("local"),
    port: z.number().default(18789),
    token: z.string().optional(),
  }),
  logging: z.object({
    level: z.enum(["debug", "info", "warn", "error"]).default("info"),
    file: z.string().optional(),
  }),
});

type Config = z.infer<typeof ConfigSchema>;

// Read + validate + cache pattern
let cachedConfig: Config | null = null;
export function loadConfig(path: string): Config {
  if (cachedConfig) return cachedConfig;
  const raw = JSON.parse(readFileSync(path, "utf-8"));
  cachedConfig = ConfigSchema.parse(raw);
  return cachedConfig;
}
```

```typescript
// PATTERN: Environment Variable Config
// Stolen from: .env.example

// Convention: PRODUCT_SUBSYSTEM_KEY
// OPENCLAW_GATEWAY_TOKEN=...
// OPENCLAW_GATEWAY_PASSWORD=...
// ANTHROPIC_API_KEY=...
// OPENAI_API_KEY=...
```

```typescript
// PATTERN: Structured Logging Setup
// Stolen from: src/logging/logger.ts

import { Logger } from "tslog";

type LogTransport = (logObj: LogTransportRecord) => void;

const log = new Logger({ name: "gateway" });
const logCanvas = log.getSubLogger({ name: "canvas" });
const logAuth = log.getSubLogger({ name: "auth" });

// Pluggable transports for file/remote logging
function registerLogTransport(transport: LogTransport) {
  /* ... */
}
```

### 🧱 Core Code Patterns

```typescript
// PATTERN: Lazy-Loading Dependency Injection
// USE WHEN: Multiple optional integrations that shouldn't load at startup
// STOLEN FROM: src/cli/deps.ts

export type ServiceDeps = {
  sendEmail: (to: string, body: string) => Promise<void>;
  sendSlack: (channel: string, text: string) => Promise<void>;
};

export function createDefaultDeps(): ServiceDeps {
  return {
    sendEmail: async (...args) => {
      const { sendEmail } = await import("./email/index.js");
      return await sendEmail(...args);
    },
    sendSlack: async (...args) => {
      const { sendSlack } = await import("./slack/index.js");
      return await sendSlack(...args);
    },
  };
}
```

```typescript
// PATTERN: Semantic Error with Auto-Retry
// USE WHEN: External service errors need automated failover decisions
// STOLEN FROM: src/agents/failover-error.ts

type FailoverReason = "billing" | "rate_limit" | "auth" | "timeout";

class ServiceError extends Error {
  constructor(
    message: string,
    public reason: FailoverReason,
    public provider?: string,
    public status?: number,
  ) {
    super(message);
  }
}

function classifyError(status: number): FailoverReason {
  if (status === 402) return "billing";
  if (status === 429) return "rate_limit";
  if (status === 401 || status === 403) return "auth";
  if (status === 408) return "timeout";
  return "timeout"; // default
}
```

```typescript
// PATTERN: In-Memory Cache with TTL
// USE WHEN: Reducing external calls without Redis
// STOLEN FROM: src/infra/ (DedupeCache pattern)

class TtlCache<K, V> {
  private cache = new Map<K, { value: V; expires: number }>();

  constructor(
    private ttlMs: number,
    private maxSize: number,
  ) {}

  get(key: K): V | undefined {
    const entry = this.cache.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expires) {
      this.cache.delete(key);
      return undefined;
    }
    return entry.value;
  }

  set(key: K, value: V): void {
    if (this.cache.size >= this.maxSize) {
      const oldest = this.cache.keys().next().value;
      if (oldest !== undefined) this.cache.delete(oldest);
    }
    this.cache.set(key, { value, expires: Date.now() + this.ttlMs });
  }
}
```

```typescript
// PATTERN: Rate Limiter
// USE WHEN: Protecting endpoints from brute-force
// STOLEN FROM: src/gateway/auth-rate-limit.ts

type RateLimiterConfig = {
  maxAttempts: number;
  windowMs: number;
  lockoutMs: number;
};

class RateLimiter {
  private attempts = new Map<string, { count: number; resetAt: number }>();

  check(key: string): { allowed: boolean; retryAfter?: number } {
    const now = Date.now();
    const entry = this.attempts.get(key);
    if (entry && now < entry.resetAt && entry.count >= this.config.maxAttempts) {
      return { allowed: false, retryAfter: Math.ceil((entry.resetAt - now) / 1000) };
    }
    return { allowed: true };
  }

  record(key: string): void {
    const now = Date.now();
    const entry = this.attempts.get(key) ?? { count: 0, resetAt: now + this.config.windowMs };
    entry.count++;
    this.attempts.set(key, entry);
  }

  constructor(private config: RateLimiterConfig) {}
}
```

```typescript
// PATTERN: Plugin Adapter Interface
// USE WHEN: Building extensible systems with optional capabilities
// STOLEN FROM: src/plugins/types.ts

export type Plugin = {
  name: string;
  version: string;
  auth?: AuthAdapter; // Optional capability
  messaging?: MsgAdapter; // Optional capability
  status?: StatusAdapter; // Optional capability
};

// Loader checks which adapters exist
function loadPlugin(plugin: Plugin) {
  if (plugin.auth) registerAuth(plugin.auth);
  if (plugin.messaging) registerMessaging(plugin.messaging);
  if (plugin.status) registerStatus(plugin.status);
}
```

### 🧪 Testing Recipes

```typescript
// PATTERN: Mock Factory
// USE WHEN: Tests need consistent mock objects
// STOLEN FROM: src/plugins/wired-hooks-llm.test.ts

import { describe, it, expect, vi } from "vitest";

function createMockRegistry() {
  return {
    getHandlers: vi.fn().mockReturnValue([]),
    register: vi.fn(),
  };
}

describe("HookRunner", () => {
  it("invokes registered handlers", async () => {
    const registry = createMockRegistry();
    const handler = vi.fn();
    registry.getHandlers.mockReturnValue([handler]);

    await runHook(registry, { event: "test", payload: { id: 1 } });

    expect(handler).toHaveBeenCalledWith(expect.objectContaining({ event: "test" }));
  });
});
```

```typescript
// PATTERN: Vitest Config for Large Projects
// USE WHEN: Tests need process isolation + coverage thresholds
// STOLEN FROM: vitest.config.ts

import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    pool: "forks",
    poolOptions: { forks: { maxForks: 16 } },
    testTimeout: 120_000,
    include: ["src/**/*.test.ts"],
    exclude: ["**/*.e2e.test.ts", "**/*.live.test.ts"],
    coverage: {
      provider: "v8",
      thresholds: { lines: 70, functions: 70, branches: 55 },
    },
  },
});
```

### 🚀 Deployment & DevOps

```dockerfile
# PATTERN: Production Docker Image
# STOLEN FROM: Dockerfile

FROM node:22-bookworm
# Install Bun for build scripts
RUN curl -fsSL https://bun.sh/install | bash

WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile
RUN pnpm build && pnpm ui:build

# Security: run as non-root
USER node
# Security: bind to loopback by default
CMD ["node", "dist/index.js", "gateway", "--allow-unconfigured"]
```

```yaml
# PATTERN: Docker Compose with Environment Config
# STOLEN FROM: docker-compose.yml

services:
  gateway:
    build: .
    ports:
      - "18789:18789" # Gateway
      - "18790:18790" # Bridge
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - openclaw-config:/home/node/.openclaw
```

```yaml
# PATTERN: Smart CI Scope Detection
# STOLEN FROM: .github/workflows/ci.yml

# Detect docs-only PRs → skip heavy builds
# Detect changed areas → run only affected jobs
# Concurrency: cancel old runs on new pushes
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

### ⚡ One-Liners & Micro-Patterns

```typescript
// Spread composition for runtime variants
export const testRuntime: RuntimeEnv = { ...defaultRuntime, exit: () => {} };
```

```typescript
// Zod enum → TypeScript type (zero duplication)
const Status = z.enum(["active", "inactive", "pending"]);
type Status = z.infer<typeof Status>;
```

```typescript
// Dynamic import for optional heavy dependencies
const sharp = await import("sharp").catch(() => null);
if (!sharp) {
  console.warn("sharp not available");
  return;
}
```

```typescript
// ESM .js extension imports (TypeScript + ESM pattern)
import { utils } from "../infra/utils.js"; // .js even for .ts files
```

```typescript
// Optional adapter check (plugin system)
if (plugin.messaging) await plugin.messaging.handle(msg);
```

```typescript
// Partial match assertions (Vitest)
expect(result).toEqual(expect.objectContaining({ status: "ok" }));
```

```typescript
// File lock for concurrent access safety
import { withFileLock } from "../infra/file-lock.js";
await withFileLock(lockPath, async () => {
  /* critical section */
});
```

### 🚫 Anti-Patterns Avoided

| Anti-Pattern                  | What OpenClaw Does Instead                                                       |
| ----------------------------- | -------------------------------------------------------------------------------- |
| **God classes**               | Composition via factory functions; no class inheritance hierarchies              |
| **Eager loading everything**  | Lazy dynamic imports — channels loaded on demand                                 |
| **`any` types**               | Strict TypeScript with Zod runtime validation                                    |
| **Singleton config**          | Factory-created config with snapshot caching and migration                       |
| **Console.log debugging**     | Structured tslog with child loggers, transports, and subsystem filtering         |
| **Hardcoded colors**          | Shared palette (`src/terminal/palette.ts`) — Lobster palette with semantic names |
| **Monolithic test suite**     | Forks pool isolation + separate suites (unit/e2e/live/docker)                    |
| **Re-export wrapper files**   | Direct imports from source modules (anti-redundancy rule)                        |
| **Class-based DI containers** | Plain function factories with typed objects                                      |
| **Global mutable state**      | `RuntimeEnv` type allows swapping behavior for testing                           |
| **Unvalidated config**        | Zod schemas at every config boundary                                             |
| **Flat error strings**        | `FailoverError` with semantic reason + provider + status metadata                |
| **Manual spinners/progress**  | Centralized `src/cli/progress.ts` with osc-progress + clack                      |

---

## Phase 5: Learning Roadmap

> Calibrated for an intermediate TypeScript developer exploring this codebase.

### 1. Immediate Wins — Adopt Today

| Technique                                   | Why It Matters                                                                         |
| ------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Zod for config validation**               | Eliminates an entire class of runtime errors. `z.infer<>` means zero type duplication. |
| **`vi.fn()` + `expect.objectContaining()`** | Makes tests resilient to payload shape changes — test what matters, ignore the rest.   |
| **ESM with `.js` extensions**               | The correct way to do TypeScript + ESM. Avoids resolution headaches.                   |
| **Child loggers**                           | `log.child("subsystem")` gives you free log filtering without any framework.           |
| **Shared color palette**                    | One file (`palette.ts`) prevents hardcoded ANSI codes scattered everywhere.            |

### 2. This Week — Study & Practice

| Technique                         | Why It Matters                                                                                                                                                       |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Factory + lazy loading DI**     | The `createDefaultDeps()` pattern is universally useful for any project with optional integrations. Practice by refactoring an existing project to use lazy imports. |
| **Plugin adapter composition**    | Understanding optional interface slots unlocks extensible architecture. Build a small plugin system with 3 adapters.                                                 |
| **Semantic error classification** | `FailoverError` pattern transforms error handling from "log and crash" to "classify and recover." Implement for any project calling external APIs.                   |
| **In-memory TTL cache**           | Replace ad-hoc `Map` caching with a proper TTL cache. Study the `DedupeCache` pattern.                                                                               |

### 3. Deep Dives Needed

| Concept                              | Why It Matters                                                                                                               | Time Investment |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- | --------------- |
| **Dynamic imports & code splitting** | Core to OpenClaw's fast startup. Understanding ESM module loading, top-level await, and circular dependency avoidance.       | 2-3 hours       |
| **Streaming architecture**           | Chunk aggregation, paragraph-based soft streaming, and block-reply flushing are advanced patterns for real-time AI delivery. | 4-6 hours       |
| **Workspace monorepo management**    | pnpm workspaces with 40+ extension packages. Understanding `workspace:*` protocol, hoisting, and plugin isolation.           | 3-4 hours       |
| **Security audit patterns**          | Automated checksum verification, permission auditing, and dangerous tool policies. Study `src/security/` thoroughly.         | 2-3 hours       |

### 4. Recommended Resources

| Resource                                                                           | Relevance                                                   |
| ---------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| [TypeScript ESM Guide](https://www.typescriptlang.org/docs/handbook/esm-node.html) | Understanding `.js` extension imports and module resolution |
| [Zod Documentation](https://zod.dev/)                                              | Config validation patterns used throughout OpenClaw         |
| [Vitest Documentation](https://vitest.dev/)                                        | Testing framework with forks pool, coverage thresholds      |
| [pnpm Workspaces](https://pnpm.io/workspaces)                                      | Monorepo management for the extensions architecture         |
| [Commander.js](https://github.com/tj/commander.js)                                 | CLI framework patterns (builder composition)                |
| [tslog](https://tslog.js.org/)                                                     | Structured logging with child loggers and transports        |
| [Node.js ESM Loaders](https://nodejs.org/api/esm.html)                             | Deep understanding of dynamic imports and lazy loading      |
| [Oxlint](https://oxc.rs/docs/guide/usage/linter.html)                              | Rust-based linting (faster alternative to ESLint)           |

---

> **Generated:** 2026-03-01 | **Source:** Static analysis of the Heavy02011/openclaw repository
