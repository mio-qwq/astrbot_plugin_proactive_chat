# Audit Dossier — astrbot_plugin_proactive_chat

Status: evidence collection only; no application code changes.
Audited baseline: `77a1d408ccd003a9d34ad4f51acfb8fc17bc35f5` (fork main and upstream main matched at audit start).
Scope: static review of backend, FastAPI console, frontend config editor, telemetry/notification clients, CI/release workflows, and upstream PR #114 review evidence.
Findings are static-review findings. Do not call them exploited or runtime-reproduced unless a later test confirms that.

## Executive summary

This is not a tiny throwaway plugin: it has modular backend boundaries, scheduling and persistence, a separate web admin, notification polling, telemetry, and a frontend configuration editor. The main engineering risk is the mismatch between architectural ambition and the lack of an evident checked-in automated test suite. Several concrete correctness and privacy concerns merit validation before a rewrite.

No application source code was modified during this audit pass.

## Evidence ledger

### A-01 — Telemetry is enabled by default
- Severity: High (privacy/consent design; not necessarily a security vulnerability)
- Confidence: Confirmed in source
- Evidence: `core/remote/telemetry_manager.py` sets `self._enabled = telemetry_config.get("enabled", True)`; `_conf_schema.json` also declares default true.
- Behavior: Persistent `.telemetry_id` UUID; sends startup/heartbeat/shutdown, feature events, config snapshots, and error details to `https://plugincenter.aloys23.link/api/ingest`.
- Mitigations present: Some UMO/path/key/prompt redaction; friend/group session lists become counts; top-level `proactive_prompt` and `web_admin.password` are removed from config snapshots.
- Next: Validate event payloads, remote retention/disclosure, and opt-out UX.

### A-02 — Config snapshot redaction appears incomplete for nested prompts
- Severity: High (potential privacy leak)
- Confidence: Strong static finding; payload must be tested
- Evidence: `track_config()` deletes `friend_settings.proactive_prompt` and `group_settings.proactive_prompt`, but does not explicitly delete nested `context_settings.platform_history_prompt` in either group. Both fields exist in `_conf_schema.json`.
- Impact: Custom prompt text could be included in a config snapshot unless later sanitization removes it. The structured config path does not recursively filter prompt fields.
- Next: Unit-test payload generation with sentinel strings at every prompt-bearing path and assert no sentinel reaches `track()`.

### A-03 — Telemetry toggle is not synchronized with the live manager
- Severity: Medium (privacy control correctness)
- Confidence: Confirmed by static review; independently reported in upstream PR #114 review
- Evidence: `TelemetryManager` caches `_enabled` in `__init__`; `POST /api/config` updates only `friend_settings`, `group_settings`, and `web_admin`, not `telemetry_config`.
- Impact: Runtime configuration changes do not propagate to the manager; the current API save contract omits the telemetry flag.
- Next: Test UI fields and API contract, then cover enable/disable lifecycle.

### A-04 — Global config editor drops notification and telemetry settings
- Severity: Medium (configuration persistence bug)
- Confidence: Confirmed in frontend/backend contract
- Evidence: `admin/js/components/config/ConfigRenderer.jsx` global save payload contains only `friend_settings`, `group_settings`, `web_admin`; `core/console/api_routes.py` allows only those keys. GET `/api/config` returns `notification_settings`, and schema declares both notification and telemetry settings.
- Impact: These settings cannot be persisted through this global editor path; UI may display inconsistently or not expose them.
- Next: API/UI contract tests and inspect full render path.

### A-05 — Manual-trigger guard is synchronous, so the suspected race is not established
- Severity: Withdrawn pending new evidence
- Confidence: Initial concern falsified by control-flow inspection
- Evidence: The route checks `manual_trigger_sessions` and immediately adds the session without an intervening `await`. Under normal single-event-loop asyncio scheduling, another coroutine cannot interleave between those operations.
- Impact: Do not count this as a confirmed race. A lock is not justified solely by this snippet; test thread-level access or future edits only if relevant.
- Next: Keep as a negative finding; verify task creation and cleanup separately.

