---
name: lminaudier:golden-master
description: Augment test coverage on a legacy codebase using the golden master (approval testing) technique — capture current behaviour as snapshots, then use code coverage to find untested paths and expand inputs until the safety net is solid.
argument-hint: [file, class, function, or module to cover]
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

Help the user add characterization tests to a legacy codebase using the golden master technique (also called approval testing). The goal is not to test what the code *should* do, but to lock in what it *currently does* — creating a safety net before any refactoring or modification.

## Core principles

- **Capture behaviour, not intent.** Golden master tests record actual output. Bugs and all. They prevent *accidental* changes, not wrong behaviour.
- **Inputs drive coverage.** The quality of the safety net is determined by the diversity of inputs you exercise. Use code coverage to find the gaps.
- **Coarse-grained first.** Start at the highest useful entry point (a public method, a use case, an HTTP endpoint). Fine-grained unit tests come later once the code is safer to refactor.
- **Normalise non-determinism.** Timestamps, UUIDs, random values, and environment-specific paths must be stripped or replaced before comparing snapshots.

---

## Step 1 — Understand the target

Before writing any test, explore the area the user wants to cover.

1. Ask the user which component, class, function, or module they want to cover (if not already specified in the argument).
2. Read the target code. Understand:
   - What are the **entry points** (public methods, endpoints, CLI commands)?
   - What does it **return or output** (return value, stdout, files written, DB mutations, events emitted)?
   - What **parameters and states** does it depend on (arguments, injected dependencies, database state, environment variables)?
3. Identify what counts as "output" for snapshot purposes. Prefer the broadest observable output (serialised return value, rendered HTML, file content, log lines) over internal state.

If the codebase has existing tests, read them to understand naming conventions and testing infrastructure already in place.

---

## Step 2 — Choose or set up a snapshot mechanism

Determine how to store and compare golden master snapshots. Pick the simplest option that fits the tech stack:

- **Dedicated approval testing library** (e.g. `ApprovalTests` for Java/.NET/Python/C++, `jest --updateSnapshot` for JS/TS, `insta` for Rust, `cupaloy` for Go).
- **Inline snapshot** in the test assertion (e.g. Jest `toMatchInlineSnapshot`).
- **File-based snapshot**: serialise output to a file under `tests/snapshots/` (or equivalent) and assert byte-for-byte equality in the test.

If the user has no preference, recommend the simplest file-based approach — write output to a `.approved` file, compare against it in the test. This has zero dependencies.

Explain the approval workflow:
- First run: the `.approved` file does not exist — the test captures the output and saves it as the baseline. Review the baseline manually; commit it if it looks correct.
- Subsequent runs: the test compares fresh output against the committed `.approved` file. Any difference is a failure.

---

## Step 3 — Write the first golden master test

1. Write a minimal test that calls the entry point with a representative input and captures its output.
2. Normalise the output before saving:
   - Replace timestamps with a fixed placeholder (e.g. `"<TIMESTAMP>"`)
   - Replace UUIDs/random IDs with `"<ID>"`
   - Sort maps/sets if order is non-deterministic
   - Strip environment-specific paths
3. On first run, let the test generate the `.approved` snapshot. Show the snapshot to the user and ask them to confirm it looks like valid (if buggy) current behaviour before committing.
4. Run the test a second time to confirm it is green and stable.

---

## Step 4 — Measure coverage and expand inputs

This is the feedback loop that turns one test into a safety net.

1. **Run with coverage** (e.g. `pytest --cov`, `jest --coverage`, `go test -cover`, `mvn jacoco:report`). Show the user the coverage report for the target file/class.
2. **Identify uncovered lines and branches.** For each uncovered path, ask: what input or state would exercise it?
   - A missing `if/else` branch → add a test case where the condition is true (or false)
   - A missing `catch` / error path → inject a dependency that throws, or pass invalid input
   - A missing loop iteration → pass an empty collection and a multi-element collection
3. **Add a new test case** for each gap. Use the same snapshot mechanism.
4. Repeat: run coverage again, check what is still uncovered.
5. Stop when you reach a satisfactory threshold (typically 80–90% line + branch coverage on the target) or when remaining gaps are genuinely unreachable (dead code, defensive guards).

Present a short summary after each coverage run:
```
Coverage before: 34% (lines), 28% (branches)
Coverage after:  81% (lines), 74% (branches)
Remaining gaps:  lines 142–147 (error path — requires DB failure injection)
```

---

## Step 5 — Handle hard-to-reach paths

Some paths are only reachable through dependency injection, mocking, or environment manipulation:

- **External services**: inject a fake/stub that returns specific responses or throws errors.
- **Time-dependent logic**: inject a clock or freeze time using a library.
- **Database state**: seed the database with specific fixtures before each test case.
- **Environment variables / feature flags**: set them in test setup and tear them down after.

Suggest the minimal seam needed — avoid over-engineering fakes. A simple lambda or anonymous class that returns a hardcoded value is often enough for characterization tests.

---

## Step 6 — Review and commit

Before finishing:

1. Run the full test suite to confirm nothing is broken.
2. List all `.approved` snapshot files created. Remind the user these **must be committed** to source control — they are the specification.
3. Remind the user what the safety net enables and what it does not:
   - ✅ Detects unintended behaviour changes during refactoring
   - ✅ Documents what the code currently does
   - ❌ Does not prove the code is correct
   - ❌ Does not replace intent-driven unit tests for new logic

---

## Guardrails

- Never delete or overwrite an existing `.approved` snapshot without asking the user to confirm they have reviewed the diff.
- Never suppress non-determinism by making the production code deterministic — normalise at the test boundary instead.
- If a test is flaky (sometimes passes, sometimes fails), do not commit it. Find and fix the source of non-determinism first.
- Do not write approval tests for trivial getters/setters or pure data holders — focus on code that actually has logic.
- Keep each test case focused on one entry point and one scenario. Do not chain multiple calls in a single golden master test.
