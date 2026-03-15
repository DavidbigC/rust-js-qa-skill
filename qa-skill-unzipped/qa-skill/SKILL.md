---
name: qa
description: >
  Use when a Rust + TypeScript/JavaScript project needs test execution, regression verification,
  failing-test triage, or production-readiness QA (/full-qa, /run-regression, /fix-failing-tests,
  /write-unit-tests).
---

# QA Skill — Rust + TypeScript/JavaScript

Full quality assurance workflow for a full-stack project with a Rust backend and a JS/TS frontend.
Covers all four entry points: writing tests, running regression, fixing failures, and full QA.

---

## Project Layout Detection

Before running any workflow, orient yourself:

```
project/
├── Cargo.toml / Cargo.lock       ← Rust workspace root
├── src/                          ← Rust source
│   └── **/*.rs
├── tests/                        ← Rust integration tests
├── package.json                  ← JS/TS frontend root (or nested under /frontend, /web, /client)
├── src/ or app/                  ← TS/JS source
├── *.test.ts / *.spec.ts         ← unit tests
└── e2e/ or playwright.config.ts  ← E2E tests
```

Check for monorepo patterns: workspaces in `Cargo.toml`, `pnpm-workspace.yaml`, `turbo.json`.
Read `package.json` `scripts` block and `Makefile` targets — never guess test commands, always verify.

---

## Command Resolution (Required)

Before running JS/TS commands:

1. Detect package manager from lockfiles:
   - `pnpm-lock.yaml` -> `pnpm`
   - `yarn.lock` -> `yarn`
   - `package-lock.json` -> `npm`
2. Prefer project-defined scripts from `package.json` (`test`, `lint`, `typecheck`, `e2e`) over raw tool commands.
3. If a script is missing, ask the user before choosing a fallback command.

Use `<pm>` below as the detected package manager.

---

## Environment Preflight (for integration/E2E)

Before integration or E2E:

- Verify required env vars (`.env.example`, README, test config)
- Verify required services (DB/Redis/queues/mock servers) are running or seeded
- Verify test artifacts/fixtures required by integration tests exist
- If preflight fails, report blockers and stop before E2E

---

## Stack Reference

For detailed commands and patterns, read the relevant reference file:
- Rust specifics → `references/rust.md`
- JS/TS specifics → `references/js-ts.md`

---

## Workflows

---

### `/write-unit-tests`

Write missing unit tests for a target file or function.

**Steps:**
1. **Identify target** — ask user or infer from recent changes which file/module needs tests
2. **Read the target** — understand exports, logic branches, error paths, async flows
3. **Read neighboring tests** — find 1–2 existing test files in the same module; match conventions exactly (naming, imports, assertion style, mock patterns, fixture approach)
4. **List gaps** — enumerate untested paths: happy path, error/panic cases, boundary values, invalid input
5. **Write tests** — follow local conventions; do not introduce new test libraries
   - Rust: `#[cfg(test)]` mod at bottom of file, or separate file in same module; use `#[tokio::test]` for async
   - TS/JS: co-located `.test.ts` or `__tests__/`; use existing framework (Jest/Vitest/Mocha)
6. **Run only the new tests first** — targeted run before full suite
7. **Fix failures** — iterate until green; never skip or `#[ignore]` without explanation
8. **Report** — list: tests added, coverage improved, edge cases intentionally deferred and why

---

### `/run-regression`

Run the full existing test suite across both Rust and JS/TS layers.

**Steps:**
1. **Detect commands** — read `Cargo.toml`, `package.json` scripts, `Makefile`
2. **Run Rust suite:**
   ```bash
   cargo test --workspace 2>&1
   cargo clippy --workspace -- -D warnings 2>&1
   ```
3. **Run JS/TS suite:**
   ```bash
   # use detected package manager and existing scripts
   <pm> run lint
   <pm> run typecheck
   <pm> run build      # always verify production build compiles
   <pm> run test       # skip if no test script — note gap in report
   ```
   If `test` script is absent: report it as a gap, do not fail the regression — build + typecheck are the floor.