### A-06 — Manual-trigger cleanup is wired through the orchestrator
- Status: Not a confirmed bug; initial concern resolved
- Confidence: Confirmed by call-path inspection
- Evidence: `main.py::check_and_chat()` calls `FlowOrchestrator.run()`, whose `finally` calls `clear_manual_trigger_state(normalized_session_id)`.
- Caveat: The early `_terminating` return occurs before the `try/finally`, but shutdown is already in progress. This should be tested for stale UI state only if it affects restart/runtime behavior.
- Next: Remove from the bug list; retain a lifecycle regression test.

### A-07 — Session state writes are non-atomic
- Severity: Medium (durability)
- Confidence: Confirmed
- Evidence: `core/store/session_store.py` opens `session_data.json` with `"w"` and writes directly. `core/store/override_store.py` instead writes a temporary file then replaces the destination.
- Impact: Process termination or disk failure mid-write can leave truncated JSON; startup then falls back to empty state after parse failure.
- Next: Fault-injection test; consider temp-file + fsync/replace.

### A-08 — Shared lock discipline remains a verification task, not a confirmed defect
- Severity: Unrated pending complete inventory
- Confidence: Initial concern weakened by call-site review
- Evidence: The inspected `save_data()` call sites mostly show an enclosing `plugin.data_lock`; the service explicitly documents that the store save is an unlocked internal operation whose callers own synchronization.
- Impact: No concrete unlocked mutation/save race has been demonstrated so far. Do not claim a bug until every caller and mutation path has been inventoried.
- Next: Build a complete mutation/lock ownership map before refactoring; retain as an audit task, not a confirmed finding.

### A-09 — WebSocket heartbeat/reconnect has known edge cases in upstream PR #114
- Severity: Medium (admin UI reliability)
- Confidence: Reported upstream; baseline still needs independent re-check
- Evidence: PR #114 review reports reconnect can fail after heartbeat send errors while socket remains OPEN; missing pong timeout may leave half-open connection marked connected.
- Impact: Admin UI can show connected state while updates are stale.
- Next: Verify against baseline `admin/js/hooks/useWebSocket.js`; test send errors, missing pong, close races and unmount/reconnect.

### A-10 — Config editor stale-request bugs reported against upstream PR #114
- Severity: Medium (admin correctness)
- Confidence: Not attributed to current baseline
- Evidence: PR #114 review reports stale config requests affecting expanded state/dirty flags, mobile scroll restoration, and missing mode switch when config is null. PR #114 is open and unmerged at audit time.
- Impact: Relevant if adopting that refactor, not established as present in current baseline.
- Next: Compare each issue with baseline and PR diff before cherry-picking.

### A-11 — Notification polling is enabled by default and contacts a third-party service
- Severity: Medium (privacy/supply-chain review)
- Confidence: Confirmed
- Evidence: `core/remote/notification_center.py` uses `https://plugincenter.aloys23.link`; `is_enabled()` defaults to true; startup refreshes and starts polling, default 300 seconds. URL includes plugin version and fixed app slug.
- Impact: Outbound network dependency exists even when user only wants local proactive-chat functionality; separate from telemetry.
- Next: Document exact payload, service behavior and failure modes; consider opt-in/disable-by-default.

### A-12 — Web admin can intentionally run without authentication
- Severity: High if exposed beyond localhost; configuration-dependent
- Confidence: Confirmed
- Evidence: `_conf_schema.json` defaults host to `127.0.0.1`, password to empty; `runtime_server.py` bypasses auth when no password is configured; schema says `0.0.0.0` enables LAN access.
- Impact: Binding to all interfaces with an empty password exposes privileged management APIs without authentication.
- Next: Test host/auth combinations; warn or require auth for non-loopback binds.

### A-13 — WebSocket token may be supplied in query string
- Severity: Low–Medium (credential handling)
- Confidence: Confirmed
- Evidence: `core/console/api_routes.py` reads `websocket.query_params.get("token")` before Authorization-header fallback.
- Impact: Query tokens may enter proxy/access logs or diagnostic URLs depending on deployment.
- Next: Prefer header where possible and avoid query-string logging.

### A-14 — Frontend dependency loading relies on external CDNs
- Severity: Medium (availability/supply chain)
- Confidence: Confirmed
- Evidence: `admin/index.html` loads React, ReactDOM, Babel, MUI, Emotion, marked, DOMPurify and Mermaid from npmmirror or unpkg selected by timezone/language. No SRI attributes visible in inspected bootstrap.
- Impact: Admin startup depends on third-party network/CDN availability and trust; mirror heuristic can fail.
- Next: Test offline/CDN failure; consider vendored pinned assets, SRI, CSP and removing runtime Babel.

