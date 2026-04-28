# BASELINE.md

Cross-project baseline for Claude Code. Defines the principles, workflow, and standards that apply regardless of domain or stack. Copy into any new project and adapt where necessary.

---

## Project Initiation

When starting a new project, the first task before writing any code is to create `SKILL.md` in the project root. This is not optional.

`SKILL.md` must cover:
- What the project does and the core domain concepts a developer needs to understand
- Non-obvious design decisions and why they were made
- Data flow through the system at a high level
- Common pitfalls specific to this codebase
- Any planned growth areas that should influence current decisions

Ask the user questions to fill gaps that cannot be derived from the code. Once `SKILL.md` exists, add a reference to it in `CLAUDE.md`. The presence of `SKILL.md` is the signal that the project has been properly initiated.

---

## Agent Workflow

Work in this loop without exception:

```
observe → plan → implement → test → reflect
```

1. **Observe** — Read the relevant files. Never assume structure or content.
2. **Plan** — Form a short, explicit plan before writing code. State it.
3. **Implement** — Make the minimal change that satisfies the requirement.
4. **Test** — Run tests. Fix failures. Do not proceed until they pass.
5. **Reflect** — Consider whether the change introduced unintended side effects.

If something fails, diagnose and change strategy. Do not retry the same failing approach.

---

## Before Writing Any Code

- Read every file you intend to modify.
- Search for existing patterns and utilities that solve the same problem.
- Understand what a function does before changing it — not just its name.
- On an existing project: work within the established structure. Do not reorganise or rename modules as a side effect of a task.
- On a new project: follow the Project Structure section to establish the layout before writing any logic.

---

## Minimal Change Principle

Make the smallest change that satisfies the requirement. Specifically:

- Do not refactor surrounding code unless it is directly blocking the task.
- Do not add features, options, or abstractions for hypothetical future needs.
- Do not add comments, docstrings, or type annotations to code you did not write or modify.
- Duplication is preferable to a premature abstraction — three similar lines is not a reason to introduce a helper. Exception: configuration and artifact naming must always be centralised (see Configuration Management and Deterministic Artifact Naming) — those are prescribed patterns with known long-term cost if scattered.

**On type hints:** Apply type hints to all new code you write and to any existing function you modify. Do not retrofit type hints to untouched code in the same file — that is a separate task.

---

## Project Structure

### Use the `src/` layout

Always place the package under `src/package_name/`, not directly at the project root:

```
project_root/
  src/
    my_package/
      __init__.py
      ...
  tests/
  pyproject.toml
  environment.yml
```

**Why this matters:** Without `src/`, Python can import directly from the project root during development, masking packaging errors that only surface in production. The `src/` layout forces an explicit install (`pip install -e .`), ensuring your installed package is always what's being tested.

### Directory structure mirrors the layer model

Each architectural layer gets its own directory. The file tree should make the architecture readable without opening any files:

```
src/my_package/
  pipelines/       # orchestration
  services/        # external API wrappers
  transforms/      # pure data transformations
  repositories/    # database access (ORM models live here or in a models/ sub-package)
  utils/           # generic helpers
  db.py            # database engine/session setup
  cli.py           # CLI entry point
```

ORM model definitions belong in the repositories layer — either inline in `repositories/` or in a `models/` sub-package beneath it. They are not a separate architectural layer.

Do not create directories that don't correspond to a layer. Do not put code in the wrong layer's directory because it's convenient.

### Test structure

```
tests/
  integration/     # tests that require real infrastructure (DB, filesystem)
  test_*.py        # unit tests, one file per source module
```

- Unit test files should mirror the source module they cover: `src/my_package/utils/vectors.py` → `tests/test_utils_vectors.py`
- Integration tests live in `tests/integration/` and are clearly separated — they may be slower or require setup
- Artefacts, outputs, and checkpoints live outside `src/` and `tests/` (e.g., `artefacts/`) and should be in `.gitignore`

---

## Architecture Discipline

Every project must have a clear layer model. Identify it early. Respect it throughout.

**Common Python layer order for pipelines:**

```
pipelines → services → transforms → repositories → utils
```

- **Pipelines** — orchestration only. Minimal logic. Call other layers in sequence.
- **Services** — thin wrappers around external systems (APIs, queues, storage).
- **Transforms** — pure functions. No I/O. No side effects. Easy to test in isolation.
- **Repositories** — the only layer that touches the database.
- **Utils** — generic helpers with no domain knowledge.