4. **Parse results** — count passed / failed / skipped per layer
5. **For each failure:** show test name + error, diagnose root cause (logic bug, env issue, flaky test, missing mock/fixture)
6. **Report** — see Report Format below

---

### `/fix-failing-tests`

Diagnose and fix a set of failing tests without breaking passing ones.

**Steps:**
1. **Run the suite** to get current failure list (or accept list from user)
2. **Triage failures** into:
   - **Real bugs** — code is wrong; fix the implementation
   - **Stale tests** — expectations outdated due to intentional behavior change; update tests
   - **Environment issues** — missing env vars, DB not seeded, port conflicts
   - **Flaky tests** — timing, ordering, or resource issues; fix root cause
3. **Fix in order:** real bugs → stale → env → flaky
4. **Re-run after each fix** — confirm no regressions introduced
5. **Never delete a test** to make it pass unless user explicitly approves
6. **Report** — what was fixed, how, and any remaining known issues

---

### `/full-qa`

Full production-readiness check. Runs everything in sequence and produces a single summary.

**Execution order:**

| Step | Layer | Command |
|------|-------|---------|
| 0 | Preflight | Validate env/services/fixtures for integration+E2E |
| 1 | Rust | `cargo fmt --check` |
| 2 | Rust | `cargo check --workspace` |
| 3 | Rust | `cargo clippy --workspace -- -D warnings` |
| 4 | Rust | `cargo test --workspace` |
| 5 | Rust (extended, optional) | `cargo test --workspace -- --include-ignored` |
| 6 | TS/JS | `<pm> run typecheck` |
| 7 | TS/JS | `<pm> run lint` |
| 8 | TS/JS | `<pm> run build` |
| 9 | TS/JS | `<pm> run test` (skip + note gap if script absent) |
| 10 | E2E | `<pm> run e2e` (or configured project E2E command) |

**Abort rules:**
- Clippy errors (step 3) → fix before continuing
- Rust unit test failures (step 4) → fix before running E2E
- Typecheck errors (step 6) → fix before running build or E2E
- Build failure (step 8) → fix before running E2E
- Preflight blockers (step 0) → resolve before integration/E2E
- Step 5 runs only when user asks for extended/nightly coverage
- E2E always runs last — only after all lower layers are green

After all steps, output the QA Summary Report.

---

## QA Summary Report Format

Always end any workflow with this structured report:

```
## QA Summary

### Rust
- Fmt:          ✅ clean      | ❌ N issues
- Clippy:       ✅ clean      | ❌ N warnings/errors
- Unit tests:   ✅ N passed   | ❌ N failed  | ⏭ N skipped
- Integration:  ✅ N passed   | ❌ N failed

### TypeScript / JavaScript
- Typecheck:    ✅ clean      | ❌ N errors
- Lint:         ✅ clean      | ❌ N issues
- Build:        ✅ clean      | ❌ failed    | ⚠️ no script
- Unit tests:   ✅ N passed   | ❌ N failed  | ⏭ N skipped | ⚠️ no test script

### E2E
- ✅ N passed  | ❌ N failed  | ⏭ N skipped

### Overall
- Status:      ✅ READY TO SHIP  |  ❌ NEEDS FIXES  |  ⚠️ WARNINGS ONLY
- Risk areas:  [modules with low coverage or known fragility]
- Deferred:    [tests skipped intentionally with reason]
```

---

## General Rules

- **Never modify tests to force them to pass** without user approval
- **Never introduce new dependencies** (crates, npm packages) without asking
- **Match existing conventions** — if the project uses `rstest`, keep it; if it uses Vitest, don't swap to Jest
- **Run targeted tests first**, full suite second — faster feedback loop
- **Read failure output fully** before diagnosing — never assume the cause
- **If the test command is unclear**, ask rather than guessing
- **Prefer project scripts** over direct tools (`<pm> run lint` over raw `eslint`, etc.)