### A-15 — Release workflow relies on source-text substitutions
- Severity: Low–Medium (release maintainability)
- Confidence: Confirmed workflow design
- Evidence: `.github/workflows/release.yml` stages a copy, Babel-compiles `admin/js`, removes source files, moves generated JS into place, then rewrites `admin/index.html` using `sed` substitutions. Sanity checks exist, but no browser smoke test was visible.
- Impact: Build correctness depends on expected source strings/resource paths.
- Next: Package and browser smoke test.

### A-16 — Automated tests are not evident in the checked-in tree
- Severity: High (engineering confidence, not a direct runtime bug)
- Confidence: Confirmed for audited tree listing
- Evidence: Recursive tree has no obvious `tests/`, `test_*.py`, or frontend test directory. CI includes Ruff and release packaging; no test workflow was visible.
- Impact: Scheduling, persistence, LLM integration, FastAPI and UI changes have little visible regression protection.
- Next: Characterization tests before refactor.

### A-17 — Reschedule state/job ordering differs between code paths
- Severity: Medium (potential task/state inconsistency)
- Confidence: Needs reproduction
- Evidence: `schedule_next_chat_and_save()` registers job and updates persistent state while holding lock; `FlowOrchestrator.finalize_and_reschedule()` persists state while locked, then registers job after releasing lock.
- Impact: Crash between persistence and registration requires recovery; concurrent operations may interleave. May be recoverable by design, not yet a confirmed bug.
- Next: Define invariants and inject failures around save/add_job/restart.

### A-18 — Override validation restricts root keys but not nested value types
- Severity: Low–Medium (configuration validation)
- Confidence: Confirmed
- Evidence: `core/store/override_store.py` allowlists root keys and recursively copies nested values without validating against `_conf_schema.json`.
- Impact: Malformed nested values can break runtime code expecting numbers, lists or mappings.
- Next: Schema-driven validation for types, bounds, unknown nested keys, non-finite numbers, regex and interval formats.

### A-19 — Some API errors expose local paths and raw exception details
- Severity: Low–Medium (information exposure; depends on admin exposure)
- Confidence: Confirmed in inspected routes
- Evidence: Markdown-read failures return `str(e)`; open-directory errors return command output and absolute plugin/data paths.
- Impact: Exposes environment details to API clients if admin is reachable by untrusted users.
- Next: Generic client errors, detailed server logs.

### A-20 — Markdown fallback renderer deserves dedicated XSS review
- Severity: Medium (unverified concern; not a confirmed XSS)
- Confidence: Needs proof
- Evidence: `admin/js/utils/markdown.js` uses DOMPurify when marked and DOMPurify are available; if parsing throws or libraries are absent, it falls back to custom renderer that emits HTML. Ordinary text escaping and link filtering exist, but Mermaid/code paths need review.
- Impact: If fallback transformation reintroduces untrusted HTML, remote notification Markdown or local docs may become a DOM injection vector.
- Next: Browser hostile-Markdown corpus covering javascript/data URLs, malformed HTML, SVG, attributes, code fences, Mermaid and library-load failures.

## Positive evidence

- Backend has recognizable boundaries: adapters, domain, features, pipeline, services, stores, console, remote.
- Token generation uses `secrets.token_urlsafe`; password comparison uses `secrets.compare_digest`.
- Override storage uses temporary file then replace.
- Session IDs and Markdown document paths have explicit normalization/containment checks.
- Telemetry redaction is deliberate, though structured config filtering appears incomplete.
- Release workflow pins analyzer SHA-256 and checks stale JSX/Babel references.
- CI includes Ruff and release packaging.

## Test gaps / validation plan

1. Unit tests for telemetry payload construction and redaction using sentinel strings.
2. API tests for auth modes, trigger races, config save contract and WebSocket auth.
3. Scheduler/property tests for reschedule invariants, concurrent messages, restart recovery and timer cancellation.
4. Storage fault injection for truncated writes, invalid JSON and concurrent saves.
5. Frontend tests for config session-switch races, WebSocket reconnect/half-open, Markdown XSS corpus and CDN failure.
6. Release smoke test that installs/starts packaged plugin against supported AstrBot versions.

