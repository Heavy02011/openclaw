# CONTEXT_REVERSE

This file is the **domain/context input** for regenerating the repository from scratch.
Use it with `PLAN_REVERSE.md` as a paired spec:

- `CONTEXT_REVERSE.md` = what to build (domain, boundaries, constraints, acceptance)
- `PLAN_REVERSE.md` = how to sequence the build and validation

---

## 1) Domain Profile (editable)

### 1.1 Product identity

- Project name: **OpenClaw**
- Product category: **personal AI assistant gateway**
- Primary value: **local-first assistant that connects one agent core to many messaging/platform channels**
- Deployment style: **single-user, self-hosted control plane + optional platform companions**

### 1.2 Core user outcomes

- Run an AI assistant locally.
- Connect existing channels (chat and device surfaces).
- Route inbound/outbound messages through a shared gateway.
- Extend behavior via plugin surfaces.

### 1.3 Runtime and language defaults

- Primary language: **TypeScript (ESM)**
- Runtime: **Node.js 22.19+ (Node 24 recommended)**
- Package manager/workspace: **pnpm monorepo**

---

## 2) Architecture Context (repo-derived)

### 2.1 Workspace shape

- Root package plus workspaces: `ui`, `packages/*`, `extensions/*`.
- Major top-level surfaces:
  - `src/` (core runtime, gateway, channels, providers, plugin SDK)
  - `extensions/` (channel/provider/plugin integrations)
  - `packages/` (shared packages and protocol support)
  - `ui/` (web UI package)
  - `apps/` (native app surfaces)
  - `docs/` (Mintlify documentation)
  - `scripts/` (build/check/test orchestration)

### 2.2 System invariants

- Core runtime remains plugin-agnostic; extension behavior enters via SDK/declared contracts.
- Gateway is orchestration/control plane; channel-specific logic lives in channel/plugin boundaries.
- Configuration and operational checks are script-driven and enforced through repository commands.

### 2.3 Scale snapshot (for regeneration scope)

- `src` TypeScript files: **~8707**
- `extensions/*` directories: **~135**
- `packages/*` directories: **~21**
- docs markdown/mdx files: **~672**

---

## 3) Build/Test/Quality Contract (repo-derived)

Use these as canonical reconstruction quality gates:

- Build: `pnpm build`
- Main check lane: `pnpm check`
- Lint: `pnpm lint`
- Tests: `pnpm test`
- Docs sanity lanes:
  - `pnpm format:docs:check`
  - `pnpm lint:docs`

For fast docs-only verification during iterative doc generation:

- `pnpm format:docs:check`
- `pnpm lint:docs`

---

## 4) Domain-Tunable Inputs (edit these to retarget)

To adapt this process to another domain, change this section first:

### 4.1 Replaceable product parameters

- `DOMAIN_NAME`: OpenClaw
- `DOMAIN_MISSION`: Local-first personal AI assistant across channels
- `PRIMARY_WORKFLOWS`:
  1. Onboard and configure runtime
  2. Send/receive channel messages
  3. Run agent tasks with tools
  4. Extend with plugins
- `PRIMARY_SURFACES`:
  - CLI
  - Gateway API/runtime
  - Channel integrations
  - Plugin SDK
  - Optional companion apps
- `NON_FUNCTIONAL_PRIORITIES`:
  - reliability
  - compatibility
  - security
  - low-friction local operations

### 4.2 Boundary policy template

- Keep orchestration core generic.
- Push domain-specific behaviors to extension/plugin boundaries.
- Keep public contracts explicit (CLI, config, protocol, SDK).

---

## 5) Regeneration Acceptance Criteria

A regenerated repository is considered equivalent at a practical engineering level when all are true:

1. Monorepo structure mirrors the same major surfaces (`src`, `extensions`, `packages`, `ui`, `apps`, `docs`, `scripts`).
2. Core/gateway/channel/plugin responsibilities follow the same boundary model.
3. Build/check/lint/test commands exist and run through repo scripts.
4. Docs and operational runbooks are present for install/use/maintenance.
5. Representative channel + plugin pathways are executable end-to-end.

---

## 6) Evidence sources used for this context

- `README.md`
- `AGENTS.md`
- `pnpm-workspace.yaml`
- `package.json` scripts section
- `REPO_REVERSE_ENGINEERING.md`
