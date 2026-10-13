# Autonomous AI Worker

A small, readable agent that takes a **natural-language task**, asks an LLM to **plan** it as structured JSON,
**executes** the plan in a real browser with Playwright, **observes** every result, **recovers** from failures,
and finally **verifies** that the goal was actually achieved by looking at the final page, not just by checking that
the steps ran.

It operates on a local mock internal system (`mock_app.html`, a support-desk dashboard), so everything runs on one
machine with no external services other than the LLM API.

---

## 1. Quick start

Requirements: Python 3.10+, and an API key for Anthropic **or** OpenAI.

```bash
cd centralign-agent-poc

python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate

pip install -r requirements.txt
playwright install chromium          # one-time browser download

# pick ONE provider
export ANTHROPIC_API_KEY=sk-ant-...  # Windows (PowerShell): $env:ANTHROPIC_API_KEY="sk-ant-..."
# export OPENAI_API_KEY=sk-...
# (or put the key in a .env file next to main.py)

python main.py
```

With no arguments `main.py` runs a demo task. Runs are headless by default; add `--headed` to watch.

### Things to try

```bash
# Write task: fill a form, submit, confirm the result
python main.py "Create a high priority bug ticket titled 'Printer on 3rd floor is offline' for alice@example.com"

# Failure injection: a blocking maintenance dialog + a transient 503 on the first submit
python main.py --chaos --headed --slow-mo 300 "Create a medium priority billing ticket titled 'Refund not received' for bob@example.com"

# Multi-step: act on existing data
python main.py "Resolve every open ticket that is high priority, then tell me how many tickets are still open"

# Read-only question: the answer is returned in the report
python main.py "How many resolved tickets are there, and who is the customer on TKT-1004?"
```

Every run writes `artifacts/run-<timestamp>/` containing `trace.json` (plan, every attempt, errors, verdict) and a
screenshot per attempt, which is the audit trail.

---

## 2. Architecture

```
          natural-language goal
                  |
                  v
   +------------------------------+
   | UNDERSTAND                   |  open start URL, snapshot the page
   +------------------------------+  (visible text, interactive elements + selectors, blocked elements)
                  |
                  v
   +------------------------------+
   | PLAN            (LLM call)   |  goal + snapshot -> JSON plan, validated with pydantic
   +------------------------------+  (invalid JSON / bad step -> error fed back, model retries)
                  |
                  v
   +------------------------------+      step ok         +-----------+
   | EXECUTE -> OBSERVE           | -------------------> | next step |
   | run step, check expect_text, |                      +-----------+
   | screenshot, log, record      |
   +------------------------------+
        | step failed
        v
   retry same step?  (transient only: timeouts, missing expected text)
        | still failing
        v
   +------------------------------+
   | RECOVER         (LLM call)   |  goal + history + error + FRESH page snapshot
   +------------------------------+  -> replacement steps (or give_up)
                  |
          all steps done
                  v
   +------------------------------+
   | VERIFY          (LLM call)   |  final page + extracted data -> verdict + answer
   +------------------------------+  evidence must be a verbatim quote of what was observed
        | not achieved  -> RECOVER (corrective replan)
        | achieved      -> report
```

| File | Responsibility |
|------|----------------|
| `mock_app.html` | The "internal system": ticket form with validation, dashboard counters, filterable table, resolve buttons, activity log. Has `data-testid`/`id` hooks and an optional `?chaos=1` failure mode. |
| `tools.py` | Playwright wrapper. A fixed action vocabulary, a uniform `ToolResult` (failures are data, never exceptions), page snapshots, screenshots, navigation allow-list. |
| `agent.py` | Pydantic models for the plan contract, `LLMClient` (Anthropic/OpenAI), the three prompts, and the loop. |
| `main.py` | CLI entry point and final report. |
| `smoke_test.py` | Optional offline test of the loop with a scripted LLM (no API key). |

### Why it is built this way

* **Plan, then act, with a feedback loop.** The plan is made once from a snapshot of the page, which is cheap and
  auditable. Surprises (a dialog, a transient error, a missing step) go through the recovery path, which re-plans
  from the *live* page.
* **Structured output is a contract.** Every LLM reply is parsed and validated (`Plan`, `Recovery`, `Verdict`). A
  step with a missing selector, or an action outside the vocabulary, is rejected and the validation error is sent
  back to the model for another try. Nothing unvalidated reaches the browser.
* **The model sees what a browser sees.** The snapshot lists each interactive element with a ready-to-use selector
  and flags elements that are physically covered by something else (`BLOCKED_BY=#announcement-modal`), so a blocking
  dialog is visible *before* the first click fails.
* **Two layers of error handling.**
  1. *Cheap retry* of the same step for transient problems (timeouts, or an `expect_text` post-condition that did not
     materialise, such as a "try again" toast). Deterministic failures such as a bad selector, a strict-mode
     violation or an overlay intercepting clicks skip the retry, because repeating them only burns a timeout.
  2. *LLM recovery*: the model gets the goal, the action history, the exact error and a fresh page snapshot, and
     returns replacement steps (or decides to give up).
* **Verification looks at outcomes.** "All steps ran" is not success. The verifier judges the final page state, and its
  evidence must be a short quote that really appears in what the browser returned, otherwise it is challenged once and
  then rejected. This guards against a model that confidently declares victory.