## Refactor direction (proposal only)

- Phase 0: Freeze behavior with characterization tests and dependency map.
- Phase 1: Privacy/security hardening: telemetry off by default, recursive redaction, effective opt-out, and explicit notification networking.
- Phase 2: Correctness fixes: atomic persistence, per-session trigger serialization, guaranteed cleanup in `finally`, API/schema consistency.
- Phase 3: Runtime design: explicit per-session state machine and scheduling invariants; isolate clock/scheduler/storage behind protocols.
- Phase 4: Simplify UI boot: vendor or integrity-pin dependencies; replace regex HTML generation where possible; test WebSocket lifecycle.
- Phase 5: Re-evaluate abstractions and keep only boundaries that reduce coupling.

## Scope boundary

This dossier is a working audit record, not a final security certification. No production source files were changed. Findings marked 'needs validation' must be reproduced or falsified before implementation.

### A-21 — Session-state loading validates only the root JSON type
- Severity: Medium (robustness / recovery)
- Confidence: Confirmed
- Evidence: `core/store/session_store.py::load()` accepts any JSON object as the entire session store. It does not validate that every value is a mapping or that fields have expected types before returning it.
- Impact: A partially malformed but syntactically valid file can later break normalization/scheduler paths that assume each session payload is a dict. The current exception handling catches JSON parse and OS errors, not downstream shape errors.
- Next: Add per-entry schema validation and tests with lists/scalars/missing fields/invalid numeric types.

### A-22 — Login endpoint has no visible brute-force throttling
- Severity: Low–Medium (deployment-dependent)
- Confidence: Confirmed in inspected route; exploitability depends on network exposure
- Evidence: `POST /api/login` compares the supplied password using `secrets.compare_digest` and issues a token on success, but the inspected route has no per-IP or global rate limit/backoff.
- Impact: If the console is reachable from a LAN or other untrusted network, password guessing is not rate-limited at the application layer.
- Next: Test deployment exposure; add rate limiting or document reverse-proxy protections if remote access is supported.

### A-23 — WebSocket token is exposed to browser URL construction and may leak in infrastructure logs
- Severity: Low–Medium
- Confidence: Confirmed
- Evidence: `admin/js/hooks/useWebSocket.js` appends the bearer token as `?token=...`; server accepts query token first.
- Impact: Reverse proxies, debug tooling, or URL logging can record the token. The default server disables Uvicorn access logs, which reduces but does not eliminate deployment-specific risk.
- Next: Prefer Authorization header for WebSocket where clients/proxies support it, or ensure query strings are redacted at every proxy/logging layer.

### A-24 — Custom regex configuration can cause excessive CPU during message segmentation
- Severity: Low–Medium (user-controlled configuration; denial-of-service risk)
- Confidence: Plausible, not reproduced
- Evidence: `core/services/sender_service.py::split_text()` compiles and runs a user-configurable regular expression with `re.findall` over the full response. Invalid regex syntax is caught, but catastrophic backtracking is not bounded by Python's standard `re` API.
- Impact: A pathological configured pattern plus long model output may block the event loop while regex evaluation runs.
- Next: Reproduce with bounded local test inputs; cap text length and/or use a timeout-capable regex engine or simpler splitting modes.

### A-25 — Config editor's numeric input can persist transient empty/invalid values inconsistently
- Severity: Low (UI correctness)
- Confidence: Confirmed behavior in source; impact needs UI test
- Evidence: `ConfigRenderer.jsx` allows `''` and `'-'` into local state without calling `handleChange`, while the parent config object retains the prior numeric value. On blur/save, displayed input and saved value can disagree.
- Impact: A user can see an empty or incomplete number while the persisted draft still contains an older value; behavior is confusing and can cause accidental saves.
- Next: Add component tests for clearing, typing a minus sign, blur, switching sessions, and save.