If a function is hard to place in a layer, it is doing too much. Split it.

**On new files:** Create a new file when a concern genuinely belongs in a new module per the layer model. Do not create files for one-use utilities that belong in an existing module. The test: could this reasonably live in an existing file without violating the layer model? If yes, put it there.

---

## Code Quality Standards

- **Type hints** on all new and modified function signatures, return types, and non-obvious local variables.
- **Functions under ~40 lines** — treat this as a signal to consider splitting, not a hard rule. A clear 45-line function is better than two poorly named 20-line ones.
- **No hidden side effects** — a function that looks like a query should not write.
- **I/O boundaries must be explicit** — functions that read files, hit APIs, or write to DBs should make that obvious from their name and signature.
- **Domain-specific exception types** — prefer named exception classes over generic `RuntimeError` for errors that callers may want to handle distinctly. Generic exceptions are fine for truly unrecoverable errors.

---

## Error Handling

Apply error handling at boundaries; trust your own internal code.

- **At system boundaries** (user input, API responses, DB results, file reads): validate eagerly, fail loudly with clear messages.
- **Inside internal code**: do not add defensive checks for scenarios your own logic makes impossible. Over-defensive internal code obscures real bugs.
- **In tests**: expect and assert specific exception types, not just "an exception was raised".

These rules are complementary, not in tension: validate what comes in from outside; trust what you control.

---

## Configuration Management

Runtime configuration (model names, batch sizes, thresholds, credentials) must be centralised:

- Define all configuration in one place — a `config.py`, a Pydantic `Settings` model, or equivalent.
- Read from environment variables and `.env`. Never hardcode credentials or environment-specific values.
- Configuration should be injectable into pipeline functions so they can be tested without environment variables.
- Avoid scattering magic constants across pipeline files — a reviewer should be able to see the full configuration surface in one place.

---

## Logging

Use structured logging throughout. Do not use `print()` for operational output.

- Configure a logger per module: `logger = logging.getLogger(__name__)`
- Use appropriate levels: `debug` for diagnostic detail, `info` for progress, `warning` for recoverable issues, `error` for failures.
- Control log level via environment variable or CLI flag — never hardcode it.
- Log at the start and end of significant pipeline stages with enough context to reconstruct what ran.

---

## Resumability and Idempotency

For any pipeline that involves expensive computation or external calls:

- **Short-circuit if output already exists** — check for a cached artifact before running a stage. This is only safe when artifact naming is deterministic and captures all inputs that affect the output (see Deterministic Artifact Naming below).
- **Checkpoint before expensive steps** — save progress so a restart resumes rather than recomputes from zero.
- **Write checkpoints atomically** — write to a `.tmp` file, then rename to the final path. Never write directly to the final path mid-computation.
- **Writes must be idempotent** — delete-then-insert keyed by a run or experiment identifier is safer than upsert for most relational writes. Running the same pipeline twice should produce the same state, not doubled rows.

---

## Deterministic Artifact Naming

All pipeline outputs (checkpoints, indexes, exported files, model artifacts) must have deterministic names derived from their inputs:

- Centralise naming logic in a dedicated module (e.g., `utils/naming.py`).
- Derive names from the parameters that affect the output: model, run ID, tag, date, or domain equivalents.
- Never hardcode output filenames inline in pipeline code.
- **Treat naming functions as stable APIs.** Changing them silently invalidates all existing cached artifacts. If a naming change is necessary, treat it as a breaking change and document it.
- "Short-circuit on existing artifact" is only valid when the artifact name encodes all inputs — if a parameter that affects output is not in the name, the cache can return stale results.

---

## Testing Standards

- Do not call third-party APIs in tests (OpenAI, Pinecone, Stripe, etc.) — mock those service wrappers.
- For database access: use the lightest suitable alternative (e.g., SQLite) for logic tests. Where the test specifically targets database-dialect behaviour (stored procedures, vendor-specific syntax), use a real instance — a docker-compose fixture is the standard approach.
- Prefer fast, deterministic unit tests.
- Test the layer you changed in isolation. Pipeline stage functions should be testable without invoking the CLI.
- New behaviour → new test. Do not finish a task until tests cover the change.

