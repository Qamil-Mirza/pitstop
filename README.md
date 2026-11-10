# PitStop

Keep your agents on the racing line: drift detection and GEPA-powered prompt correction for agent workflows. Open source, runs on your machine.

[![CI](https://github.com/Qamil-Mirza/gepa-financial-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/Qamil-Mirza/gepa-financial-agent/actions/workflows/ci.yml)
[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Python >= 3.11](https://img.shields.io/badge/python-%E2%89%A53.11-blue.svg)](pyproject.toml)

![PitStop landing hero: an agent run drifts off the racing line, is flagged P1 off-task, and is steered back](docs/assets/hero.png)

<sub>Still from the landing-page mockup. The `pip install pitstop` shown there is planned; the package is not on PyPI yet.</sub>

## Why PitStop

Agents drift in two ways. They go **off-task**: chasing a stale directive, looping, or reporting a figure their evidence does not support. Or they **degrade**: still on the right task, but quality, tool calls or cost quietly get worse after a model, provider or prompt change. Some drift needs action within seconds (an off-task call to a tool with side effects), some can wait for a weekly fix. Existing observability tools stop at the alert; PitStop is meant to also steer the agent back in the moment and rewrite the prompt that caused the drift.

> **Status: pre-release, development paused after milestone 3a (November 2026).** The repo contains the drop-in Python SDK (drift detection, steering and hard dollar budgets), adapters for the OpenAI Agents SDK and LangChain, an OpenTelemetry span processor, the background `pitstop scan` pass, the CLI to inspect it, the local dashboard (`pitstop ui`, with a background scan worker), the research-swarm example watched by the SDK ([examples/financial-research](examples/financial-research/README.md)), the GEPA calibration tooling and benchmarks. The L3 run judge, L4 endpoint canaries and agent-prompt correction are on the [Roadmap](#roadmap), planned but not built.

## Contents

- [Why PitStop](#why-pitstop)
- [Quickstart (run the example)](#quickstart-run-the-example)
- [Add PitStop to your agent](#add-pitstop-to-your-agent)
  - [Pre-action gate](#pre-action-gate)
  - [Fallback](#fallback)
  - [Alerts](#alerts)
  - [Background checks: `pitstop scan`](#background-checks-pitstop-scan)
  - [What changed?](#what-changed)
- [Dashboard: `pitstop ui`](#dashboard-pitstop-ui)
- [OpenAI Agents SDK](#openai-agents-sdk)
- [LangChain / LangGraph](#langchain--langgraph)
- [Already tracing with OpenTelemetry?](#already-tracing-with-opentelemetry)
- [Set a hard budget](#set-a-hard-budget)
- [How it works today](#how-it-works-today)
  - [The SDK and the background pass](#the-sdk-and-the-background-pass)
  - [One run of the research swarm](#one-run-of-the-research-swarm)
  - [Calibrating the detector prompt](#calibrating-the-detector-prompt)
- [Where it's going (architecture)](#wheres-it-going-architecture)
  - [Runtime](#runtime)
  - [Detection, urgency and correction](#detection-urgency-and-correction)
- [Repo map](#repo-map)
- [Benchmarks](#benchmarks)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgements](#acknowledgements)

## Quickstart (run the example)

You need [uv](https://docs.astral.sh/uv/) and Python 3.11 or newer.

```sh
git clone https://github.com/Qamil-Mirza/gepa-financial-agent.git
cd gepa-financial-agent
uv sync
uv run finresearch demo
```

Then open http://127.0.0.1:8770. `finresearch` is the research-swarm example's CLI ([examples/financial-research](examples/financial-research/README.md)), installed by `uv sync` next to `pitstop`. The demo works offline: it replays a bundled snapshot of a real paired run, in which two Nemotron Nano supervisors (the original prompt and the GEPA-rewritten prompt) each watch and correct their own three research agents. Use `--port` if 8770 is taken, for example `uv run finresearch demo --port 8771`.

**Run the example offline.** The swarm itself is watched by the PitStop SDK with a few lines (`pitstop.workflow`, `pitstop.agent` per research agent, and the paid You.com call as a `side_effects=True` tool). `--offline` runs it with no keys and no network: a scripted worker model, canned research answers and a scripted judge for PitStop's pre-action gate. The default scenario tells the NVIDIA data-center agent, from step 2, to research consumer GeForce gaming instead; PitStop's gate blocks those research calls before they run:

```sh
uv run finresearch run --offline
```

```text
  agent_nvda_dc-e016     error         research blocked by PitStop: out_of_scope_action: {"gate": "about consumer gaming, not the directive", ...
...
agent_nvda_dc: drift injected at step 2 → PitStop blocked 2 research call(s), first at step 2: out_of_scope_action: {"gate": "about consumer gaming, not the directive", "prompt": "l1_gate_v2", "tool": "youcom_research"}
agent_amd_ai: no injection → not blocked by PitStop
agent_supply: no injection → not blocked by PitStop
PitStop run 7a210f33… (ok); `pitstop drifts` lists what PitStop caught
```

The blocked agent gets the reason instead of a research answer and re-plans; the workflow is `medium`, so PitStop blocks the call but does not halt the run. Then list what PitStop recorded:

```sh
uv run pitstop drifts
```

```text
                                                       Drifts (open)
┏━━━━┳━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┓
┃ id ┃ level ┃ workflow           ┃ drift_type          ┃ action ┃ occurrences ┃ first seen          ┃ last seen           ┃
┡━━━━╇━━━━━━━╇━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━┩
│ 1  │ P0    │ financial-research │ out_of_scope_action │ block  │ 2           │ 2026-10-22 20:29:39 │ 2026-10-22 20:29:39 │
└────┴───────┴────────────────────┴─────────────────────┴────────┴─────────────┴─────────────────────┴─────────────────────┘
```

`--no-inject-drift` runs the same scenario without the injection, and PitStop records no drift.

**With keys (paid calls).** To run the swarm live on Crusoe Managed Inference and the You.com Finance Research API:

```sh
cp .env.example .env       # fill in CRUSOE_API_KEY and YDC_API_KEY
uv run finresearch check       # one Crusoe inference call per configured model
uv run finresearch run
```

`finresearch run` requires both keys. It makes Crusoe inference calls and up to `PITSTOP_MAX_RESEARCH_CALLS` (default 12) live You.com calls per run; cached answers are reused for free. Live, each research call is judged by PitStop's gate judge (`[judge]` in `config.toml`; by default `openai/gpt-oss-120b` on Crusoe with `CRUSOE_API_KEY`). `--legacy-supervisor` also turns on the hackathon's per-step drift supervisor; with `--correct-on-drift`, a query it flags is blocked and the agent receives a correction instead.

Runs are written to `~/.pitstop/runs/<run_id>/`. Set `PITSTOP_HOME` to move the whole state directory, or `PITSTOP_RUNS_DIR` to move only the runs. Other commands:

| Command | What it does |
|---|---|
| `uv run finresearch timeline --run-dir <run dir>` | Rebuild and print the curated drift timeline of a run (`--highlights` for flags and responses only) |
| `uv run finresearch serve` | JSON + server-sent-events API over your runs' timelines, on port 8765 |
| `uv run finresearch compare --run-dir <run dir> --prompt-b <prompt.txt>` | Re-judge a recorded run's exact supervisor inputs with two prompts, side by side; no new research calls (requires keys) |
| `uv run finresearch run --legacy-supervisor --shadow-prompt <prompt.txt>` | Run a second supervisor prompt on the same inputs, logged but never acting (requires keys) |

## Add PitStop to your agent

The SDK works with any OpenAI-compatible client and with the Anthropic SDK (`pip install "pitstop[anthropic]"`). Wrap the client, mark the workflow, and guard your tools:

```python
import pitstop

client = pitstop.wrap(OpenAI(base_url=..., api_key=...))      # metered, drift-checked
# or: client = pitstop.wrap(Anthropic())                       # the same, for client.messages.*
SYSTEM = pitstop.prompt("support.system", "You are a support agent...")   # versioned; your edits win

@pitstop.tool                       # read-only: loops are steered, then halted
def search(query): ...

@pitstop.tool(side_effects=True)    # off-task calls to this are blocked
def send_report(to, body): ...

with pitstop.workflow("support", goal="Answer the ticket", criticality="medium",
                      budget_usd=1.0, tools=("search", "send_report")):
    run_agent(client)               # raises pitstop.Halted or pitstop.BudgetExceeded when it must stop
```

State lives in `~/.pitstop` (or `$PITSTOP_HOME`). If you pass a custom home instead, call `pitstop.init(home=...)` first, before any other PitStop call: the first wrapped call, tool, prompt or workflow otherwise initializes the default home. Catch `pitstop.BudgetExceeded` before `pitstop.Halted`, since it is a subclass.

Steering is counted per run, not per agent: parallel agents in one workflow (for example several `pitstop.agent(...)` scopes under `asyncio.gather`) share the run's budget of 2 steers, and the next steer after that halts the whole run.

`pitstop.workflow(...)` (and `pitstop.agent(...)`) can also decorate a generator or async generator function: the run opens on the first step and ends when the generator finishes, raises or is closed. Code that consumes the generator between yields runs outside the run. Each step runs in the consumer's own task, so an `asyncio.timeout` or task group that spans a `yield` inside the generator keeps working.

Metered methods (on the client, on `with_options(...)`/`copy()` clients, and under `with_raw_response` / `with_streaming_response` where the SDK offers them): `chat.completions.create`, `chat.completions.parse` and `chat.completions.stream`; `responses.create`, `responses.parse` and `responses.stream`; and legacy `completions.create`, sync and async. On an Anthropic client (`Anthropic` / `AsyncAnthropic`): `messages.create` (also with `stream=True`), `messages.parse` and `messages.stream`, and the same three under `beta.messages`, whose streams keep `text_stream`, `get_final_message()` and `get_final_text()` working as unwrapped. Anthropic cache writes are charged at their write premium, never under it: 1.25x the input price for 5-minute writes and 2x for 1-hour writes (2x when the response does not break them down), and cache reads at the cached price. A `stop_reason` of `"refusal"` counts as a refusal for drift detection. A `claude-*` model with no price entry is priced at a conservative $15 / $75 per million input / output tokens until you add a `[prices."<model>"]` override (see [docs/config.example.toml](docs/config.example.toml)). A stream helper (`.stream(...)`) reserves when it is called, settles when its `with` block exits, and releases its reservation if the manager is never entered. On a chat stream helper, `current_completion_snapshot` hides the usage PitStop asked for, so it returns a fresh copy on each access; `event.snapshot` on the stream's events is the SDK's own live object and shows that usage once the stream has received it. A `with_raw_response` stream and a `with_streaming_response` call can't be read by PitStop, so they are charged their full reservation; a `with_streaming_response` manager is charged when it is entered, and releases its reservation if it is never entered or entering it fails. A Responses call with `background=True` (or one that returns still `queued` / `in_progress`) is charged its full reservation, because its output is billed after the call returns and polling with `responses.retrieve` is not metered. A background stream read to its end settles from its real usage. Resuming a stored response with `responses.stream(response_id=...)` is metered as a new call, so it can count that response twice (over-counted, never under-counted). With `previous_response_id` or `conversation`, the earlier turns are billed as input but are not in the request, so the prompt estimate adds 32,000 tokens for them, which covers most conversations; a longer history can still be under-reserved, and the call settles from the provider's real usage. A legacy `completions.create` prompt given as token ids counts as that many tokens. On the Responses API, custom tool calls and hosted tool calls (web and file search, code interpreter, computer use, MCP, image generation, local shell) count as tool calls for drift detection but never as tool-call parse failures, and a `refusal` content part counts as a refusal. `beta.messages.tool_runner` is metered too: each model call the runner makes is reserved and settled as its own call. Not metered yet: the Assistants, Realtime and Batch APIs (including Anthropic `messages.batches`), and every other `client.beta.*` method (OpenAI's included). Every other method is passed through unchanged, so calls through those APIs are neither budgeted nor recorded.

Anthropic premium billing modes (fast speed, long-context premiums and server-tool fees such as web search) are not modelled: PitStop prices those calls at the standard token rates, so give such a model a `[prices."<model>"]` override with rates high enough to cover them to keep it within the hard budget.

Only calls on the default `agent` meter feed drift detection and set the run's model. PitStop's own judge, supervisor and GEPA calls inside a run are metered against the budget but never counted as the agent's behavior; each `llm_call` event records its `meter`.

Then run `uv run pitstop ui` to see your workflow on the pit wall: its open drifts, spend against budget and last run, with a live feed (setup in [Dashboard](#dashboard-pitstop-ui)).

### Pre-action gate

Before a `@pitstop.tool(side_effects=True)` tool runs inside a `high` or `medium` workflow, and after the L0 checks let it through, a judge model is asked whether this one call serves the workflow's goal (and the agent's goal, if set). It sees the goals, the tool name, its arguments (first 2,000 characters) and the agent's last 5 tool calls; the prompt is frozen and versioned (`l1_gate_v2`). The arguments and the recent calls reach the judge inside `<tool_args>` and `<recent_calls>` blocks that the prompt treats as data, never as instructions; a `</` inside them is written as `<\/`, so a tool argument cannot close its block and smuggle in instructions. The judge endpoint receives the tool's name and arguments and the agent's last 5 tool calls; choose an endpoint you trust with that data. Read-only tools and `low` workflows are never gated. A "no" is recorded as an `out_of_scope_action` drift (P0) and the tool returns the usual `{"blocked_by": "pitstop", "reason": ...}` instead of running; on a `high` workflow the run also halts at its next call boundary. Every verdict is a `gate` event.

Each gated call costs one judge call and adds roughly 0.5–1.5 s of latency. Judge calls go through the `watch` meter: they count against your budgets (a budget hit raises `BudgetExceeded` from the tool call) but never feed drift detection. Async tools run the judge in a worker thread, not on the event loop.

The judge is configured in `[judge]` in `config.toml` (default: `openai/gpt-oss-120b` on Crusoe, key read from `CRUSOE_API_KEY`, 10 s timeout, no retries; `base_url` must be an `http://` or `https://` URL with a host; see [docs/config.example.toml](docs/config.example.toml)). To use your own client, pass a sync OpenAI-compatible client: `pitstop.init(judge_client=OpenAI(...), judge_model="...")` (PitStop wraps it; an async client is refused and counts as no judge). Pick a model from a different family than your agent where you can, and prefer a fast non-reasoning model: a reasoning model spends tokens and seconds thinking before it answers (the judge call allows up to 2,048 output tokens). The judge is resolved once per runtime: a key exported later takes effect after a restart, `pitstop.sdk.runtime.reset_runtime()` or `pitstop.init(judge_client=...)`.

When the judge is missing, times out, errors or returns no usable verdict, the workflow's failure policy decides: `high` workflows block the call (fail closed) and `medium` workflows run it (fail open). Override it per workflow with `pitstop.workflow(..., on_gate_error="block" | "allow")`. Every failure is recorded as a `gate_error` event with the policy applied; a blocked call's reason says how to fix it (configure `[judge]` or set `on_gate_error="allow"`). So with no judge configured, side-effect tools on `high` workflows do not run.

Try it without keys: `uv run python examples/quickstart/agent.py --offline --inject-loop` (see [examples/quickstart](examples/quickstart/README.md)). Then inspect the local state:

```text
$ uv run pitstop drifts
 id  level  workflow    drift_type         action  occurrences
 1   P1     quickstart  unproductive_loop  steer   3
$ uv run pitstop status      # per-workflow runs, open drifts by level, spend and last status
$ uv run pitstop budget      # configured budgets: spent, limit, remaining
```

The `occurrences` column counts every occurrence of the drift; the DB keeps the newest 20 evidence entries per drift.

### Fallback

Pin a known-good model on a workflow with `pitstop.workflow(..., fallback_model="...")`. The fallback engages when all of these hold:

- a P1 `tool_call_breakage` or `refusal_spike` drift is detected in a run (a `low` workflow's P2 never engages it);
- the run's model changed in the last 24 hours: it differs from the model of the workflow's previous setup, and the first run on the new model started less than a day ago (a breakage with no recent change is only recorded). Runs that went to the pinned model don't count as a setup, so clearing the fallback and going back to your old model is not a change, while a switch you made while the fallback was on still is;
- the run is not already on the pinned model.

From the next call on, every agent-meter call in that workflow is sent to the pinned model instead of the one your code asked for; the agent sees no other difference. The reservation, cost and `llm_call` event use the pinned model, and the event records the requested one as `fallback_from`. The drift's action is `fallback`, and a `fallback` event records `{"from", "to", "change"}`. Judge and supervisor calls (other meters) and calls outside the workflow are never swapped. If PitStop cannot read its state, the call keeps its own model.

The fallback handles model changes only. A change of provider `base_url` with the same model still records the P1 drift and alerts, but does not engage the fallback: the pinned model id may mean nothing to the new provider. Only the model id is swapped: the call still goes to the client's own `base_url`, so pick a `fallback_model` that every endpoint the workflow calls actually serves. An `llm_call` event of a call that named no model records `fallback_from` as `"(unset)"`.

The fallback is sticky: it stays on across runs and processes until you clear it, since only you can tell when the new model or provider is fixed:

```text
$ uv run pitstop fallback status        # workflow, pinned model, since, the change that engaged it
$ uv run pitstop fallback clear support # the next call goes back to the requested model
```

Once cleared, `pitstop drifts` shows the drift's action as `fallback (cleared)`.

### Alerts

Drifts are routed by urgency (P0-P3, see the [design spec](docs/superpowers/specs/2026-10-20-pitstop-runtime-design.md) §5):

| Level | Meaning | Default route |
|---|---|---|
| **P0** | Act now: off-task side-effect tool, budget hit, loop on a high-criticality workflow | PagerDuty, Opsgenie, Slack, webhook (sent right away) |
| **P1** | Hours: tool-call or refusal spike, quality step after a model/provider change | Slack, email, webhook (sent right away) |
| **P2** | This week: quality degrading within contract | Daily digest: email, Slack, webhook |
| **P3** | Trend: topic shift, slow token or latency creep | Daily digest: email, Slack, webhook |

Every alert says what drifted, what likely changed (for a drift from [`pitstop scan`](#background-checks-pitstop-scan), the run tags L5 attribution found changed; otherwise the run's model or provider change in the last 24 hours, or "no change detected" with the run's model and `base_url`) and what PitStop did (blocked, steered, halted, fell back or recorded). P0/P1 alerts are sent from a background thread, so a slow channel never blocks your agent, and it finishes sending before the process exits (a normal exit or an uncaught exception such as `Halted`; not a kill). Waiting for it can delay the exit: by up to about 16.5 s for each unreachable HTTP channel and 31.5 s for each unreachable SMTP channel (3 attempts with a 5 s or 10 s timeout, plus the backoff), since an alert's channels are sent one after another. Each open drift alerts once per channel (deduplicated per workflow and drift type); a drift that escalates from P1 to P0 alerts again. A P0/P1 send that fails with a network error, a 5xx or a 429 is retried twice (after 0.5 s and 1 s); an SMTP refusal (authentication, sender, recipients or message rejected) and any other HTTP status are not retried; a send that still fails is retried at the drift's next occurrence, and a send left unfinished by a killed process stops blocking after 2 minutes. Each attempt is a row in the `alerts` table (`sent` or `failed` with an error such as `HTTP 500`).

A channel is used only when its `[alerts.<channel>]` table is in `config.toml` **and** its env var is set. Secrets live only in environment variables: `config.toml` holds the variable's name, and an inline `routing_key`, `api_key`, `webhook_url`, `password` or `url` is rejected as an invalid config. PitStop never stores or logs a secret (it hides httpx's own request log line while sending), but third-party HTTP instrumentation in your process can still record the full Slack or webhook URL: for example Sentry's httpx breadcrumbs or the OpenTelemetry HTTPX instrumentor. Scrub those URLs there, or keep that instrumentation off in processes that send alerts.

| Channel | Table | Secret env var (default) | Notes |
|---|---|---|---|
| PagerDuty Events v2 | `[alerts.pagerduty]` | `PAGERDUTY_ROUTING_KEY` (`routing_key_env`) | P0 critical, P1 error; SMS and phone calls go through PagerDuty or Opsgenie |
| Opsgenie | `[alerts.opsgenie]` | `OPSGENIE_API_KEY` (`api_key_env`) | `region = "us"` or `"eu"` |
| Slack incoming webhook | `[alerts.slack]` | `SLACK_WEBHOOK_URL` (`webhook_url_env`) | the webhook URL is a secret |
| Email (SMTP) | `[alerts.email]` | `SMTP_PASSWORD` (`password_env`, only with `username`) | `smtp_host`, `smtp_port` (587), `starttls` (true), `from_addr`, `to` |
| Generic webhook | `[alerts.webhook]` | `PITSTOP_WEBHOOK_URL` (`url_env`) | JSON POST of the alert plus `title` and `body`; any 2xx is success |

Change the routes with `[alerts.routes]` (`P0`, `P1` and `digest`, each a list of channel names). Check your setup, then schedule the digest:

```text
$ uv run pitstop alerts test                  # one test alert per channel: sent, failed: <reason> or not configured
$ uv run pitstop alerts test --channel slack
$ uv run pitstop alerts digest                # open P2/P3 drifts not yet sent; at most once per 24 h (cron it);
                                              # exits 1 when every channel failed
$ uv run pitstop alerts digest --force        # send now regardless of the last digest
```

### Background checks: `pitstop scan`

The checks above run inside each call. `pitstop scan` looks across runs: it reads each finished run once and feeds its per-run numbers into one change detector per workflow and metric (L2, a two-sided tabular CUSUM). Each detector's state is saved in the local DB (the `l2_state` table), so it carries over between passes and processes.

| Metric | Per run | Drift type | Alarms when it goes |
|---|---|---|---|
| `tool_error_rate` | errored tool results / tool calls (runs with tool calls) | `tool_call_breakage` | up |
| `refusal_rate` | refused agent calls / agent calls | `refusal_spike` | up |
| `tokens_per_call` | mean prompt + completion tokens per agent call | `cost_or_latency_creep` | up |
| `llm_calls` | number of agent calls | `cost_or_latency_creep` | up |
| `cost_usd` | agent spend (OTel and observed estimates included) | `cost_or_latency_creep` | up |
| `duration_s` | run wall time | `cost_or_latency_creep` | up |
| `output_chars` | mean characters per agent reply | `quality_drop` | up or down |

How a detector behaves:

- **It learns first.** The first 30 runs that report a metric only learn its baseline (mean and spread); nothing can alarm before that. The baseline is then frozen.
- **Then it accumulates evidence.** Each later run adds how far it sits from the baseline (in baseline standard deviations, minus a 0.5 slack) to a running sum; an alarm fires when the sum passes 10 / 8 / 6 for sensitivity `low` / `normal` / `high`. A large jump alarms on its first run; a smaller one takes a few runs.
- **It is tuned against crying wolf.** The spread is widened for a 30-run sample and floored: 0.15 for the rates, so on a zero baseline one fully failed run does not alarm at `normal` or `low` sensitivity but two in a row do (`high` alarms on the first); 5% of the baseline for the others, with a small minimum, so a flat baseline does not alarm on its first different value. In simulation at `normal`, about 3% of steady two-sided streams (under 1% of up-only ones) alarm at least once in 300 runs, and a 3-standard-deviation step is caught within 10 runs, about 5 runs after it starts on average. Small, slow drifts can go unflagged.
- **After an alarm it relearns.** The detector resets and learns a new baseline from the next 30 runs, so the same metric does not alarm again until then.

On an alarm, `scan` runs L5 attribution around the run where the shift began (see [What changed?](#what-changed)) and records a drift with action `record`, deduplicated with an open drift of the same workflow and drift type. The level follows the urgency table: `tool_call_breakage` and `refusal_spike` are P1 (P2 on `low` workflows), the others P2 (P3 on `low`). When the attribution names a `model`, `base_url` or `prompt_versions` change, the drift is raised to P1 (P2 on `low` workflows), never lowered. P0/P1 drifts alert right away, with the attribution summary as the "what likely changed" line; P2/P3 go in the digest.

Run it from cron, let it loop, or let the [dashboard's worker](#dashboard-pitstop-ui) run it every minute.

```text
$ uv run pitstop scan                 # one pass over up to 500 finished, unscanned runs
scanned 45 runs, 2 alarms
P1 research cost_or_latency_creep (tokens_per_call): model changed from m1 to m2 (100% of runs since 2026-10-22T14:35Z)
P1 research cost_or_latency_creep (cost_usd): model changed from m1 to m2 (100% of runs since 2026-10-22T14:35Z)
$ uv run pitstop scan --every 300     # a pass every 5 minutes until Ctrl-C
```

Runs still in progress wait for a later pass. `refused` and `dropped` runs are marked scanned but add nothing. Two scanners at once (two cron jobs, or a cron job and `--every`) are safe: each run is claimed and fed to its detectors in one transaction, so it is counted once. The drifts appear in `pitstop drifts`, and `pitstop explain <drift-id>` repeats the attribution with the metric's mean before and after; the drift's evidence holds the metric, the direction, the alarming value, the baseline mean and spread, the change point (`change_at`) and the attribution.

### What changed?

`pitstop explain <drift-id>` (L5 attribution) compares the drift's workflow runs just before a point in time with the runs from that point on, and names the run tags that changed: `model`, `base_url`, `prompt_versions`, `pitstop_version` and `adapter`. The point is the change point in the drift's newest evidence (`change_at`) when it names one, else when the drift was first recorded. Up to 50 runs are compared on each side, the nearest to the point; `refused` and `dropped` runs are left out. A tag value counts as a change when it has at least half of the runs after and at most 10% of the runs before; a value that was already common, or that only some runs after use, is not a change.

```text
$ uv run pitstop explain 7
model changed from m1 to m2 (100% of runs since 2026-10-22T14:05Z)
┃ tag   ┃ before ┃ after ┃ share before ┃ share after ┃
│ model │ m1     │ m2    │ 0%           │ 100%        │
tokens_per_call mean: 120 over 30 runs before, 240 over 20 runs after
```

Times are UTC. `prompt_versions` reads as the run's prompt versions in JSON with sorted keys (`{"researcher": 1}`), and a tag with no value before reads `(unset)`. When no tag changed, it prints `no run tag changed around <time>; likely an input or upstream shift`; with no earlier runs, `no runs before <time> to compare`. The mean line appears when the drift's evidence names a metric (`metric`); otherwise it prints how many runs were compared. An unknown drift id exits 1.

## Dashboard: `pitstop ui`

A local dashboard over the same SQLite store the SDK writes. From a clone of the repo, build the UI once (Node 22) and start it:

```sh
uv sync                                           # pitstop, with the ui extra's packages (fastapi, uvicorn)
npm --prefix ui ci && npm --prefix ui run build   # the dashboard, into src/pitstop/server/static/
uv run pitstop ui                                 # http://127.0.0.1:8787, opens your browser (--no-open to skip)
```

Without the build, the server answers with a "Dashboard not built" page that lists those two npm commands. Once PitStop is published, `pip install "pitstop[ui]"` installs it with the dashboard already built.

`pitstop serve` is the same server without opening a browser. Both load a `.env` from the current directory like every pitstop command, and neither the server nor its worker makes a model call. An empty store shows a setup snippet and the demo command instead of the pit wall.

- **Pit wall.** One tile per workflow, those that need you first, with its open drifts by level, spend against its budget and the last run's status, plus a live feed of drifts, halts, fallbacks, refused runs and model changes.
- **Workflow page.** A Run graph tab (the agents, tools and calls of a time window or one run, with per-node latency and the checks that fired; click a node for its details), a What changed tab (cost, tokens and error series with markers where a model, base URL or prompt version changed) and a Runs tab. The right column lists the workflow's open drifts: **Explain** shows the L5 attribution (as `pitstop explain`) and **Dismiss** closes a drift. A later occurrence of the same finding opens a new one.
- **Run detail.** One run's facts, graph and event timeline.
- **Settings.** Read-only: the effective config (workflows, budgets, judge, alert channels and routes, worker, where data lives). Secrets show as the name of the env var and whether it is set, never the value. Change settings in `config.toml`.

Quality scores and the Endpoints page arrive with the L3 run judge and L4 canaries (milestone 3b).

**The worker replaces a cron `pitstop scan`.** While the server runs, a background worker runs `scan` every 60 seconds (`--scan-every SECONDS`) and the P2/P3 digest check hourly. `--no-worker` turns it off, for example when cron already runs `pitstop scan`.

**Offline demo.** `dashboard_demo.py` records realistic runs through the real SDK with scripted models: no keys, no network. It refuses a home that already has a store.

```sh
uv run python examples/quickstart/dashboard_demo.py --home /tmp/pitstop-demo
PITSTOP_HOME=/tmp/pitstop-demo uv run pitstop ui
```

**Host safety.** The server binds `127.0.0.1` and has no login. The dashboard shows prompts and tool arguments, so keep it on loopback. A non-loopback `--host` prints a warning and exposes the dashboard to that network. A wildcard `--host` (`0.0.0.0`, `::`) is refused (exit 2): pass a concrete address, such as the machine's LAN IP, and browse to it. The server only accepts requests whose `Host` is a loopback name or the exact `--host` value; any other gets a 400. Writes (Dismiss) need an `X-PitStop: 1` header, and the server sends no CORS headers, so another web page cannot write to it. After startup the API returns no stack traces; a locked store answers 503.

## OpenAI Agents SDK

Install `pip install "pitstop[openai-agents]"`. Guard the agent's tools, pass the run hooks, and give the SDK a wrapped client:

```python
import pitstop
from agents import Agent, Runner, function_tool, set_default_openai_client
from openai import AsyncOpenAI
from pitstop.sdk.adapters.openai_agents import Hooks, guard

set_default_openai_client(pitstop.wrap(AsyncOpenAI()), use_for_tracing=False)   # metered, budgeted
# or per agent: model=OpenAIResponsesModel("gpt-5", openai_client=pitstop.wrap(AsyncOpenAI()))

agent = guard(Agent(name="support", instructions="...", tools=[search, send_email]),
              side_effects={"send_email"})

with pitstop.workflow("support", goal="Answer the ticket", criticality="medium", tools=("search", "send_email")):
    result = await Runner.run(agent, question, hooks=Hooks())   # raises pitstop.Halted / BudgetExceeded
```

`guard(agent, side_effects=...)` adds PitStop's tool input guardrail to each of the agent's function tools (once, however often you call it) and returns the same agent; the tools named in `side_effects` are side-effect tools, and a name that is none of the agent's function tools raises `ValueError`. Each tool call then goes through the same checks as a `@pitstop.tool` call before it runs: a pending halt, the scope and loop checks, and the pre-action gate for side-effect tools on `high` and `medium` workflows. A blocked call doesn't run: the model gets `{"blocked_by":"pitstop","reason":...}` as the tool's result (a steer gets the corrective text). A tool whose body is itself a `@pitstop.tool` of the same name is checked once, not twice, and each call records one result. `guard` changes the tools in place, so a tool shared with another agent is guarded there too; call `guard` on each agent of a handoff chain. Use `guard`, not the bare guardrail: `tool_guardrail(side_effects=...)` (for `function_tool(..., tool_input_guardrails=[...])`) doesn't mark the tool body as checked or record its results, so around a `@pitstop.tool` of the same name it checks each call twice and counts loops at half the threshold.

`Hooks()` names the running agent: the agent's model calls, tool calls and halt events carry its name (after a handoff, the new agent's), and the run is tagged `adapter = openai-agents`. When an agent run as a tool (`agent.as_tool(..., hooks=Hooks())`) ends, the name goes back to the agent that called it. The named agent keeps the goal of the `pitstop.agent(..., goal=...)` scope `Runner.run` runs in, and the pre-action gate judges side-effect calls against it (against the workflow goal when there is no such scope). A `pitstop.agent(...)` scope entered inside the run still names its own calls. Two `Runner.run` calls running at once in the same scope share one name (the agent that started last, until it ends; no name outlives both runs); give each its own `pitstop.agent(...)` scope to keep them apart. Hooks also raise a pending halt before the next model call, even when the model isn't metered. Subclass `Hooks` (and call `super()`) to add your own run hooks.

`pitstop.Halted` and `pitstop.BudgetExceeded` reach the caller of `Runner.run` when a guarded call, a hook or a wrapped model call raises them. Their class there is also an `agents.exceptions.AgentsException`, the only kind the SDK re-raises unchanged; `except pitstop.Halted` works as usual. A budget hit inside a tool body is different: the SDK hands that `BudgetExceeded` to the model as an error message, so PitStop raises the halt it left pending once the tool returns, and it arrives as `pitstop.Halted`, not `pitstop.BudgetExceeded` (catch `pitstop.Halted` to cover both).

Limits: only function tools are guarded, because the SDK runs tool input guardrails for function tools only; hosted tools (web search, file search, code interpreter, computer use), MCP server tools and handoffs are not checked. Model calls are metered and budgeted only through a `pitstop.wrap`-ed client. `guard` replaces each guarded tool's `on_invoke_tool`, so `tool.__wrapped__` no longer reaches the original function.

## LangChain / LangGraph

Install `pip install "pitstop[langchain]"`. For a LangChain v1 agent (`create_agent`, which runs on LangGraph), add PitStop's middleware, and for hard budgets give the chat model a wrapped client:

```python
import pitstop
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from openai import AsyncOpenAI, OpenAI
from pitstop.sdk.adapters.langchain import PitStopMiddleware

model = ChatOpenAI(model="gpt-5", client=pitstop.wrap(OpenAI()).chat.completions,
                   async_client=pitstop.wrap(AsyncOpenAI()).chat.completions)   # metered, budgeted

agent = create_agent(model, [search, send_email],
                     middleware=[PitStopMiddleware(side_effects={"send_email"}, agent="support")])

with pitstop.workflow("support", goal="Answer the ticket", criticality="medium", tools=("search", "send_email")):
    result = agent.invoke({"messages": [{"role": "user", "content": question}]})   # raises pitstop.Halted
```

Each tool call then goes through the same checks as a `@pitstop.tool` call before it runs: a pending halt, the scope and loop checks, and the pre-action gate for the tools named in `side_effects` on `high` and `medium` workflows. A blocked or steered call doesn't run: the model gets a `ToolMessage` with `status="error"` whose content is `{"blocked_by":"pitstop","reason":...}` (or the corrective text of a steer). A tool whose body is itself a `@pitstop.tool` of the same name is checked once, and each call records one result. The middleware also raises a pending halt before each model call, so `pitstop.Halted` (and `BudgetExceeded` from a wrapped client) comes out of `invoke` / `ainvoke`. `agent=` names the agent for the run's model calls, tool events and halt events; it keeps the goal of the enclosing `pitstop.agent(..., goal=...)` scope, which the gate judges side-effect calls against (the workflow goal when there is none). The run is tagged `adapter = langchain`. Outside a `pitstop.workflow(...)` the middleware does nothing. A `side_effects` name that is none of the agent's tools is logged as a warning once.

What is enforced and what is only observed: tool calls are enforced (blocked, steered, halted). Model calls are budgeted only through a `pitstop.wrap`-ed client as above, and such a call is recorded once, by `wrap`, whatever its ledger meter (`pitstop.sdk.wrap.meter(...)`). A model call that no wrapped client metered (any other chat model, or `ChatOpenAI` without a wrapped client) is still recorded, as an `llm_call` event marked `"observed": true`, named by the model the request named (the chat model's `model_name` or `model`, else the one the reply names), with the tokens the reply reports and a cost estimated from the price table, and the L0 checks run on it (refusal spikes, and invalid tool calls as tool-call breakage). Its cost never reaches the budget ledger, so it can't trip a budget or the spend-spike halt.

Limits: only tool calls that reach the agent's tool node are guarded; tools the provider runs server side are not. A tool call that pauses on a LangGraph `interrupt()` is checked again by the guard when it resumes, so it counts twice toward the loop checks. A plain LangGraph `StateGraph` without `create_agent` doesn't use middleware: decorate its tools with `@pitstop.tool` and give its model a `pitstop.wrap`-ed client.

Known limitations of observed model calls: PitStop records one observed event per model-node turn, so when one turn of the model node makes several unwrapped model calls (a retry or fallback middleware inside PitStop's, say), only the reply the turn returns is recorded. And a handler that mixes wrapped and unwrapped calls in one turn hides the unwrapped one: once a wrapped client has metered a call in that turn, the turn's unwrapped calls are not observed.

## Already tracing with OpenTelemetry?

If your agent is already traced with OpenTelemetry GenAI spans (Langfuse, Logfire, OpenLLMetry, OpenInference/Phoenix or the GenAI semantic conventions), PitStop can read those spans without touching your agent code. Install `pip install "pitstop[otel]"` and add its span processor to your tracer provider:

```python
from pitstop.otel import SpanProcessor

provider.add_span_processor(SpanProcessor(workflow="support"))   # before your exporter's processor
```

Each trace with at least one GenAI span (an LLM, tool or agent span) becomes one PitStop run (id `otel-<trace id>`, shown by `pitstop status` and `pitstop drifts` like any other run); a trace of plain spans, such as an HTTP health check, writes nothing. The run ends `ok` or `error` with its root span. At most 1,000 traces are followed at once: past that the oldest is dropped and its run ends with status `dropped`. The workflow name is the root span's `pitstop.workflow` attribute (set it when the span starts), else the `workflow=` argument, else the root span's name; `goal`, `criticality`, `sensitivity` and `tools` work as in `pitstop.workflow(...)`. LLM spans become `llm_call` events with their model, token counts and a cost estimated from the price table; tool spans become `tool_call` and `tool_result` events; agent spans become `agent` events, and each event names the agent it ran under. An agent name set on a span after it started is read again when the span ends, so it names the span's own event and the spans that end after it. A GenAI span that ends after its root span still writes its events to the run when the trace already has one, and the run keeps the status its root gave it. The first LLM span that names a model sets the run's model and its `base_url`, built as `https://<server.address>`. The L0 checks run on the trace: unproductive tool loops, out-of-scope tools (checked as read-only, since a span does not say whether a tool had side effects) and refusal spikes (a `content_filter` or `refusal` finish reason). Tool-call breakage is not detected: spans carry no tool-call parse-failure signal. Drifts are recorded and P0/P1 alerts are sent as usual. The checks that span several calls (loops, refusal spikes) and the verdict events need those calls to share a root span, for example an agent or request span around them: a bare instrumented LLM call with no parent is a one-span trace of its own.

This is observe-only: the spans are read after they end, so PitStop can't block, steer, halt or fall back, and the estimated costs never count against a budget. OTel runs never drive the actions of SDK runs of the same workflow either: they never count as the model change that engages a fallback, as the previous run whose endpoint budget refuses a run, or in the spend-spike baseline. For hard budgets and blocking, use `pitstop.wrap` and `@pitstop.tool` as above. Spans that run inside a `pitstop.workflow(...)` are skipped, since `wrap` and `@pitstop.tool` already record them. A GenAI-kind root span (an `invoke_agent` span, say) that wraps a `pitstop.workflow(...)` block is still recorded, as a second (OTel) run: keep the workflow block outside the request span, or give the processor a different `workflow=` name. The processor never raises into your app: a malformed span or a locked store costs you at most one logged warning per kind of failure (reading a span, writing the store) for the processor's lifetime.

Every drift PitStop records on a trace adds a `pitstop.verdict` event to the trace's root span while it is still open, so the trace you already export shows what PitStop decided: `pitstop.drift_type`, `pitstop.level`, `pitstop.action` (always `record` here), `pitstop.run_id`, `pitstop.drift_id` and `pitstop.reason`. Add the PitStop processor before the exporting processor so the root span is exported with those events. The same event is added to the current span for drifts found by `wrap` and `@pitstop.tool`. `pitstop.reason` is the drift's evidence, up to 300 characters, and can include a tool's arguments, which then reach your tracing backend.

## Set a hard budget

Copy [docs/config.example.toml](docs/config.example.toml) to `~/.pitstop/config.toml`. Each `[budgets.<name>]` entry sets a dollar limit for a provider `base_url` and period. Every model call reserves its worst-case cost first, and once the limit would be crossed the call raises `BudgetExceeded` before it reaches the provider. PitStop's own calls (supervisor, judge, GEPA) count against the same budget. A per-workflow cap is the `budget_usd` argument of `pitstop.workflow`. An invalid `config.toml` (a bad period, limit or timezone) never silently removes a budget: `pitstop.init()` raises a `ValueError`, every metered call, `pitstop.prompt` and `pitstop.workflow` (and any guarded tool inside a workflow) raises `pitstop.Halted("pitstop config invalid: ...")` before reaching the provider, and `pitstop status`/`drifts`/`budget` print the error and exit 2. A `pitstop.db` written by a newer PitStop is refused the same way, so a downgrade never runs without its budgets.

Once a budget is used up, new runs are refused at the start: entering `pitstop.workflow(...)` raises `BudgetExceeded` when the workflow's own `budget_usd` for today, a budget without a `base_url`, or the budget for the endpoint the workflow's previous run used is already spent. The refused run is recorded with status `refused`, next to a P0 `budget_exhausted` drift; each refused entry attempt is its own `refused` run row, counted in the `runs today` column of `pitstop status` (whose `last status` then reads `refused`). A budget for an endpoint the run has not used yet still stops the run at its first call to it. When the store can't be read at that moment, PitStop logs a warning and the run starts; the per-call check still applies.

A runaway run halts before it reaches the limit. After each of the run's agent calls is settled, PitStop compares the run's spend so far with its workflow's baseline: the median agent spend of the workflow's last 20 runs that finished `ok`, floored at $0.01. Past 10x (sensitivity `low`), 5x (`normal`) or 3x (`high`) the baseline, the run gets a P0 `spend_spike` drift with action `halt`, whatever its criticality, and its next wrapped call raises `Halted`. A workflow with fewer than 10 `ok` runs has no baseline yet and is never flagged. Judge, supervisor and GEPA calls don't count toward the run's spend.

A process that dies mid-call leaves its worst-case reservation counting against the budget. `pitstop budget reconcile` lists reservations still open after 60 minutes (`--older-than-min N` to change that): they were left by processes that died or by calls still running. `--release` frees them and lists what it freed, so only use it once those calls are done.

## How it works today

### The SDK and the background pass

Your code calls `pitstop.wrap` (OpenAI and Anthropic clients), `workflow`, `agent` and `@tool`, or adds one of the adapters or the OpenTelemetry span processor. Each step is written to a local SQLite file (`pitstop.db` under `~/.pitstop`). Inline, in your process:

- the L0 code checks (loops, scope, refusals, tool-call breakage) and, for `side_effects=True` tools on high and medium workflows, the L1 pre-action gate run before the call; the resulting action is steer, block or halt;
- the hard dollar budget ledger, which refuses a call (or a whole new run) once a limit is used up, and the spend-spike halt;
- the pinned-model fallback and P0/P1 alerts.

The OpenAI Agents SDK and LangChain adapters apply the same tool guard to those frameworks. The OpenTelemetry span processor is observe-only: it turns GenAI spans into runs and records drifts without blocking anything or touching budgets. Every recorded drift also adds a `pitstop.verdict` span event.

`pitstop scan` is the background pass. It reads each finished run once, computes per-run metrics, runs the L2 change-point detector per workflow and metric, and records an alarm as a drift with its L5 attribution (`pitstop explain <drift-id>`). It runs when you call it (or on an interval with `--every`); there is no always-on server yet.

### One run of the research swarm

The research swarm is an example built on PitStop, in [examples/financial-research](examples/financial-research/README.md) (package `finresearch`). A scenario gives three research agents a shared objective. By default only the PitStop SDK watches it: each paid research call is a `side_effects=True` tool that PitStop's pre-action gate judges against the agent's directive (see the offline run above). With `finresearch run --legacy-supervisor`, the hackathon's per-step supervisor also runs, as follows. Before each agent's proposed research query is sent, the supervisor judges it. Code computes trajectory signals (for example, near-duplicate queries and steps since the last finding), a Nemotron model returns a verdict (ON_TRACK, DRIFTING or UNCERTAIN) with the evidence event ids it relied on, and code rejects evidence ids that do not exist in the trajectory. The alert policy fires on a confident DRIFTING verdict (confidence at least 0.75) or on two DRIFTING verdicts in a row at confidence at least 0.5.

```mermaid
flowchart TD
    S["Scenario<br/>objective + 3 agents"] --> W["Worker agent<br/>proposes a research query"]
    W --> SUP["Supervisor<br/>trajectory signals + LLM verdict<br/>+ evidence-id validation"]
    SUP --> P{"AlertPolicy<br/>fires?"}
    P -- "no" --> R["Run the query<br/>(You.com Finance Research)"]
    R --> F["Extract findings"]
    F --> W
    P -- "yes, --correct-on-drift" --> C["Block the query,<br/>send a correction"]
    C --> W
    P -- "yes, --halt-on-drift" --> H["Halt the agent"]
    F --> RS["Report synthesizer<br/>uses only unflagged findings"]
    RS --> D[("Run dir<br/>events.jsonl<br/>decisions.jsonl<br/>timeline.jsonl<br/>meta.json, report.md")]
    SUP -.-> D
    D --> T["finresearch timeline<br/>finresearch serve"]
    D --> UI["Demo UI<br/>(finresearch demo)"]
```

### Calibrating the detector prompt

The supervisor's prompt is calibrated offline with GEPA on labeled hard pairs: 18 drift categories, each with a drift case and a healthy twin. Train is NVIDIA, val is Broadcom and test is AMD; GEPA never sees test. The gate promotes a candidate only if its test accuracy beats the current prompt and it raises no more false alarms.

```mermaid
flowchart LR
    HP[("Hard pairs<br/>train / val / test")] --> G["GEPA<br/>pitstop.correct.gepa<br/>reflects on train, selects on val"]
    SEED["Current prompt<br/>supervisor_v0.txt"] --> G
    G --> CAND["Candidate prompt"]
    CAND --> GATE{"Gate on test<br/>accuracy up?<br/>false alarms not up?"}
    GATE -- "pass" --> PROMO["Promoted detector prompt<br/>becomes the current prompt"]
    GATE -- "fail" --> KEEP["Keep the current prompt"]
```

The code is in [benchmarks/calibration](benchmarks/calibration): `loop.py` runs the recursive swarm → judge → mine → optimize → gate loop, and `optimize.py` runs a single GEPA optimization whose output you compare with `benchmarks.hard_pairs.run_hard_cases`. The GEPA engine itself is in [src/pitstop/correct](src/pitstop/correct). The hackathon's GEPA prompts are in `examples/financial-research/finresearch/assets/prompts/gepa/`; `finresearch run --legacy-supervisor` defaults to the hand-written `supervisor_v0.txt` (pass `--legacy-supervisor <prompt.txt>` to use another).

## Where it's going (architecture)

> **Planned (milestones 3b–4).** This section describes the design. The SDK, `pitstop scan` and the local dashboard server are covered above. L3, L4 and the correction loop do not exist yet. Full design: [docs/superpowers/specs/2026-10-20-pitstop-runtime-design.md](docs/superpowers/specs/2026-10-20-pitstop-runtime-design.md).

### Runtime

A few lines in your code (`wrap`, `@tool`, `workflow`) record every step to a local SQLite file. Inline checks and budgets run in your process and never depend on the server. A separate server process runs the heavier detection layers, correction and alert routing, and serves a local dashboard.

```mermaid
flowchart LR
    subgraph USER["Your process"]
        SDK["wrap / @tool / workflow<br/>L0 code checks<br/>L1 pre-action gate<br/>budget ledger<br/>P0 alerts"]
    end
    DB[("SQLite (WAL)<br/>pitstop.db under ~/.pitstop")]
    subgraph SERVER["pitstop server process"]
        WK["Background worker<br/>L2 change-point stats<br/>L3 run judge<br/>L4 endpoint canaries<br/>L5 attribution<br/>GEPA correction + gate<br/>alert routing (P1–P3)"]
        DASH["Local API +<br/>dashboard"]
    end
    SDK -- "writes" --> DB
    DB --> WK
    WK --> DB
    DB --> DASH
```

### Detection, urgency and correction

Each layer emits signals that map to an urgency level, and the level decides the in-the-moment action. Confirmed P1/P2 drift starts the correction loop, which rewrites only prompts registered with `pitstop.prompt` and scores candidates with a task-quality scorer, never with the drift detector.

```mermaid
flowchart TD
    IN["Inline, in your process<br/>L0 code checks<br/>L1 pre-action gate"] --> U{"Urgency"}
    BG["Background, in the server<br/>L2 change-point stats<br/>L3 run judge<br/>L4 endpoint canaries<br/>L5 attribution"] --> U
    U --> P0["P0: act now"]
    U --> P1["P1: hours"]
    U --> P2["P2: this week"]
    U --> P3["P3: trend"]
    P0 --> BH["Block the tool<br/>or halt the run"]
    P1 --> FB["Fall back to<br/>the pinned model"]
    P3 --> LOG["Log; add examples<br/>to the eval set"]
    U -- "off-task step,<br/>no side effects" --> STEER["Steer: skip the step,<br/>send a correction,<br/>agent re-plans"]
    P1 --> FR
    P2 --> FR
    subgraph CL["Correction loop"]
        FR["Flagged + healthy runs"] --> GE["GEPA rewrites<br/>the managed prompt"]
        GE --> GT{"Gate: held-out score up,<br/>no negative flips,<br/>off-task flags not up,<br/>size limit"}
        GT -- "pass" --> MODE{"manual or auto"}
        GT -- "fail" --> DROP["Discard"]
        MODE --> CAN["Canary"]
        CAN -- "holds" --> PRO["Promote"]
        CAN -- "underperforms" --> RB["Roll back"]
    end
```

## Repo map

```
src/pitstop/          the installable package (the SDK only) and the `pitstop` CLI
  sdk/                the drop-in SDK: wrap, workflow, agent, tool, prompt, the shared tool guard (guard.py)
  sdk/adapters/       the OpenAI Agents SDK (openai_agents.py) and LangChain (langchain.py) adapters
  otel/               the OpenTelemetry span processor and the pitstop.verdict span events
  detect/             L0 checks, L1 gate, L2 per-run metrics and CUSUM, L5 attribution, spend spike, `scan`
  server/             the local dashboard: FastAPI app and endpoints, run-graph and change views, the scan worker (`pitstop[ui]`)
  budget/             the budget ledger
  alerts/, store/     alert channels and routing, the SQLite store
  correct/            the GEPA engine (pitstop.correct.gepa) and its model clients
  llm.py              the OpenAI-compatible client used by the example and the benchmarks
  assets/prompts/     the L1 gate prompt
examples/quickstart/  a keyless demo agent that shows steer-then-halt, and dashboard_demo.py, which fills a home for the dashboard
ui/                   the dashboard source (React, Vite, Node 22); built into src/pitstop/server/static/
examples/financial-research/  the research-swarm example (package `finresearch`, CLI `finresearch`), a uv workspace member
  finresearch/agents/       research worker agents and the report synthesizer
  finresearch/supervisor/   drift supervisor: signals, LLM verdict + evidence validation, AlertPolicy, corrections
  finresearch/research/     You.com Finance Research client with an on-disk cache
  finresearch/demo/         the GEPA comparison demo server, its static page and the bundled snapshot
  finresearch/ui/           a prototype local run viewer (python -m finresearch.ui.server)
  finresearch/assets/       prompts (baseline, GEPA, worker) and scenarios
benchmarks/
  hard_pairs/         the labeled hard-pairs eval (18 categories, drift + healthy twins) and its runner
  calibration/        GEPA calibration of the detector prompt and the recursive loop with its gate
  held_out/           builder for the harder held-out set (unseen companies, distractors)
  synth/              the synthetic drift-trace generator (real workers, real You.com evidence)
tests/                offline test suite: SDK, correct, benchmarks, the example (tests/examples), repo hygiene
docs/                 design specs and plans (docs/superpowers/), hackathon-era documents (docs/history/)
```

## Benchmarks

**Hard pairs, AMD test split.** Train is NVIDIA and val is Broadcom; GEPA never saw this split. Each arm has 36 cases × 5 trials = 180 judgments, and an alert fires on DRIFTING at confidence ≥ 0.75.

| Supervisor | Prompt | Accuracy (5-trial range) | Drift caught | False alarms (of 180) |
|---|---|---|---|---|
| Nano 30B | original | 0.817 (0.778–0.861) | 0.667 | 3 |
| Nano 30B | GEPA (`pitstop_nano_800`) | **0.878** (0.861–0.917) | 0.756 | **0** |
| Super 120B | original | 0.861 (0.833–0.917) | 0.789 | 6 |
| Super 120B | GEPA (`pitstop_super_600`) | **0.956** (0.917–0.972) | 0.933 | 2 |

The small model with a rewritten prompt roughly matches the 4× larger model with the original prompt (0.878 vs 0.861; the trial ranges overlap). Nano's GEPA run took 29 minutes and cost about $0.70.

**Limitations.** Read these before relying on the numbers.

- **Small test set.** The test split is 36 cases. Per-case sign tests are not significant (Nano p = 0.69, Super p = 0.125), even though the gains are consistent across trials.
- **Weak transfer.** The test split shares its 18 categories with train and differs only by company. On the harder held-out set (unseen companies and close-call distractors), Nano fails with either prompt, flagging 75–81% of healthy steps. On the earlier team synthetic dataset, GEPA mostly shifted which drift types the supervisor favoured, so it was not promoted.
- **Longer prompts.** The GEPA prompts grew about 5× (Nano: 1.6k to 7.8k characters), and Nano checks got about 30–40% slower.
- **The gate rejected the recursive loop's first round.** Round 0 raised test accuracy (0.844 → 0.867) but also false alarms (1 → 11), so the current prompt was kept.
- **Synthetic data.** The hard-pairs trajectories are synthetic and their figures are illustrative. One rule in the Nano GEPA prompt still quotes an NVIDIA train figure.

**Reproduce** (requires `CRUSOE_API_KEY`; Crusoe inference only, no You.com):

```sh
uv run python -m benchmarks.hard_pairs.run_hard_cases --splits test --trials 5 \
    --model nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B \
    --prompt examples/financial-research/finresearch/assets/prompts/gepa/pitstop_nano_800.txt
```

Results go to `~/.pitstop/runs/eval-<prompt>-<timestamp>/`. The baseline result files are in [benchmarks/hard_pairs/results](benchmarks/hard_pairs/results). The raw hackathon traces and generated datasets are not committed; they are archived as a GitHub release asset (`pitstop-hackathon-data-2026-10-20.tar.gz`). The full hackathon write-up is in [docs/history/PITSTOP_RESULTS.md](docs/history/PITSTOP_RESULTS.md).

## Roadmap

From the [design spec](docs/superpowers/specs/2026-10-20-pitstop-runtime-design.md), §10. Development is paused after milestone 3a (November 2026): nothing past 3a is built, and the 3b plans are in [docs/superpowers/plans/](docs/superpowers/plans/).

1. **Cleanup and restructure. Done.** One installable package, the example agent still runs, tests pass in one command.
2. **SDK and detection. Done.** M2a: `wrap`, `workflow`, `agent`, `tool`, `prompt`, the hard-stop budget ledger, L0 detection with P0-P3 urgency, steer/block/halt actions, and the `pitstop status`, `pitstop drifts` and `pitstop budget` commands. M2b: hardened config and store (schema v4, migrations, `pitstop budget reconcile`), `wrap` for the OpenAI and Anthropic SDKs, the L1 pre-action gate, fallback to a pinned model, and P0/P1 alerts (PagerDuty, Opsgenie, Slack, email, webhook). M2c: refusing runs on a used-up budget and a P0 spend-spike halt, an OpenTelemetry span processor with verdicts as span events, OpenAI Agents SDK and LangChain/LangGraph adapters, L2 change-point statistics, L5 attribution (`pitstop explain`), the background `pitstop scan` pass, and the research swarm moved to `examples/financial-research/`.
3. **Dashboard. 3a done; 3b and 3c paused.** 3a, done: `pitstop ui` / `pitstop serve` (a local server with a background scan worker, `pitstop[ui]`), the pit wall with a live feed, workflow page (run graph, what changed, runs, open drifts with Explain and Dismiss), run detail and read-only settings, call latency and agent parentage in the SDK, and an offline demo. 3b, planned ([plan](docs/superpowers/plans/2026-10-23-pitstop-m3b-watch-layers.md), [carry fixes](docs/superpowers/plans/2026-10-23-pitstop-m3b-carry-fixes.md)): the L3 run judge, L4 endpoint canaries, the watch budget and the Endpoints page. 3c: the landing site, README screenshots and a GIF.
4. **Correction. Planned.** Managed prompts, replay, the GEPA loop with its gate, manual and auto modes, canary and rollback.
5. **Later. Not designed yet.** Remote collection over HTTP, more framework adapters, editing settings from the UI, a LiteLLM gateway plugin.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup, tests and pull requests. The test suite runs offline: `uv run pytest`.

## License

[Apache-2.0](LICENSE).

## Acknowledgements

PitStop started at the Crusoe x Cerebral Valley Recursive Agents Hackathon (October 2026, San Francisco), built on Crusoe Managed Inference by Qamil Mirza, Gautam Bhooma and Arjun Mittal. Prompt optimization uses [GEPA](https://github.com/gepa-ai/gepa) (the `gepa` library). Research agents use the You.com Finance Research API.

The post-hackathon redesign was built extensively with [Claude Code](https://claude.com/claude-code). That includes the research, the design spec, the restructure, the README and the tests.