### A-26 — Global config schema and API contract are structurally inconsistent
- Severity: Medium (configuration correctness / maintainability)
- Confidence: Confirmed
- Evidence: The schema contains top-level `friend_settings`, `group_settings`, `web_admin`, `notification_settings`, and `telemetry_config`; GET config omits telemetry, while POST accepts only the first three groups. The renderer uses all top-level schema entries in global mode.
- Impact: Some schema-described fields can be rendered with defaults but cannot be loaded or persisted through the matching API. This is broader than a single missed checkbox and indicates schema/API drift.
- Next: Define one canonical configuration contract and test every schema root key against GET/POST behavior.


## Upstream history and regression signals

The public upstream issue/PR history was reviewed as supporting context. These are not proof that current code is broken, but they help prioritize regression coverage:

- Issue #103 reports a v1.2.5 release ZIP whose WebUI showed a blank page because a referenced JS file returned 404; replacing the bundled admin directory fixed it. This is direct evidence that release packaging needs browser smoke tests, not only source-tree checks.
- Issue #105 reports persisted scheduling tasks failing to restore when plugin startup happened before a custom-ID platform adapter initialized. The current lifecycle code contains delayed startup/retry logic, but this scenario deserves a regression test.
- Issue #88 reports AstrBot event handler signature incompatibility after a host upgrade; the adapter boundary should be tested against supported host versions.
- Issue #75 reports host process crash around configuration hot reload/port conflict; current server has port probing, but hot-reload behavior still deserves an integration test.
- Issue #69/#68/#57/#66 show historical dependency and FastAPI/Starlette compatibility failures. The current broad lower bounds in `requirements.txt` make a supported dependency matrix and upper-bound policy worth considering.
- PR #114 is an open, unmerged WebUI refactor. Review comments reported stale configuration request state, WebSocket heartbeat/reconnect edge cases, mobile scroll restoration, and cache headers. These are **PR-only reports unless verified on the audited baseline**.
- The current CI run for the audited baseline showed Ruff and release packaging workflows, but no automated test workflow was visible. The repo's recent major refactor was very large, making characterization tests especially important.

## Architecture assessment

The architecture is not simply random spaghetti. The backend has coherent module categories and recognizable seams. However, the current abstractions are not yet demonstrated by tests or contracts:

1. `ProactiveChatPlugin` is effectively a service container and shared mutable-state owner; this makes cross-service coupling easy and synchronization assumptions implicit.
2. The pipeline is extensible at the Stage/Feature level, but default features reach back through the orchestrator into the whole plugin container. That limits true isolation and makes unit testing require large mocks.
3. Scheduler, persistence, message history and UI state share session identifiers and implicit timing invariants. Those invariants are spread across event handlers, scheduler methods, and the orchestrator rather than defined in one state machine.
4. Configuration schema, API payloads and UI save logic are maintained separately and have already drifted (A-04/A-26).
5. Remote notifications, telemetry, and core proactive-chat behavior are constructed together during plugin initialization. Optional remote features should not make the local core harder to reason about.
6. The UI is built around global browser objects and manually queued scripts rather than a conventional module graph. This can work, but increases reliance on load ordering, generated artifacts and hand-maintained string rewrites.

### Recommended refactor target (not implemented)

- Define typed domain models for session state, configuration, trigger result, and delivery result.
- Give each session an explicit state machine with idempotent transitions for: idle → scheduled → generating → review → delivering → persisted/rescheduled, plus cancellation/failure.
- Isolate scheduler and clock behind interfaces so time and races can be tested deterministically.
- Make persistence a single owner of serialization and atomic replacement; do not let arbitrary services write raw mutable dictionaries.
- Make configuration schema the source of truth for validation and API contract; add round-trip tests for every top-level group.
- Make telemetry and remote notifications independent opt-in services with explicit data inventory, no prompt/config secrets by default, and runtime-safe disable semantics.
- Keep a modular architecture only where module boundaries can be tested independently; avoid adding more abstractions until each one has an enforceable contract.

## Findings that were explicitly downgraded or withdrawn

- The suspected manual-trigger check/set race (A-05) is not supported by the current code path because there is no `await` between check and add in a single asyncio event loop.
- The suspected missing manual-trigger cleanup (A-06) is resolved by the orchestrator's `finally` block.
- The shared-lock concern (A-08) remains a code-audit task; current call-site evidence does not establish an unlocked save race.
- Markdown XSS (A-20), regex ReDoS (A-24), and reschedule-ordering issues (A-17) remain hypotheses requiring targeted reproduction, not confirmed vulnerabilities.