### Quality Gate

Run this before considering any task complete:

```bash
make all    # or equivalent: test + lint + typecheck
```

Fix every warning and error. Do not suppress linting rules without a comment explaining why.

---

## CLI Entry Points

- Keep CLI argument parsing separate from orchestration logic. The orchestration function should be callable from both the CLI and from tests.
- Provide sensible defaults for all optional arguments.
- Document every argument with a help string.
- A project should have one CLI entry point. If a script and a CLI module both exist, they must call the same underlying functions — never duplicate orchestration logic between them.

---

## External API Wrappers (Services)

Every external API must be wrapped in a thin service module that:

- Handles client construction and credential loading from environment.
- Owns retry logic and backoff.
- Detects and handles known failure modes (rate limits, unexpected responses, partial failures).
- Exposes a clean interface so the rest of the codebase never imports the external SDK directly.

This makes services mockable in tests and isolates SDK-specific quirks from business logic.

---

## Database Access

- All SQL lives in the repository layer. No raw SQL in pipelines, transforms, or services.
- Write operations must be explicitly transactional — commit on success, rollback on failure.
- Validate that expected columns are present in fetched data before operating on it — fail fast with a clear message, not a KeyError mid-pipeline.
- Respect foreign key order on insert — parent records before child records.

---

## Environment and Dependency Setup

This project uses a **conda + pip hybrid** pattern. Understanding the split is essential:

| File | Responsibility |
|------|---------------|
| `environment.yml` | Pins Python version and invokes pip. Nothing else. |
| `pyproject.toml` | Defines all runtime and dev dependencies, tool config, and build metadata. |

`environment.yml` is intentionally minimal:

```yaml
dependencies:
  - python=3.11
  - pip
  - pip:
      - "-e .[dev]"
```

`-e .[dev]` installs the package in **editable mode** with dev extras. This means:
- Changes to `src/` take effect immediately without reinstalling.
- Both production dependencies (`[project.dependencies]`) and dev tools (`[project.optional-dependencies] dev`) are installed in one step.

### Where to add dependencies

- **New runtime dependency** → `[project.dependencies]` in `pyproject.toml`
- **New dev/test/tooling dependency** → `[project.optional-dependencies] dev` in `pyproject.toml`
- **Never add packages directly to `environment.yml`** — it bypasses version tracking in `pyproject.toml`.

After any change to `pyproject.toml` dependencies, re-run the install:

```bash
pip install -e .[dev]   # or: make setup
```

### Version pinning

Use `>=x.y` lower bounds with `<next-major` upper bounds for dependencies where breaking changes are likely (e.g., `pandas>=2.3,<3.0`). This is what "pin versions" means in this project: constrain the range, do not use exact pins (`==`). Exact pins belong in a lockfile, not in `pyproject.toml`.

### Toolchain configuration

All tool configuration lives in `pyproject.toml`:
- **ruff** — linting, line length, target Python version
- **mypy** — type checking strictness
- **pytest** — test paths, default flags

**Important:** mypy is configured permissively by default (`disallow_untyped_defs = false`). This means `make all` will not fail on untyped functions even though the coding standard requires type hints. If you tighten mypy settings, expect to fix existing violations before the gate is clean. Consider tightening incrementally: enable `disallow_untyped_defs = true` once the existing codebase is fully typed.

---

## Dependency Management

- Do not introduce a dependency for something the stdlib handles adequately.
- Add all dependencies to `pyproject.toml`. Never install ad-hoc.
- Use range constraints (`>=x.y,<next-major`), not exact pins. See Version Pinning above.

---

## Output Format for Code Changes

When delivering code changes, always provide:

1. **Brief plan** — what is changing and why.
2. **Files modified** — list them.
3. **Code changes** — the actual diff or full updated function.
4. **Tests added or updated** — what is now covered.
5. **Quality gate result** — confirm `make all` (or equivalent) passes.

---

## What Not to Do

- Do not skip pre-commit hooks or bypass linting to get past failures — fix the root cause.
- Do not use destructive git commands (`reset --hard`, `push --force`, `checkout .`) without explicit user instruction.
- Do not add error handling for scenarios your own code makes impossible.
- Do not design for hypothetical future requirements.
- Do not add backwards-compatibility shims for code you just deleted.
- Do not commit secrets, credentials, or `.env` files.
