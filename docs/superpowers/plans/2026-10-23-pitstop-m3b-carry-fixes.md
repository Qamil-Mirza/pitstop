# Milestone 3b: fixes for the items carried from 3a

Companion to `2026-10-23-pitstop-m3b-watch-layers.md`. Tasks 1–4 of that plan take their requirements from the sections below. Each item has its own verdict:

- **FIX:** do exactly what it says.
- **DROP:** no change, with the reason recorded here.
- **KEEP CARRIED:** moves to the next carry list (plan Task 17).

Line numbers are from `main` at c98ba6b and may drift by a few lines.

Part A covers the server, data, SDK and alerts (plan Tasks 1 and 2). Part B covers the UI, build, Python tests, example and docs (plan Tasks 3 and 4).


# Part A: server, data, SDK and alerts

Part A tally: 21 fix, 13 drop, 1 keep carried.
Measured on a temporary store (none of ~/.pitstop or the repo's pitstop.db*). With the tests/server/test_scale.py seed (5,000 runs x 20 events):
- `change_series`: 7.2 ms, 402 queries.
- `build_graph` (30d): 8.4 ms, 3 queries.
- `board`: 3.3 ms, 28 queries for 3 workflows.

With 200 runs x 100 events and 500-character payloads: graph 53.5 ms, changes 31.6 ms.

## Server/data (src/pitstop/server/*, src/pitstop/cli.py)

### Polling cost: FIX (one batched metrics query; no response cache)
- **Real.** `changes.py:86` runs `metrics = run_metrics(store, r["id"])` for each run, which is 2 queries per run (402 per call). `graph.py:262-263` reads every event: `SELECT id, run_id, ts, agent, kind, name, payload, ...  FROM events WHERE run_id IN ({marks})`.
- **Decision.**
  - In `detect/metrics.py`, move the body of `run_metrics` that runs after its status check into `_metrics(run, rows) -> dict[str, float]`. `run_metrics` keeps its signature and calls `_metrics`.
  - Add `runs_metrics(store, runs: list[sqlite3.Row]) -> dict[str, dict[str, float]]`. It sends one query, `SELECT run_id, kind, payload, cost_usd FROM events WHERE run_id IN (...) AND kind IN ('llm_call','tool_call','tool_result')`, and groups rows by `run_id`. A refused or dropped run maps to `{}`.
  - Rewrite `changes.change_series`:
    - one runs query with `LIMIT ?` set to `limit`, which also selects `ended_at`;
    - then `runs_metrics(store, rows)`; every row with non-empty metrics becomes a point;
    - delete the OFFSET loop and `EXAMINE_FACTOR`;
    - update the "Queries:" sentence in the module docstring.
  - Leave the graph unchanged.
- **Why not a `PRAGMA data_version` cache.** `data_version` can only be compared within one connection, and `closes_connection` opens a new connection per request, so the cache would need its own long-lived connection. The board and the graph also depend on `now`, so the cache key would need a time bucket. At about 50 ms worst case per 10 s poll, a cache is not justified yet.
- **Tests.**
  - `tests/server/test_changes.py::test_change_series_reads_events_in_one_query`: seed 50 runs. Through `conn.set_trace_callback`, exactly 1 statement contains `FROM events`, and the points equal the old output.
  - `tests/detect/test_metrics.py::test_runs_metrics_matches_run_metrics`: for each seeded run, `runs_metrics(...)[id] == run_metrics(store, id)`.
  - Delete or replace `test_runs_without_metrics_take_no_slot`. It monkeypatches a path that cannot happen (see "tag markers" below).

### ~10 queries per workflow per board poll: DROP
- About 9 indexed queries per workflow (`views.py:148-177` and the `cli_report` helpers), measured at about 1.1 ms per workflow. 50 workflows would take about 55 ms per poll, which does not need fixing.

### Runs cursor skips ties on `started_at`: FIX
- **Real.** In `views.py:346-353`, `sql += " AND started_at < ?"` and `next_before=_num(rows[-1]["started_at"]) if more`. Rows that share the last row's `started_at` but fall past the page end are never returned (the docstring at :337-338 says so).
- **Decision.** Extend the page to the whole tie group:
  - Select `rowid AS _rowid, *`.
  - If `more` is true and `rows[limit]["started_at"] == rows[limit-1]["started_at"]`, append the rest of the tie group with the same WHERE clause plus `AND started_at = ? AND rowid < ? ORDER BY rowid DESC`. Use the last row's `started_at` and `_rowid`.
  - Then recompute `more` as `EXISTS(... AND started_at < ?)`.
  - A page can now hold more than `limit` rows. Update the docstring.
- **Test.** `tests/server/test_views.py::test_runs_page_cursor_keeps_ties`:
  - Seed runs at `started_at` 300, 200, 200, 200 and 100.
  - `runs_page(limit=2)` returns 4 runs with `next_before == 200`.
  - `runs_page(before=200)` returns the run at 100 with `next_before is None`.
  - Across both pages, all 5 ids appear and none repeats.

### Feed id tie-break sorts as strings: FIX
- **Real.** `views.py:250` does `items.sort(key=lambda i: i.id)`, which puts `"drift:10"` before `"drift:9"`.
- **Decision.** Replace both sorts (:250-251) with one sort, `items.sort(key=_feed_order, reverse=True)`. Define `_feed_order(i)` as follows: `kind, _, rest = i.id.partition(":")`, then return `(i.ts if i.ts is not None else -inf, kind, len(rest), rest)`. At equal times this puts newer ids first.
- **Test.** `tests/server/test_views.py::test_feed_ties_order_by_numeric_id`: 10 P2 drifts with the same `ts` come back as `drift:10`, `drift:9`, ... `drift:1`. Re-run `test_feed_kinds_and_order` afterwards.

### Explain 422 body shape differs from FastAPI's: DROP
- `routes.py:80-81` raises `HTTPException(422, str(e))`, which gives `{"detail": str}`. FastAPI's own 422 gives `{"detail": [...]}`.
- The UI's `errorMessage` (`ui/src/api/client.ts:62-67`) already handles both shapes, and `client.test.ts:93-97` tests both.
- The endpoint already has validation 422s, so its OpenAPI 422 schema cannot name just one shape anyway.

### `redact_text` stops at `( ) < > " '`: FIX
- **Real.** `views.py:115` has `_URL = re.compile(r"[A-Za-z][A-Za-z0-9+.-]*://[^\s()<>\"']+")`.
- `(`, `)` and `'` are legal unencoded in userinfo and query. For example, `https://u:p(ss@h/v1` gets cut at `(`, and `ss@h/v1` is left in the text unredacted.
- **Decision.**
  - Change the pattern to `[^\s<>\"]+`.
  - In `redact`, strip trailing characters in a loop:
    - any character in `_URL_TAIL + "'"`;
    - `)` only while `url.count(")") > url.count("(")`.
  - Then pass the stripped URL to `redact_url`, as today. This keeps the existing "(model X at base_url https://...)." text correct.
- **Test.** `tests/server/test_views.py::test_redact_text_paren_and_quote_in_url`:
  - `redact_text("at base_url https://u:p(a)ss@h.example/v1?k=s'x).")` returns `"at base_url https://h.example/v1)."`;
  - `redact_text("see 'https://h/v1?key=z'")` returns `"see 'https://h/v1'"`.

### Feed cutoff at exactly 200 runs: FIX
- **Real.** `views.py:247` has `if len(runs) == FEED_RUN_WINDOW:`. When the store holds exactly 200 runs, nothing was cut off, yet older drifts are dropped.
- **Decision.** Fetch `FEED_RUN_WINDOW + 1` rows, then set `full = len(runs) > FEED_RUN_WINDOW`, `runs = runs[:FEED_RUN_WINDOW]` and `if full:` at :247.
- **Test.** `tests/server/test_views.py::test_feed_window_cut_only_when_more_runs_exist`, with `monkeypatch.setattr(views, "FEED_RUN_WINDOW", 3)` and one P2 drift older than every run:
  - with 3 runs, the drift is in the feed;
  - after a 4th run, it is not.

### Changes OFFSET batching races: FIX (folded into "Polling cost")
- **Real but practically unreachable.** `changes.py:83` uses `LIMIT ? OFFSET ?` on separate read snapshots. A second batch only runs when a selected run returns `{}` metrics.
- The batched rewrite above removes the OFFSET loop. No separate test is needed.

### Tag markers only computed over runs with metrics: DROP (not real)
- `run_metrics` always returns at least `{"llm_calls": ...}` for a run that is not refused or dropped (`metrics.py:68-70`, `:93`). The changes query already excludes refused and dropped runs (`changes.py:82`).
- So every examined run is a point, and `_tag_markers` sees them all. The batched rewrite removes the skip path.

### `base_url` marker compared unredacted: FIX
- **Real.** In `changes.py:43-48`, `value = _tag_value(tag, r[tag])` is compared raw. A change only in credentials or query draws a marker that reads `https://h/v1 → https://h/v1`.
- The feed has the same bug at `views.py:225`: `(prev["model"], prev["base_url"]) == (r["model"], r["base_url"])`.
- **Decision.**
  - In `_tag_markers`, set `value = _tag_text(tag, _tag_value(tag, r[tag]))` before comparing, and store the redacted value as `before` and `after`.
  - In `feed`, compare `(r["model"], redact_url(r["base_url"]))`.
- **Test.** `tests/server/test_changes.py::test_base_url_change_only_in_secret_parts_draws_no_marker`, with runs on `https://k1@h/v1?key=a` and then `https://k2@h/v1?key=b`:
  - `change_series` has no base_url marker;
  - `feed(rt)` has no `model_change` item.

### Graph node-id collisions with `|` in agent/tool names: FIX
- **Real.** `graph.py:82` has `node_id = f"{kind}:{SEP.join(path)}|{name}"`. Agent `a|b` with tool `c` collides with agent `a` with tool `b|c`. An agent named `"x › y"` collides with agent `y` under `x`.
- **Decision.**
  - Add `_esc(s) = s.replace("\\", "\\\\").replace("|", "\\|").replace("›", "\\›")`.
  - Build `_agent_id` and the leaf id from `SEP.join(map(_esc, path))` and `_esc(name)`.
  - Labels stay raw. Ids without these characters are unchanged, so the existing tests and `ui/src/test/fixtures/graph.json` stay valid.
- **Test.** `tests/server/test_graph.py::test_node_ids_distinct_with_pipe_and_separator`:
  - the two collision pairs above give 4 distinct node ids;
  - each node's `calls == 1`.

### Graph halt/fallback placement depends on processing order: FIX
- **Real.** `graph.py:212-214` uses `b.leaf("tool", path, name, create=False) or b.leaf("llm", ...)`. A halt lands on the tool only if a run processed earlier (newest first) already created the tool node. Otherwise it lands on the agent.
- **Decision.** In `build_graph`'s loop (:271-274), append `(e, path, run_id)` to a `deferred` list when `e["kind"] in ("halt", "fallback")`. After all runs, call `_add_event` for each deferred item.
- **Test.** `tests/server/test_graph.py::test_halt_lands_on_tool_regardless_of_run_order`:
  - The newest run has only a `halt` event named `search`. An older run has `_tool(..., "search")`.
  - The `tool:planner|search` node has `verdicts.halt == 1`.
  - The agent node has no `halt`.

### Graph events query fetches all event kinds: DROP
- Every event kind written today (`agent` and the 9 kinds in `_LEAF_KINDS`; see the INSERT sites in `sdk/` and `otel/processor.py`) is read by the graph. A `kind IN` filter would save nothing.
- The cost is payload parsing, which was measured above.

### OTel agent events written before the parent fix lack `payload.parent`: DROP
- Only rows written by pre-fix development builds are affected, and the package is unpublished.
- When the span walk fails, such agents hang under the root (`graph.py:153-157`). The largest window is 30 days, so these rows age out.

### Worker probe-then-scan race: FIX
- **Real.** In `worker.py:79-82`, the probe (`with self.store.transaction(immediate=True): pass`) and then `scan(...)` leave a gap. If a writer takes the lock in between, `scan` logs a warning and returns, and `last_error` is set to None (`scan.py:80-82`).
- **Decision.**
  - Add `stopped: str | None = None` to `ScanReport`. `scan` sets it to `type(e).__name__` on both early exits (`scan.py:74-76` and :80-82).
  - In `Worker._scan`, set `self._last_error = report.stopped` instead of None. Keep the probe, which avoids a warning on every pass.
- **Test.** `tests/server/test_worker.py::test_worker_reports_a_scan_stopped_by_the_store`:
  - monkeypatch `pitstop.server.worker.scan` to return `ScanReport(0, (), stopped="OperationalError")`;
  - after `w.run_once()`, `w.status()["last_error"] == "OperationalError"`.

### Write guard is a method denylist: FIX
- **Real.** `app.py:48` has `_WRITE_METHODS = frozenset({"POST", "PUT", "PATCH", "DELETE"})`, and :108 checks `scope["method"] in _WRITE_METHODS`.
- **Decision.**
  - Replace it with `_SAFE_METHODS = frozenset({"GET", "HEAD", "OPTIONS"})`, and require the header when `scope["method"] not in _SAFE_METHODS`.
  - Update the module docstring line ("POST, PUT, PATCH and DELETE need...") to "every method other than GET, HEAD and OPTIONS needs...".
- **Test.** `tests/server/test_app.py::test_unlisted_methods_need_the_header`:
  - for `m` in `("PROPFIND", "TRACE", "FOO")`, `client.request(m, "/api/health")` returns 403 with `{"error": "missing X-PitStop header"}`;
  - with `X-PitStop: 1`, it does not return 403.

### 500 log includes the exception message: DROP
- `app.py:128-129` logs `exc_info=e`, so the traceback includes the message.
- The log only goes to the operator's own terminal, and the traceback is the point of logging. The client body is already only `{"error": "<Type>"}`, which `test_endpoint_errors_become_json` checks.

### Socket not closed if startup fails after bind: FIX
- **Real.** In `cli.py:345-359`, after `sock = _listen(host, args.port)`, an exception from `Worker(...)`, `create_app(...)`, `webbrowser.open` or `server.run` leaves `sock` open. This only matters in-process (tests); a real process exit closes it.
- **Decision.** Wrap everything from `worker = ...` through `server.run(sockets=[sock])` in `try: ... finally: sock.close()`. Closing twice after uvicorn's own shutdown is harmless.
- **Test.** `tests/server/test_cli_ui.py::test_ui_closes_socket_when_startup_fails`:
  - wrap `pitstop.cli._listen` to capture the socket;
  - monkeypatch `pitstop.server.app.create_app` to raise `RuntimeError`;
  - after `pytest.raises(RuntimeError)`, `captured[0].fileno() == -1`.

### `--host 0` / `0.0` pass the wildcard check: FIX
- **Real.** I checked it here: `ipaddress.ip_address("0")` raises ValueError, while `socket.inet_aton("0")`, `"0.0"`, `"0x0"` and `"000.0.0.0"` all give `0.0.0.0`. `_is_wildcard` (`cli.py:290-294`) therefore returns False.
- **Decision.**
  - Add `_ip(host)`. It tries `ipaddress.ip_address(bare)`. On ValueError it tries `ipaddress.IPv4Address(socket.inet_aton(bare))`. On OSError it returns None. It never does a DNS lookup.
  - `_is_wildcard` returns `ip is not None and ip.is_unspecified`. `_is_loopback` is `"localhost"` or `ip.is_loopback`, which also accepts `127.1`.
- **Test.** Extend `tests/server/test_cli_ui.py::test_ui_refuses_wildcard_host`'s parametrize with `"0"`, `"0.0"`, `"0x0"` and `"000.0.0.0"`. Each must exit 2 with `WILDCARD_HOST_MSG`, and nothing may be bound.

### tests/server: split the not-built-page assertions back out: FIX
- **Real.** `tests/server/test_app.py:109-119` (the `empty` static dir, "Dashboard not built", the npm commands, `/api/health` and the `missing` dir) sits inside `test_nul_in_static_path_is_404_without_traceback`.
- **Decision.** Move those lines into a new `test_not_built_page(make_client, tmp_path)`. They need no other change.

### `pitstop scan --every 1e300` overflows `time.sleep`: FIX
- **Real.** I checked it here: `time.sleep(1e300)` raises `OverflowError: timestamp out of range for platform time_t` (`cli.py:165`), after the first pass and with a traceback.
- **Decision.** Use `sc.add_argument("--every", type=_scan_every, ...)` (`cli.py:405`) and add "(at most 86400)" to its help text.
- **Test.** Add `"1e300"` and `"86401"` to the parametrize of `tests/detect/test_scan.py::test_scan_cli_rejects_a_bad_interval`. The command must exit 2, with `--every` and no traceback in stderr.

## SDK and alerts (src/pitstop/sdk/*, src/pitstop/otel/*, src/pitstop/alerts/*)

### `is_checked` consume not atomic across threads: FIX
- **Real.** In `guard.py:119-123`, `if mark is None or mark.consumed: return False` is followed by `mark.consumed = True`. The Mark is shared with threads started inside the block (the case deferred to M4), so two threads can both skip the guard.
- **Decision.** Add a module-level `_consume_lock = threading.Lock()`, and run the check-and-set under `with _consume_lock:`.
- **Test.** `tests/sdk/test_guard.py::test_mark_consumed_once_across_threads`:
  - inside `checked(ctx, "t")`, start 16 threads with `contextvars.copy_context().run`, all behind a `threading.Barrier`, each calling `is_checked(ctx, "t")`;
  - `sum(results) == 1`, repeated over 50 rounds.

### `_tag_run` check-then-update not atomic (adapters): DROP
- The check is only a cache (`openai_agents.py:246-252`; langchain's `_tag_run` at `langchain.py:300-303` has none). The atomic guard is the UPDATE's `WHERE ... (adapter IS NULL OR adapter='wrap')`.
- The race costs at most one duplicate UPDATE, which changes nothing.

### Nested same-name tool latency includes adapter overhead: DROP
- `record_tool_result` (`guard.py:150-154`) times from the outer mark, which `checked()` creates just before the framework invokes the tool.
- The extra time is the framework's own argument handling, which is reasonably part of the call. It is sub-millisecond.

### A halted nested Agents SDK run leaves its name on the stack: KEEP CARRIED (M4 adapter rework)
- A `Halted` escapes as `AgentsException` and ends the run, so the stale name dies with its `RunContext`.
- The real case is a nested run that fails without a halt. `as_tool`'s default `failure_error_function` (`agents/tool.py:697-711`) turns that into a tool error, and the outer agent continues with the nested name current (`context.py:173-197`).
- The stack is shared per scope (`id(base)`) across parallel tool calls. Neither "unwind on tool exit" nor "reconcile in `on_llm_start`" is safe without stacks per task.

### OTel processor uses one warn key for two failures: FIX
- **Real.** At `processor.py:290-292`, a drift that was not recorded warns under the key `"store"`, which is also used by `_ensure_run` and `_write` failures (:198, :363). An earlier event-write failure therefore hides the drift-record failure, which is logged only at debug.
- **Decision.** Use `self._warn("drift", "pitstop: could not record a drift from an OpenTelemetry trace; ignored", exc_info=False)` at :291.
- **Test.** `tests/otel/test_processor.py::test_drift_record_failure_warns_under_its_own_key`:
  - pre-add `"store"` to `proc._warned`;
  - monkeypatch `pitstop.sdk.actions.apply_findings` to return `[SimpleNamespace(drift_id=None)]`;
  - drive the 3-tool loop from `test_unrecorded_p0_warning_once`;
  - exactly one WARNING contains "could not record a drift".

### `_ensure_run` holds the trace lock during INSERT: DROP
- This is by design (`processor.py:181`, "so the insert and the root's close never interleave"). The lock is per trace.
- Without it, each thread would wait on the same SQLite busy_timeout anyway.

### End-time agent name only for spans ending after the agent span: DROP
- This is documented (`processor.py:11-12`). The run graph places OTel events by span links to the `agent` event, which carries the end-time name (`graph.py:166-169`), not by the stale `agent` column.
- Fixing it would mean rewriting earlier events.

### `__signature__` captured at wrap time (adapters): DROP
- `openai_agents.py:155-157`. A callable's signature does not change, and `__agents_bind_function_tool__` (:181-184) builds a new `_CheckedInvoker`, which captures the signature again.

### emit caches a failed import for the process lifetime: DROP
- `emit.py:15-24`. A package does not appear in the middle of a process. Retrying would repeat a `sys.path` scan for every verdict.
- The OTel processor passes `span=` and never reaches the import.

### `list[str]` prompt estimate sums per item (sdk/wrap): DROP
- At `shapes.py:424-425`, a `list[str]` prompt to `completions.create` is a batch, and each prompt is billed separately, so summing is correct.
- Rounding up per item overestimates by less than 1 token per item, which is the safe direction for a budget reservation.

### Budget cause prints 2 decimals: FIX
- **Real.** `alerts/model.py:110` has `f"Budget {scope} is used up ({spent:.2f} of {limit:.2f} USD)."`. A $0.01 per-run budget reads "0.01 of 0.01".
- **Decision.** Use `:.4f` for both values, which matches `pitstop budget`'s `_usd` (`cli.py:46-47`).
- **Tests.**
  - Update `tests/alerts/test_dispatch.py::test_refused_run_cause_names_budget` to expect `"(5.2000 of 5.0000 USD)."`.
  - Add `test_budget_cause_keeps_sub_cent_amounts`: with spent 0.0103 and limit 0.01, the cause contains `"0.0103 of 0.0100 USD"`.

### `_deliver_unrecorded` duplicates `_update` (alerts/dispatch.py): FIX
- **Real.** `dispatch.py:152-155` calls `_insert(...)` and then runs `UPDATE alerts SET status=?, error=? WHERE id=?`, the same SQL as `_update` (:256-258).
- **Decision.**
  - Give `_insert` the parameters `status: str = "pending"` and `error: str | None = None`, and insert them instead of the literal `'pending'`.
  - In `_deliver_unrecorded`, write the final row with one `_insert(c, None, alert.level, ch.name, "immediate", status="sent" if error is None else "failed", error=error)` and delete the UPDATE.
- **Test.** Covered by the next item.

### Send-first test doesn't assert the follow-up row: FIX
- **Real.** `tests/alerts/test_dispatch.py:697-721` (`test_unrecorded_p0_sends_before_recording`) ends with `assert dispatch.wait_idle(30)` and never reads `alerts`.
- **Decision.** After `wait_idle`, assert that `[(r["drift_id"], r["level"], r["channel"], r["kind"], r["status"], r["error"]) for r in _rows()] == [(None, "P0", "webhook", "immediate", "sent", None)]`.

### Tautological assertion in the store-refused test: FIX
- **Real.** At `tests/alerts/test_dispatch.py:745`, `rt.store is get_runtime().store` is always true, because `_init` returns `pitstop.init(...)`, which is that runtime.
- **Decision.** Delete that clause and keep `assert _rows() == []`. That part is meaningful: it proves the real store was not used in place of the refusing one.

# Part B: UI, build, Python tests, example and docs
Line numbers are from `main` at c98ba6b. "Test:" gives the file, the test name and the key assertions.

## Vitest upgrade

- **Real.** `ui/package.json` pins `vitest` at `3.2.7`, which pulls in `@vitest/mocker@3.2.7` (advisory range `<=4.1.11`), `tinypool@1.1.1` (advisory range `<=2.1.1`) and `vite-node@3.2.4`.
- **Version.** The cutoff is 2026-10-09 (14 days before 2026-10-23). Release dates from `npm view vitest time`:
  - 5.0.3: 2026-09-30, eligible.
  - 5.0.4: 2026-10-12, too new.
  - 5.0.5: 2026-10-15, too new.
  - 4.1.12: 2026-10-15, too new.
  - 4.1.11: 2026-08-18, but its `@vitest/mocker` is still inside the advisory range.
  - **Pin `vitest` at `5.0.3`.** Its `@vitest/mocker` is `5.0.3`, which is outside the advisory range. Its dependency list has no `tinypool` and no `vite-node`, so both advisories go away.
- **Nothing else moves:**
  - vitest 5.0.3's peer ranges are `vite ^6.4||^7||^8`, `jsdom *` and `@types/node ^22||>=24`. The repo's `vite 7.3.7`, `jsdom 29.1.1` and `@types/node 22.20.5` all fit.
  - Its Node engine is `^22.12||^24||>=26`, which `engines.node >=22.12` covers.
  - `@testing-library/jest-dom 7.0.1` declares the peer `vitest >=0.32` and augments `declare module 'vitest'`, so it still works.
  - The repo uses no other `@vitest/*` package (no coverage, ui or browser).
  - `.npmrc` has `save-exact=true`.
- **Steps.**
  - `cd ui && npm install -D vitest@5.0.3`. This writes `"vitest": "5.0.3"` to `package.json` and updates `package-lock.json`.
  - Then check that `npm ls tinypool vite-node` prints nothing, that `npm audit` reports 0, and that `npm test -- --run` and `npm run typecheck` pass.
- **Config impact.**
  - `ui/vite.config.ts` uses only `defineConfig` from `vitest/config` (still exported in 5.0.3), plus `environment: "jsdom"`, `setupFiles`, `restoreMocks` and `unstubGlobals`. It sets none of the options that Vitest 4 removed or renamed (`workspace`, `poolOptions`, `deps.*`, `coverage.all`, the `basic` reporter).
  - In Vitest 4, `restoreMocks: true` only restores `vi.spyOn` spies and no longer clears `vi.fn` call history. Every test builds a fresh `vi.fn`, so nothing depends on the old behaviour.
  - `ui/src/test/setup.ts` needs no change.
  - The 5.x changelog postdates my knowledge and I couldn't fetch it offline, so the test run is the gate. The places to look first if it fails are the fake-timer defaults (`poll.test.tsx`, `Workflow.test.tsx`) and the jsdom environment setup.

## UI tests to add

- **Dismiss 404/409.**
  - The path is real but untested: `ui/src/pages/Workflow.tsx:253-256` (`if (e instanceof ApiError && (e.status === 404 || e.status === 409)) { refresh(); return; }`). `client.test.ts:99-103` only covers the messages.
  - Test: `ui/src/pages/Workflow.test.tsx`, `it.each([404, 409])("dismiss of a drift already gone (%i) stays dismissed, no error")`.
  - The POST to `/api/drifts/2/dismiss` returns `json({status:"not_found"|"not_open"}, code)`. After clicking Yes and flushing:
    - "Out of scope action" is absent.
    - The list has 1 article and no `alert`.
    - `/api/workflow` was requested at least twice (the refresh).
- **"≥" and "—" time rendering.**
  - The code is `ui/src/graph/nodes.ts:60-63` (`timePerRun`) and `NodeDrawer.tsx:43`. No test fixture has `time_partial: true` or a null time (every node in `graph.json` has `time_partial: false`).
  - Test: `Workflow.test.tsx`, `"node cards show a lower bound and an unknown time per run"`.
    - Build the fixture with `structuredClone(graph)`: set `agent:writer` to `time_partial = true`, and set `time_per_run_s = null` on `tool:writer|publish_note`.
    - `nodeCard("writer, agent")` contains "≥ 1.1 ms per run".
    - `nodeCard("publish_note, tool under writer")` contains "— per run".
    - Clicking writer opens a drawer containing "(some calls had no recorded latency)".
- **Explain 404 and 422.**
  - Real gap: `ExplainPanel.tsx:17-20` passes the raw server detail through, so a 404 reads "Couldn't explain this drift: no drift 3". The file header promises "says so in words".
  - Fix: in `message()`, add `if (e instanceof ApiError && e.status === 404) return "This drift no longer exists.";`. Keep the 422 detail ("drift 3 has no workflow", from `server/routes.py:80-81`).
  - Test: new file `ui/src/components/ExplainPanel.test.tsx`, `"says a gone or unexplainable drift in words"`. It stubs fetch with 404 `{detail:"no drift 3"}` and then 422 `{detail:"drift 3 has no workflow"}`.
    - The `alert` text is "Couldn't explain this drift: This drift no longer exists.".
    - Then it contains "drift 3 has no workflow".
- **Poll during a pending dismiss.**
  - Untested: the hidden set (`Workflow.tsx:238,288`) must win over a fresh poll answer.
  - Test: `Workflow.test.tsx`, `"a poll while a dismiss is pending keeps the card hidden"`. Use fake timers and the runs tab (`workflowHref(NAME, "runs")`); the POST promise is left pending.
    - Click Yes, then `advanceTimersByTimeAsync(10_000)`.
    - `/api/workflow` was requested twice, but "Out of scope action" is still absent.
    - Resolve the POST with 200 `{status:"dismissed"}`: the card stays absent and there is no `alert`.
- **Poll hidden-abort and refresh-abort.**
  - Untested: `ui/src/api/poll.ts:80-91` (`stop()` aborts on hide; `refreshRef.current = () => { stop(); void tick(); }`).
  - Add two tests to `ui/src/api/poll.test.tsx`, where `fn` never settles and pushes its signals:
    - `"hiding the tab aborts the request in flight"`: `setHidden(true)` → `signals[0].aborted`. Then `setHidden(false)` → `fn` called twice.
    - `"refresh aborts the request in flight and fetches again"`: Probe gains a button that calls `refresh`. Click it → `signals[0].aborted`, `fn` called twice. Resolve the second call → the output shows its data.
- **Chart single point, all-equal and missing metric.**
  - Untested edges in `SeriesChart.tsx:139-155`.
  - Test: `ui/src/components/SeriesChart.test.tsx`, `"draws one run, a flat series and a metric no run has"`.
    - One point: one `circle[data-run]`, and `path.series-line` `d` matches `/^M[\d.]+ [\d.]+$/`.
    - Three points of 5: every `cy` is equal and the top tick reads "5".
    - Every point missing the metric: no `circle`, no `path.series-line`, and the head says "no data".

## Accessibility polish

- **`aria-label` on `<p>` in the tile.**
  - Real: `ui/src/components/Tile.tsx:71` (`<p className="tile-levels" aria-label="Open drifts">`). ARIA prohibits naming a paragraph, so screen readers ignore the label.
  - Fix: drop the attribute and put `<span className="sr-only">Open drifts: </span>` as the first child of the `<p>`.
  - Test: `PitWall.test.tsx`, add to `"tile shows text flag label and level chips"`: the red tile's `.tile-levels` has no `aria-label` and its text starts with "Open drifts:".
- **Unannounced skeletons on the pit wall.**
  - Real: `ui/src/pages/PitWall.tsx:55` and `:73` put `aria-busy` and `aria-label` on a `<div>`, with no live region. Every other page uses `<p className="sr-only" role="status">` (for example `Workflow.tsx:75`).
  - Fix: remove `aria-label` from both. Add `<p className="sr-only" role="status">Loading the pit wall…</p>` to `Skeleton` and `<p className="sr-only" role="status">Loading the feed…</p>` to `FeedSkeleton`.
  - Test: `PitWall.test.tsx`, `"announces loading"`. Fetch never settles: `getByRole("status")` has the text "Loading the pit wall…".
- **Node buttons hide their facts from screen readers.**
  - Real: `ui/src/components/RunGraph.tsx:70` (`aria-label={nodeName(node)} aria-pressed={open}`). The label replaces the calls, cost, p50, time and badges text.
  - Fix in `PitNodeCard`:
    - `const descId = useId()`.
    - Wrap the two `.gnode-facts` spans and the badges in `<span id={descId} className="gnode-desc">`, and add `.gnode-desc { display: contents; }` to `workflow.css`.
    - Put `aria-describedby={descId}` on the button and replace `aria-pressed={open}` with `aria-expanded={open}`.
  - Test: `Workflow.test.tsx`, `"node buttons describe their facts and say when open"`, on the button named "m-a, model under researcher":
    - `toHaveAccessibleDescription(/180 calls · .*p50 0\.6 ms/)`.
    - `aria-expanded` is "false" and there is no `aria-pressed`. After a click, `aria-expanded` is "true".
- **Scope line `role="status"` announces every poll.**
  - Real: `RunGraph.tsx:302` (`<p className="graph-scope" role="status">`). The run count in it changes as runs arrive, so it is re-announced.
  - Fix: remove `role="status"`. The picker and the select already tell the user the scope.
  - Test: `Workflow.test.tsx`, add to the first test: `getByText("last 24 hours · 95 runs, averaged")` has no `role` attribute and `.closest('[role="status"],[aria-live]')` is null.
- **Focus lost after Dismiss.**
  - Real: `DriftCard.tsx:80-83` unmounts the focused Yes button, and `Workflow.tsx:243` hides the card, so focus falls to `<body>`.
  - Fix in `WorkflowPage`:
    - `const sideTitle = useRef<HTMLHeadingElement>(null)`.
    - Give the `<h2 id="drifts-title">` at line 372 `ref={sideTitle} tabIndex={-1}`.
    - Call `sideTitle.current?.focus()` in `dismiss` right after `setHidden`.
  - Test: `Workflow.test.tsx`, `"dismiss moves focus to the Open drifts heading"`. After Yes, `getByRole("heading", { name: /^Open drifts/ })` has focus.
- **Duplicate "Explanation" region names.**
  - Real: `ExplainPanel.tsx:38` (`aria-label="Explanation"`). It is used by `DriftCard.tsx:97` and `MarkerLegend.tsx:71`.
  - Fix:
    - Add a prop `label: string` and set `aria-label={`Explanation: ${label}`}`.
    - `DriftCard` passes `title`.
    - `MarkerLegend` passes `markerLabel(marker, metrics)`.
  - Test: `Workflow.test.tsx`, extend `"explain opens inline"`: the region's name is "Explanation: Cost or latency creep". The existing `/Explanation/` queries still match.
- **Drawer heading ids collide (`useId`).**
  - Real but theoretical: `NodeDrawer.tsx:34` (`node-${node.id.replace(/[^a-zA-Z0-9_-]/g, "_")}`) maps `a|b` and `a:b` to the same id.
  - Fix: `const titleId = useId();`. The `-verdicts` and `-runs` suffixes stay.
  - Test: new file `ui/src/components/NodeDrawer.test.tsx`, `"names the drawer and its sections"`. Render inside a `MemoryRouter`:
    - `getByRole("complementary", {name: node.label})`.
    - `getByRole("region", {name: "Verdicts"})` and `getByRole("region", {name: /^Runs/})`.
    - The heading's id does not start with "node-".
- **The tile focus ring relies on `:has()`.**
  - Real: `ui/src/styles/pitwall.css:168-175`. `.tile-name a:focus-visible { outline: none; }` applies unconditionally, so a browser without `:has()` shows no ring at all.
  - Fix: wrap both rules (the `.tile:has(...)` ring and the `outline: none`) in `@supports selector(:has(*)) { … }`.
  - Test: `PitWall.test.tsx`, `"tile focus ring falls back without :has()"`. Use `import css from "../styles/pitwall.css?raw"` and assert that `css` matches `/@supports selector\(:has\(\*\)\)\s*\{[^@]*\.tile-name a:focus-visible\s*\{\s*outline: none/`.

## UI robustness

- **Layout failure shows an empty canvas.**
  - Real: `RunGraph.tsx:204-206`. On rejection it calls `setLaid(null)`, so `nodes` becomes `[]` and an empty React Flow renders.
  - Fix:
    - Add `const [layoutError, setLayoutError] = useState<unknown>(null)`. Set it on rejection and clear it on success.
    - In the non-empty branch, when `layoutError` is set, render `<ErrorState error={layoutError} title="The run graph couldn't be drawn" onRetry={graph.refresh} />` in place of the canvas.
  - Test: new file `ui/src/components/RunGraph.test.tsx`, `"a layout failure says so and offers a retry"`.
    - Use `vi.mock("../graph/layout", () => ({ layoutGraph: vi.fn(() => Promise.reject(new Error("elk failed"))) }))` and the fetch stub for `/api/graph` and `/api/runs`.
    - `findByRole("alert")` contains "The run graph couldn't be drawn" and "elk failed", and there is a "Try again" button.
- **Chart hit-testing mixes CSS px and viewBox units.**
  - Real: `SeriesChart.tsx:175` (`const px = clientX - el.getBoundingClientRect().left;`) is compared with `dots[i].cx`, which is in viewBox units. The SVG has `max-width: 100%` (`changes.css:83`).
  - Fix in `nearest`:
    - `const rect = el.getBoundingClientRect();`
    - `const px = (clientX - rect.left) * (rect.width > 0 ? (right + PAD.right) / rect.width : 1);`
  - Test: `SeriesChart.test.tsx`, `"pointer finds the run when the chart is scaled down"`.
    - Stub the SVG's `getBoundingClientRect` to `{left: 0, width: viewBoxWidth / 2}`.
    - `pointerMove` at `clientX = lastDotCx / 2`, then `getByTestId("series-tip")` contains the last run's short id.
- **`tickText` has no M or B units.**
  - Real: `SeriesChart.tsx:65` (`if (Math.abs(v) >= 1000) return `${trim((v / 1000).toFixed(1))}K`;`), so 2.5e6 reads "2500K".
  - Fix: before the K check, add `>= 1e9 → "B"` (divide by 1e9) and then `>= 1e6 → "M"`, with the same `trim(toFixed(1))`.
  - Test: `SeriesChart.test.tsx`, extend `"formats ticks by unit"`: `tickText("", 2_500_000) === "2.5M"`, `tickText("", 3e9) === "3B"`, and `tickText("", 2500) === "2.5K"` (unchanged).
- **The line joins across missing-metric runs.**
  - Real: `SeriesChart.tsx:155`. The path walks `dots` only (missing slots already dropped), with `M` only at `i === 0`.
  - Fix:
    - Keep the slot index in each dot: `{ p, i, cx, cy }`.
    - Emit `M` when `k === 0 || d.i !== dots[k - 1].i + 1`, otherwise `L`.
  - Test: `SeriesChart.test.tsx`, `"breaks the line where a run has no value"`. The metric is present at slots 0, 1 and 3 of 4: `path.series-line` `d` has exactly 2 `M`.
- **Unknown latency or cost is omitted, not shown as "—", in the event timeline.**
  - Real: `EventTimeline.tsx:166-167` (`{e.latency_s !== null && …}{e.cost_usd !== null && <Money …/>}`). The header promises "unknown is '—', never 0".
  - Fix: for `e.kind === "llm_call"` or `"tool_result"`, always render `<span title="latency">{duration(e.latency_s)}</span>`. For `llm_call`, always render `<Money usd={e.cost_usd} />`. Other kinds keep today's behaviour.
  - Test: `EventTimeline.test.tsx`, `"shows unknown latency and cost of a model call as —"`. An `llm_call` with `latency_s: null, cost_usd: null`: its `.ev-facts` has the text "—" for latency and contains "unknown" (Money's sr-only text) for cost.
- **The truncated notice is only at the bottom.**
  - Real: `EventTimeline.tsx:176`, after up to 2,000 events.
  - Fix: render the same `<p className="ev-truncated">` before the groups as well (top and bottom).
  - Test: `EventTimeline.test.tsx`, update `"says when only the first 2,000 events are shown"`:
    - `getAllByText("Showing the first 2,000 events")` has length 2.
    - The first one precedes the first `.ev-group`: `compareDocumentPosition & DOCUMENT_POSITION_FOLLOWING`.
    - `RunDetail.test.tsx:61` changes to `getAllByText(...)[0]`.
- **`RunGraph` with `controls={false}` still polls.**
  - Real but cheap: `RunGraph.tsx:192`. With no controls it still sets state with `Promise.resolve(NO_RUNS)` every 10 s, which re-renders the graph.
  - Fix:
    - Add a 4th parameter `enabled = true` to `usePoll`, include it in the effect deps, and when it is false set `{data: null, error: null, loading: false}` and return.
    - `RunGraph` calls `usePoll((s) => getRuns(name, null, { signal: s }), EVERY_MS, [name], controls)`.
  - Test: `poll.test.tsx`, `"a disabled poll never calls fn"`. Advance 30 s: `fn` is never called and the output reads "none".
- **`statusWord` lives in `RunsTable`.**
  - Real: it is defined at `RunsTable.tsx:22-26` and imported from there by `SeriesChart.tsx:19` and `RunDetail.tsx:11`.
  - Fix: move `RUN_STATUS` and `statusWord` to `ui/src/format.ts` and update the three imports.
  - No new test: `npm run typecheck` and the existing `RunsTable`, `SeriesChart` and `RunDetail` tests cover it.
- **≤900 px canvas captures vertical swipes.**
  - Real: `RunGraph.tsx:128-146` leaves `panOnDrag` at its default (true). On narrow screens the canvas is a full-width block (`workflow.css:857-862`), so a touch swipe pans the graph instead of scrolling the page.
  - Fix:
    - Add `useNarrow()` in `RunGraph.tsx`: `useSyncExternalStore` over `matchMedia("(max-width: 900px)")`, returning false when `matchMedia` is missing.
    - Pass `panOnDrag={!narrow}` to `<ReactFlow>`. Zoom stays on the Controls buttons and pinch.
  - Test: `RunGraph.test.tsx`, `"narrow screens leave vertical swipes to the page"`.
    - Stub `matchMedia` to `matches: true`: `.react-flow__pane` does not have the class `draggable`.
    - Without the stub, it has `draggable`.
- **Settings: alerts empty state.**
  - Real: `Settings.tsx:229-291`. The server always lists all 5 channels (`server/views.py:443-447`), so "none configured" shows only as five "no" cells.
  - Fix: when `!alerts.channels.some((c) => c.configured)`, render `<p className="set-empty">No alert channel is configured: drifts show here and in <code>pitstop drifts</code> only.</p>` above the table.
  - Test: `Settings.test.tsx`, extend `"renders each section from fixture"` (every channel in the fixture is unconfigured): `getByText(/No alert channel is configured/)`.
- **Settings: negative remaining budget.**
  - Real: `Settings.tsx:174-176` renders `<Money usd={b.remaining_usd} />` as "-$0.5000".
  - Fix: when `b.remaining_usd < 0`, render `<Money usd={0} /> <span className="set-over">over by <Money usd={-b.remaining_usd} /></span>`.
  - Test: `Settings.test.tsx`, `"an overspent budget says by how much"`. A budget row with `remaining_usd: -0.5`: the row contains "over by $0.5000" and does not contain "-$".
- **Settings: `ConfigNote` path with a custom `PITSTOP_HOME`.**
  - Real: `Settings.tsx:23-29` hardcodes `~/.pitstop/config.toml`.
  - Fix:
    - `function ConfigNote({ home }: { home: string })` renders `<code>{inHome(home, "config.toml")}</code>`.
    - Thread `home={s.home}` through `Budgets`, `Judge` and `Alerts` (three new props).
  - Test: `Settings.test.tsx`, `"config notes point at the configured home"`. With `home: "/custom/ps"`, `getAllByText("/custom/ps/config.toml")` has length 4 (three notes plus the Home section) and `queryByText("~/.pitstop/config.toml")` is null.
- **Settings: silent health-poll failure.**
  - Real: `Settings.tsx:378` and `:423` (`<Worker worker={h?.worker ?? s.worker} />`). A failing `/api/health` silently falls back to the settings snapshot, and the version line disappears.
  - Fix: pass `stale={health.error !== null}` to `Worker`. It renders `{stale && <Stale what="Worker status" sinceS={null} />}` under its `dl`.
  - Test: `Settings.test.tsx`, `"says when worker status can't refresh"`. `/api/health` returns 503: `findByRole("status", { name: "Worker status not refreshed" })`.

## Build

- **Rollup warns about a >500 kB chunk for the lazy elkjs chunk.**
  - Real: the last build left `src/pitstop/server/static/assets/RunGraph-*.js` at 1,642,257 bytes. It holds elkjs's bundled build and is loaded only on the graph tab. `index-*.js` is 380 kB.
  - Fix: in `ui/vite.config.ts:14` add `chunkSizeWarningLimit: 1800` to `build`, with the comment `// the lazy RunGraph chunk carries elkjs (~1.6 MB), fetched only on the Run graph tab`.
  - Check: `npm run build` prints no "larger than" warning. No unit test.
- **`ui/scripts/gen_fixtures.py` bare `next()`.**
  - Real: `gen_fixtures.py:80` (`alarm = next(c for c in workflow["drifts"] if c["drift_type"] == "cost_or_latency_creep")`). If the demo changes, the script dies with a bare `StopIteration`.
  - Fix: use `next((...), None)`, then `if alarm is None: raise SystemExit("the demo recorded no cost_or_latency_creep drift on earnings-research; fixtures not written")`.
  - No test; it is a dev script.
- **Poll tests coverage.** Drop as a separate item. It is covered by the "poll hidden-abort and refresh-abort" and "a disabled poll never calls fn" tests above. No coverage tool is configured, and adding `@vitest/coverage-v8` is not needed for this.
- **Light-token comments (p3 and p1/p2 chroma).**
  - Real: `ui/src/styles/tokens.css:22-26`.
    - p1 changed from the mockup's `oklch(60% 0.16 48)` to `54% 0.15`, and p2 from `66% 0.13 85` to `52% 0.11`. The comments mention only lightness.
    - p3 changed from `55%` to `52%` with no comment.
    - Computed here: p1 4.65:1, p2 4.95:1 and p3 4.75:1 on their `-bg`. The mockup's p3 is 4.18:1. The lowered chroma keeps p1 and p2 inside sRGB at the new lightness.
  - Replace the comments with:
    - p1: `/* the mockup's oklch(60% 0.16 48) is 3.6:1 on --p1-bg; 54% clears AA (4.6:1), chroma 0.15 keeps it in sRGB */`
    - p2: `/* the mockup's oklch(66% 0.13 85) is 2.8:1 on --p2-bg; 52% clears AA (4.9:1), chroma 0.11 keeps it in sRGB */`
    - p3: `/* the mockup's 55% is 4.2:1 on --p3-bg; 52% clears AA (4.7:1) */`
  - No test.

## Python tests to add (tests/)

- **`latency_s` on the async and error tool paths.**
  - Untested: `src/pitstop/sdk/tool.py:36-42` (async) and `:55-61` (sync) call `record_tool_result(ctx, tname, error=e)` inside `checked()`. `guard.py:152-154` adds `latency_s` when the mark is present. Only sync success is tested (`tests/sdk/test_guard.py:262`).
  - Test: `tests/sdk/test_guard.py`, `test_tool_result_latency_on_async_and_error_paths`.
    - An async `@pitstop.tool(name="aslow")` awaits `asyncio.sleep(0.05)` and returns. A sync `@pitstop.tool(name="boom")` sleeps 0.02 and raises `ValueError`. An async `@pitstop.tool(name="aboom")` sleeps 0.02 and raises.
    - Run them under `async with workflow("wf")` via `asyncio.run`, with `pytest.raises` around the two raisers.
    - Each `tool_result` has `latency_s >= 0.05` or `>= 0.02` respectively. Both error payloads have `"error"` containing "ValueError".
- **A failing agent-event write fails open.**
  - Untested: `src/pitstop/sdk/context.py:199-214` (`note_agent` → `_safe(write)`).
  - Test: `tests/sdk/test_context.py`, `test_agent_event_write_fails_open`.
    - `pitstop.init(home=tmp_path)`. Inside `with workflow("wf"):`, `monkeypatch.setattr(get_runtime().store, "transaction", boom)`, where `boom` raises `sqlite3.OperationalError`.
    - Run `with agent("a"): x = 1`, then call `monkeypatch.undo()` before the workflow exits.
    - No exception, `x == 1`, `"pitstop: internal error ignored"` is in `caplog.text`, and `_agent_events() == []`.
- **A re-entered `agent()` writes one event.**
  - Untested: the per-`(parent, name)` dedupe at `context.py:202-206`.
  - Test: `tests/sdk/test_context.py`, `test_reentered_agent_writes_one_event`.
    - In `with workflow("wf"):`, loop 3× over `with agent("a"): pass`, then call a function decorated `@agent("a")` twice.
    - `_agent_events() == [("a", "a", {"parent": None, "source": "sdk"})]`.
- **prompt_versions key order and malformed JSON.**
  - Untested: `server/changes.py:42-43` uses `detect/attribution.py:55-63` (`_tag_value`: `json.dumps(json.loads(v), sort_keys=True)`, falling back to `str(v)` on `ValueError`).
  - Test: `tests/server/test_changes.py`, `test_prompt_versions_key_order_and_malformed_json`.
    - Runs, in order: `'{"a": 1, "b": 2}'`, `'{"b": 2, "a": 1}'`, `'not json'`, `'not json'`.
    - Exactly one marker: `("prompt_versions", '{"a": 1, "b": 2}', 'not json', run3)`.
    - `client.get("/api/changes", params={"workflow": "wf"}).status_code == 200`.

## Example and docs

- **The finresearch fallback hardcodes the tool name.**
  - Real: `examples/financial-research/finresearch/agents/worker.py:60` (`pitstop.tool(side_effects=True, name="youcom_research")`). `orchestrator.py:84` defines `RESEARCH_TOOL = "youcom_research"` separately.
  - Fix:
    - Define `RESEARCH_TOOL = "youcom_research"` in `agents/worker.py` and use it at line 60.
    - In `orchestrator.py`, delete line 84 and import it: `from .agents.worker import RESEARCH_TOOL, ResearchWorker`.
  - Test: `tests/examples/financial_research/test_sdk_wiring.py`, `test_worker_fallback_uses_the_research_tool_name`.
    - `orchestrator.RESEARCH_TOOL is worker.RESEARCH_TOOL`.
    - A `ResearchWorker(..., watched=True)` built without `research_call` calls the patched `pitstop.tool` with `name=RESEARCH_TOOL`.
- **`--offline` truthiness checks.**
  - Real: `finresearch/cli.py:225-226` uses `("--cache-dir", args.cache_dir)` and `("--max-research-calls", args.max_research_calls)`, so `--offline --max-research-calls 0` (and `--cache-dir ""`) passes. `cli.py:99` uses `args.max_research_calls or s.max_research_calls`, so a live `0` silently becomes 12. `benchmarks/calibration/loop.py:150` has the same pattern.
  - Fix: use `args.cache_dir is not None` and `args.max_research_calls is not None` in the list. Line 99 and loop.py:150 become `s.max_research_calls if args.max_research_calls is None else args.max_research_calls`.
  - Test: `tests/examples/financial_research/test_cli_run_config.py`, add `(["--max-research-calls", "0"], "--max-research-calls")` to the `test_offline_rejects_unsupported_flags` parametrize.
- **Only the first unsupported flag is reported.**
  - Real: `cli.py:228` (`r.error(f"--offline does not support {unsupported[0]}")`).
  - Fix: `r.error(f"--offline does not support {', '.join(unsupported)}")`. The existing single-flag cases still match.
  - Test: `test_cli_run_config.py`, `test_offline_names_every_unsupported_flag`. `_cli(tmp_path, "--offline", "--no-report", "--cache-dir", "x")` has returncode 2 and stderr containing "--offline does not support --no-report, --cache-dir".
- **The benchmark test's monkeypatch is brittle.**
  - Real: `tests/examples/financial_research/test_benchmark_halted.py:19`.
    - It patches `orch.Swarm` and depends on `loop.cmd_swarm` importing `Swarm` inside the function (`loop.py:133`).
    - It still builds a real `Supervisor`, real `LLMClient`s and a real `FinanceResearchClient` from `load_settings()`.
  - Fix:
    - Extract `def halted_exit(results) -> int` in `benchmarks/calibration/loop.py` from lines 164-168. It prints `swarm run {r.run_id} halted: {r.halted}` to stderr for each halted run and returns 1, or 0 when none halted.
    - `cmd_swarm` does `if halted_exit(results): return 1`.
  - Test: rewrite `test_benchmark_fails_on_halted_run` to call it directly.
    - `loop.halted_exit([SimpleNamespace(run_id="nvda_amd-abc123", halted="budget exceeded")]) == 1`, and stderr names the run.
    - `loop.halted_exit([SimpleNamespace(run_id="x", halted=None)]) == 0`.
- **examples/quickstart/dashboard_demo.py refuses only `pitstop.db` and `config.toml`.**
  - Real: `dashboard_demo.py:355-358`. Its own `--home` help says "an empty directory", but a home holding only `pitstop.db-wal` or other files is accepted.
  - Fix:
    - Replace the loop with `if home.exists() and any(home.iterdir()): print(f"{home} is not empty; pick an empty directory", file=sys.stderr); sys.exit(2)`.
    - Update the docstring at `:19` and the last sentence of `examples/quickstart/README.md:40` to "refuses a directory that is not empty".
  - Test: `tests/server/test_dashboard_demo.py`, `test_dashboard_demo_refuses_non_empty_home`. The home contains `notes.txt`: returncode 2, "pick an empty directory" in stderr, and the directory still lists only `["notes.txt"]`.
- **The quickstart's "no env var key is read" is loose.**
  - Real: the phrase is in the `dashboard_demo.py:18` docstring. `src/pitstop/config.py:10-12` runs `load_dotenv(find_dotenv(usecwd=True))` when pitstop is imported. The README itself says only "needs no keys" (`README.md:34`), which is accurate.
  - Fix: reword line 18 to `request is answered in-process with a scripted reply, so nothing leaves the machine and no API key is needed (the clients get a placeholder; pitstop still loads a .env from the current directory, as it always does).`
  - No test.
- **The demo test calls `init()` in-process.** Drop. Every server test builds its app on `init()` (`tests/server/conftest.py:10-11`), and the root conftest calls `reset_runtime()` around each test (`tests/conftest.py:15-17`).
- **A spend assertion could flake at midnight.**
  - Real: `tests/server/test_dashboard_demo.py:42` (`0 < earnings["spend_today_usd"] < 2.0`). The demo stamps runs with the real clock, so a run straddling midnight reads 0.
  - Fix:
    - Change it to `0 <= earnings["spend_today_usd"] < 2.0`.
    - Add `assert sum(r["cost_usd"] or 0 for r in client.get("/api/runs", params={"workflow": "earnings-research"}).json()["runs"]) > 0`.
- **The dashboard empty state's step 3 says `pitstop ui`.**
  - Real: `ui/src/components/EmptyState.tsx:14-16`.
    - `SERVE = "pitstop ui"`, `DEMO = "python examples/…"` and `DEMO_SERVE = "PITSTOP_HOME=… pitstop ui"`.
    - Step 1 tells the reader to run `uv sync` "in a clone", so those commands need `uv run`.
  - Fix:
    - `SERVE = "uv run pitstop ui"`.
    - `DEMO = "uv run python examples/quickstart/dashboard_demo.py --home /tmp/pitstop-demo"`.
    - `DEMO_SERVE = "PITSTOP_HOME=/tmp/pitstop-demo uv run pitstop ui"`.
    - Optionally `dashboard_demo.py:359`'s "Run: PITSTOP_HOME={home} uv run pitstop ui", with the matching assertion at `test_dashboard_demo.py:32`.
  - Test: `ui/src/components/EmptyState.test.tsx`, update `"EmptyState shows the setup snippet"`: `getByText("uv run pitstop ui")` and `getByText("uv run python examples/quickstart/dashboard_demo.py --home /tmp/pitstop-demo")`.
- **README sections outside Dashboard lead with `pip install "pitstop[...]"` for an unpublished package.**
  - Real. The lines are `README.md:114` (Add PitStop to your agent, `pitstop[anthropic]`), `:307` (OpenAI Agents SDK, `pitstop[openai-agents]`), `:335` (LangChain / LangGraph, `pitstop[langchain]`) and `:364` (OpenTelemetry, `pitstop[otel]`).
  - `:283` is inside the Dashboard section and already says "Once PitStop is published". `:501` and `:562` are not install instructions.
  - `uv sync` already installs every extra's packages through the `dev` group (`pyproject.toml:28-32`).
  - Fix: reword each line to "In a clone, `uv sync` installs it (once PitStop is published: `pip install "pitstop[<extra>]"`)." For example, `:307` becomes "In a clone, `uv sync` installs the Agents SDK support (once PitStop is published: `pip install "pitstop[openai-agents]"`). Guard the agent's tools, …".
  - Test: optionally, in `tests/test_repo_hygiene.py`, `test_readme_installs_lead_with_the_checkout`: every README line containing `pip install "pitstop[` also contains "once" and "published".