* **Hard limits.** Caps on replans (`--max-replans`) and total browser actions (40), per-action timeouts, and a
  navigation allow-list (the agent can only visit the mock app) keep a confused model from looping or wandering off.

---

## 3. Configuration

| Setting | How | Default |
|---------|-----|---------|
| Provider | `--provider anthropic\|openai` or `LLM_PROVIDER` | auto-detected from whichever API key is set |
| Model | `--model NAME` or `LLM_MODEL` | `claude-sonnet-5-5` (Anthropic), `gpt-4o` (OpenAI). Model names change, so override if yours differs |
| Temperature | `LLM_TEMPERATURE` | not sent (provider default) |
| Failure injection | `--chaos` | off |
| Browser window | `--headed`, `--slow-mo MS` | headless |
| Action timeout | `--timeout-ms` | 3000 |
| Recovery budget | `--max-replans` | 3 |
| Output folder | `--artifacts DIR` | `./artifacts` |

Exit code is `0` when the verifier confirmed the goal, `1` otherwise, and `2` for setup errors such as a missing API
key, so it is easy to script.

---

## 4. Example console trace (chaos mode)

The plan below tries to submit while the maintenance dialog is still open. The trace shows the recovery path:

```
UNDERSTAND  goal: Create a high priority bug ticket ...
PLAN        Create the ticket by filling the form and submitting it
            1. type #ticket-title 'Printer on 3rd floor is offline'
            ...
            5. click [data-testid="submit-ticket"]
EXECUTE     #05 FAIL click [data-testid="submit-ticket"]  (try 1/2)
            -> <div ... id="announcement-modal" ...> intercepts pointer events
RECOVER     (1/3) A dialog blocks the page; close it, then submit
            1. click #dismiss-announcement
            2. click [data-testid="submit-ticket"]
EXECUTE     #06 OK   click #dismiss-announcement  (try 1/2)
EXECUTE     #07 FAIL click [data-testid="submit-ticket"]  (try 1/2)
            -> The action ran, but the expected text 'created successfully' never appeared on the page.
EXECUTE     #08 OK   click [data-testid="submit-ticket"]  (try 2/2)
VERIFY      success=True | The new ticket appears in the table.
```

Exact wording varies by model. The structure (blocked click, recovery, transient 503, retry, verify) is what the chaos
mode is designed to exercise.

---

## 5. Testing

`python smoke_test.py` drives the real browser against the mock app with a *scripted* stand-in for the LLM, so it needs
no API key. It checks: the happy path, the evidence-grounding challenge, recovery from a blocking dialog, the retry of a
transient 503 via `expect_text`, verifier-triggered corrective replans, a read-only question, replan budgets, `give_up`,
and plan schema validation.

It tests the machinery around the model. How well a given model plans and recovers should be judged by running
`main.py` with a real key.

---

## 6. Limitations (known and deliberate for a PoC)

* **One-shot planning from one snapshot.** A plan can't reference elements that don't exist yet, such as the row of a
  ticket it is about to create; those are handled by verification and recovery. A per-step observe-and-decide
  (ReAct-style) loop would be more adaptive, at the cost of more LLM calls.
* **Retries are not idempotency-aware.** Re-running a failed `click` is safe here (the mock app keeps form state on
  failure), but retrying a non-idempotent action against a real system could double-submit.
* **Text-only perception.** The model sees extracted text and selectors, not pixels. Screenshots are saved for
  humans, but are not sent to the model, so canvas-heavy or visually encoded UIs are out of reach.
* **Single tab, single origin, no auth.** No login, MFA, file uploads/downloads, iframes, shadow DOM or multiple tabs.
* **Selector brittleness.** It works well because the mock app exposes stable `data-testid`s. Real apps with generated
  class names will need role/label-based locators or a richer accessibility snapshot.
* **Verification is LLM-based.** Grounding reduces false positives but cannot prove correctness. A critical
  workflow should also assert against the system of record (an API or database), not just the UI.
* **No secrets handling.** Typed values appear in logs and `trace.json`. Don't point this at real credentials.
* **No cost or latency tracking, no concurrency, no persistence between runs.**

## 7. Future improvements

1. **Adaptive loop:** re-observe after each step and let the model pick the next action, with the up-front plan kept
   as a guide, for stability on dynamic pages.
2. **Accessibility-tree perception** (roles, names, states) instead of CSS-ish selectors, plus optional screenshots
   for vision-capable models.
3. **Risk-aware actions:** tag steps as read / reversible / irreversible, require human approval for
   irreversible ones (payments, deletes), and add dry-run mode.
4. **Deterministic verifiers:** pluggable, task-specific checks (an API query, a DB assertion) that run alongside the
   LLM verdict.
5. **Observability:** structured traces (OpenTelemetry), token/cost accounting, and a replay UI for `trace.json`.
6. **Evaluation harness:** a benchmark of tasks with expected end states and chaos variants, run across models for
   success rate, steps, retries, and cost.
7. **Memory and reuse:** cache successful plans per app and task type ("skills"), and fall back to the LLM only on
   drift.
8. **Hardening:** secrets vault, per-run network allow-lists, sandboxed browser profiles, parallel workers, queueing,
   and timeouts per task.
