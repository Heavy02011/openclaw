# PLAN_REVERSE

This is the **execution plan** paired with `CONTEXT_REVERSE.md`.
Treat both files together as an executable reconstruction spec.

- `CONTEXT_REVERSE.md` supplies domain truth and constraints.
- `PLAN_REVERSE.md` supplies sequence, milestones, and validation loops.

---

## 0) Execution contract

1. Load `CONTEXT_REVERSE.md` first.
2. Build only what is justified by the context model and acceptance criteria.
3. Preserve architecture boundaries (core orchestration vs plugin/extension ownership).
4. At each phase end, run the listed checks and record pass/fail before moving on.

---

## 1) Phase plan (regenerate whole repo)

## Phase A — Skeleton and toolchain

Goal: recreate the monorepo scaffold and baseline tooling contract.

Deliverables:

- Workspace root with package manager/runtime configuration.
- Root scripts for build/check/lint/test and docs checks.
- Baseline TypeScript, lint, test, and format configuration.

Validation:

- Root commands exist and resolve.
- Docs checks run clean:
  - `pnpm format:docs:check`
  - `pnpm lint:docs`

## Phase B — Core runtime and gateway spine

Goal: establish the minimal runnable core.

Deliverables:

- Core runtime/environment surfaces.
- Gateway process/server shell with routing entrypoints.
- Configuration loading/validation flow.
- CLI entrypoint and command wiring shell.

Validation:

- `pnpm build` succeeds.
- `pnpm check` passes baseline integrity checks.

## Phase C — Channel and provider framework

Goal: restore pluggable communication + model/provider integration seams.

Deliverables:

- Channel abstraction and routing contracts.
- Provider/model invocation contracts.
- At least one representative channel path and one provider path wired end-to-end.

Validation:

- Targeted tests for routing/message flow pass.
- `pnpm test` includes representative channel/provider cases.

## Phase D — Plugin SDK and extensions ecosystem

Goal: restore extension model and package boundaries.

Deliverables:

- Public SDK surface for plugin development.
- Extensions workspace with plugin/channel/provider packages.
- Registry/discovery wiring for loading extensions.

Validation:

- SDK and extension package build integration works.
- Boundary checks/lints pass under repo check lanes.

## Phase E — UI/apps/docs surfaces

Goal: restore non-core product surfaces that complete the repo.

Deliverables:

- `ui/` package baseline and integration points.
- `apps/` structure for platform companions.
- docs IA and runbooks for install/usage/troubleshooting.

Validation:

- docs checks pass.
- cross-surface build/check still green.

## Phase F — Hardening and parity pass

Goal: verify regenerated repo is functionally and structurally faithful.

Deliverables:

- Final parity audit against `CONTEXT_REVERSE.md` acceptance criteria.
- Gap list with severity and owner.
- Remediation loop until all critical gaps are closed.

Validation:

- `pnpm lint`
- `pnpm test`
- `pnpm build`
- `pnpm check`

---

## 2) Reflection loop (required at each phase)

For each phase, explicitly answer:

1. What was regenerated?
2. Which context constraints were satisfied?
3. What remains mismatched versus context?
4. Which evidence proves correctness (commands/tests/artifacts)?
5. What changes are needed before advancing?

If evidence is missing, do not mark phase complete.

---

## 3) How context retargeting works

To use this plan for another domain:

1. Edit only `CONTEXT_REVERSE.md` domain sections first.
2. Keep this plan structure unchanged.
3. Re-run phases with new domain constraints.
4. Produce a new parity audit against the updated context.

This separation makes `PLAN_REVERSE.md` reusable while `CONTEXT_REVERSE.md` becomes the domain-specific dial.

---

## 4) Correctness checklist (final gate)

Mark complete only when all items are true:

- [ ] Repo shape matches context-defined major surfaces.
- [ ] Architecture boundaries align with context boundary policy.
- [ ] Build/check/lint/test command contract is implemented and passing.
- [ ] Docs and operational runbooks exist and validate.
- [ ] Representative end-to-end flows prove runtime viability.
- [ ] Remaining gaps are non-critical or explicitly documented.
