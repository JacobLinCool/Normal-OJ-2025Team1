# Normal-OJ 2025 Team 1 — Codebase Review (2025-12-29)

## Scope & Method
- Repository snapshot only contains documentation; the three submodules (`Back-End`, `Sandbox`, `new-front-end`) are not checked out here. Findings below are based on the in-repo analysis docs (e.g., `Docs and Ref/ARCHITECTURE_ANALYSIS.md`, DevNotes) that reference the upstream source code.
- No automated tests or linters were run in this workspace because the source trees are absent; testing status is based on documented coverage figures.

## Repository State
- Monorepo intended to host Backend (Python), Sandbox (Python), and Frontend (Vue) via submodules; current checkout lacks their source, so build/test cannot be executed.
- Documentation is well organized (`Docs and Ref`) and already contains deep dives (architecture, flows, guides, dev notes).

## Critical / High Findings (from documented code review)
1. **Duplicate `Problem` classes** (`Back-End/model/problem.py` vs `Back-End/mongo/problem/problem.py`) with overlapping responsibilities. Risks: maintenance overhead, behavior divergence. *Recommendation:* split API vs ORM naming or unify into a single domain layer.
2. **Config field alias confusion** where legacy misspellings of the static-analysis and scoring-script keys are still persisted alongside the correct spellings in DB and pipeline configs. Risks: data inconsistency, migration hazards. *Recommendation:* migration script to normalize, deprecate legacy aliases, validate via schema (e.g., Pydantic).
3. **Submission directories committed with bad permissions** under `Sandbox/submissions/` and `Back-End/submissions/`. Risks: leaked test data, cleanup failures. *Recommendation:* purge, gitignore, move to configurable tmp path.
4. **Missing scan ignore for Sandbox** (`.antigravityignore` absent). Risks: slow scans, accidental leakage of caches/logs. *Recommendation:* add ignore list for caches, logs, submissions.
5. **Oversized monolithic modules** (e.g., `Sandbox/dispatcher/dispatcher.py` ~1200 lines, `Back-End/model/problem.py` ~1100 lines, `Back-End/mongo/submission.py` ~1600 lines). Risks: high complexity, merge conflicts. *Recommendation:* modularize by responsibility (build/execute/result, asset handling, etc.).
6. **Configuration scattering** across multiple files (`Back-End/mongo/config.py`, `Sandbox/dispatcher/config.py`, `docker-compose*.yml`, secrets). Risks: drift between environments. *Recommendation:* centralized settings (e.g., pydantic-settings) with env-based layering.
7. **Dockerfile duplication** (7 variants across Sandbox/Backend) with shared base layers. Risks: maintenance overhead, inconsistent hardening. *Recommendation:* multi-stage or Compose `extends` to consolidate shared layers.
8. **Type hints and error handling inconsistency.** Many public functions lack annotations; mixed use of exceptions vs HTTPError vs silent logging. *Recommendation:* add typing to public APIs first; define error taxonomy and map to HTTP responses centrally.
9. **Low/uneven test coverage** (Backend ~60%, Sandbox ~50%, Frontend unknown). Critical areas lacking coverage: dispatcher build strategy, `mongo.problem` asset mgmt, interactive mode errors, custom scorer/checker. *Recommendation:* prioritize tests around dispatcher flow, asset uploads, interactive sandboxing, and custom scoring.

## Medium / Low Findings (selected)
- **Debug artifacts & magic values** remain in dispatcher/static analysis modules; logging noise and magic numbers for language/status codes (e.g., `language >= 3` guard, status ints `0=AC`, `1=WA`, `2=TLE`, and a 1 GiB output limit expressed as `1024 ** 3`). Normalize via enums/constants and prune debug-only comments.
- **Import and naming hygiene**: `from X import *` causing cycle risk; mixed snake_case/camelCase between API and DB layers. Prefer explicit imports and alias handling via serialization models.
- **Hardcoded paths** for assets/submissions; should be centralized in path helpers with env overrides.
- **Docs gaps**: API lacks generated OpenAPI/Swagger despite rich markdown docs; would aid clients/QA.
- **Dependency drift**: parallel use of Poetry and `requirements.txt`; converge on a single tool and prune unused files/logs.
- **Submodule strategy**: `.gitmodules` lists Backend/Sandbox/frontend though they are core components. Decide between true multi-repo or full monorepo to simplify CI.

## Security & Operations
- Documented network/file controls exist, but without code present, enforcement cannot be verified. Ensure Sandbox images:
  - drop unneeded capabilities (e.g., `CAP_NET_RAW`, `CAP_SYS_ADMIN`);
  - apply an appropriate seccomp profile;
  - mount submissions/tmp as `noexec,nodev,nosuid`;
  - keep outbound network disabled except for whitelisted local services.
  Mirror these in any consolidated Dockerfiles.
- Logging and artifact directories should be gitignored and rotated; current docs note large `gunicorn_error.log`.

## Recommendations & Next Steps
1. Check out/populate submodules, then re-run lint/test to validate the documented issues.
2. Execute the migration/cleanup items: config alias normalization, submissions directory relocation, add Sandbox ignore file.
3. Begin modularization of the largest files (dispatcher, problem models) with clear ownership boundaries.
4. Establish unified settings management and Docker build strategy to reduce drift.
5. Expand automated tests around dispatcher/build/interactive flows and asset handling; add mypy/flake8 and a minimal pre-commit suite.
6. Generate OpenAPI specs from the Backend routes to align API/DB naming and improve client confidence.

## Testing Note
- Not run here: source code for Backend/Sandbox/Frontend is not present in this checkout, so no commands could be executed. Re-run unit/e2e suites once submodules are available.
