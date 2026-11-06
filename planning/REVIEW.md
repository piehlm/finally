# Review: `planning/PLAN.md`

## Critical

1. **The new Section 13 makes the implementation contract self-contradictory.** `PLAN.md` says every agent uses it as the shared contract, but the appended review records unresolved questions, recommendations, and mutually exclusive alternatives as part of that contract. For example, it both specifies `previous price` and `change direction` in SSE and recommends dropping either/both; it specifies Docker Compose as optional and recommends removing it; and it offers several incompatible chart and volume choices. An agent cannot tell which statements are requirements. Move this material to `planning/REVIEW.md` (or a dated decision log), and amend the normative sections only after each decision is made.

2. **The frontend/backend API contract is still insufficient to implement compatible clients.** The endpoint table supplies only paths and one request fragment. It does not define response bodies for portfolio, watchlist, history, chat, or errors; nor does it define the SSE framing, event name, batching, JSON fields, timestamp/unit conventions, or reconnection/initial-state behavior. The newly added section identifies the gap but does not close it. Add versioned examples or schemas for every endpoint, including successful and rejected trades and per-action chat results, before parallel implementation begins.

3. **Trade/chat execution is not an atomic concurrency contract.** Manual trades, auto-executed chat trades, and snapshots can interleave against SQLite. “Same validation” does not say whether an LLM response’s actions are sequential, all-or-nothing, or independently committed, and no transaction/locking rule prevents two concurrent purchases from spending the same cash. Specify a serialized transactional trade service (including chat actions), its action ordering/failure behavior, SQLite busy/WAL policy, and snapshot timing. Use fixed-point money or an explicit rounding rule rather than leaving `REAL` arithmetic unspecified.

4. **The storage/deployment specification contradicts itself.** Section 11 runs `-v finally-data:/app/db`, which is a named volume; the next sentence says the project-root `db/` directory maps to that same path, which would require `-v ./db:/app/db`. Section 4 likewise calls `db/` a volume mount target. These approaches have different persistence and inspectability semantics. Choose one command, make the scripts and tests use it, and state where the database exists on the host.

## High

5. **Ticker eligibility and price availability are undefined, which blocks consistent market, trade, and watchlist implementations.** A user/LLM may add `PYPL` or an invalid symbol, but the simulator only promises seed prices for examples and Massive may not return a price promptly. Define normalization, validation/unknown-ticker behavior, how dynamically added simulator tickers get deterministic parameters, and whether a ticker without a current quote may be displayed or traded. Also explicitly define the priced universe as `watchlist ∪ open positions` so removing a held ticker does not make portfolio valuation fail.

6. **“Daily change” cannot be calculated from the stated data model.** The UI requires daily change %, while the cache contains only current and previous tick prices. Decide on a baseline (previous close, start-of-session, or seed price), provide it through initial REST and SSE payloads, and state session/reset behavior for simulator and Massive modes.

7. **The lifecycle model is ambiguous at the points users first interact with the app.** The plan says initialization is on startup “(or first request),” calls it lazy initialization, and has market data as an in-process task, but never establishes ordering. Define that schema/seed initialization and initial quote-cache population complete before API readiness, when polling starts/stops, what an SSE client sees initially, and what `/api/health` verifies. Also state the required single-process/single-uvicorn-worker deployment assumption; the in-memory cache is otherwise inconsistent across workers.

8. **LLM failures and persistence semantics are undefined.** The plan requires a complete response and persistence but does not say what happens on provider timeout, malformed structured output, unavailable API key, or an action validation failure. Define status/response behavior, whether user and/or assistant messages are stored, a timeout/retry policy, and the exact `actions` JSON schema. Mock mode also needs a documented input-to-output mapping; “deterministic” alone is not testable.

## Medium

9. **History retention/query semantics will eventually make the chart slow and tests nondeterministic.** Snapshots are written every 30 seconds and on each trade, but `/api/portfolio/history` has no ordering, range, cap, aggregation, or retention rule. Define these (including duplicate timestamp behavior) and the response’s price/time representation.

10. **The frontend technology and visualization requirements leave incompatible choices open.** “Canvas-based charting library preferred (Lightweight Charts or Recharts)” is inaccurate: Recharts renders SVG, while Lightweight Charts does not provide the requested treemap. Pick the libraries/approach—or explicitly assign the main price chart, P&L chart, and heatmap implementations—so dependency and rendering decisions do not diverge.

11. **Static-export routing is under-specified.** FastAPI must route API/SSE endpoints before static handling and decide how direct navigation/unknown paths behave. Confirm whether the app has only exported static routes, whether an SPA fallback is needed, and that `/api/*` never falls through to `index.html`.

12. **Test determinism is requested but not designed.** GBM, correlated moves, random events, wall-clock timestamps, external Massive calls, and a background task make the listed unit/E2E tests flaky unless the interface admits a seedable RNG, controllable clock, and fake market source. Add those seams and prescribe the E2E simulator configuration.

## Low

13. **The plan references an environment-specific skill instead of a portable implementation requirement.** Section 9 says to use `cerebras-inference`, while the new skill is only under `.claude/skills`; other agents/tools may not discover it. The plan should directly state the provider contract (model, `extra_body`, structured-output method, dependency versions) and keep the skill as optional guidance. The current LiteLLM example should also be verified against the pinned LiteLLM API before coding.

14. **The contributor automation has correctness issues.** `.claude/agents/change-reviewer.md` has a typo in its description and tells its subagent to run an unpinned `codex exec` command, which can recursively invoke the same review workflow and overwrite `planning/REVIEW.md`. Replace it with explicit review instructions or a non-recursive command, and define whether generated review files are intended to be committed.

15. **Operational details remain incomplete.** The start script claims it builds only when needed but no staleness rule is defined; `.env` loading alternates between “mounted” and `--env-file`; and `OPENROUTER_API_KEY` is called required even though mock mode is intended for keyless development/CI. State the behavior for mock mode and missing env files, and commit a safe `.env.example`.
