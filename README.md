# CentrAlign Autonomous AI Worker (proof of concept)

A small, readable agent that takes a **natural-language task**, asks an LLM to **plan** it as structured JSON,
**executes** the plan in a real browser with Playwright, **observes** every result, **recovers** from failures,
and finally **verifies** that the goal was actually achieved by looking at the final page, not just by checking that
the steps ran.

It operates on a local mock internal system (`mock_app.html`, a support-desk dashboard), so everything runs on one
machine with no external services other than the LLM API.

---

## 1. Quick start

Requirements: Python 3.10+, and an API key for Anthropic, OpenAI, **or** Groq (via OpenAI-compatible endpoint).

```bash
cd centralign-agent-poc

python -m venv .venv
source .venv/bin/activate         # Windows: .venv\Scripts\activate

pip install -r requirements.txt
playwright install chromium        # one-time browser download

# Set your API provider and keys (Example using Groq)
$env:OPENAI_API_KEY="gsk_..."
$env:OPENAI_BASE_URL="[https://api.groq.com/openai/v1](https://api.groq.com/openai/v1)"

# Write task: fill a form, submit, confirm the result (using Groq / OpenAI provider)
python main.py --provider openai --model openai/gpt-oss-20b "Create a high priority bug ticket titled 'Printer on 3rd floor is offline' for alice@example.com"

# Failure injection: a blocking maintenance dialog + a transient 503 on the first submit
python main.py --provider openai --model openai/gpt-oss-20b --chaos --headed --slow-mo 300 "Create a medium priority billing ticket titled 'Refund not received' for bob@example.com"

# Multi-step: act on existing data
python main.py --provider openai --model openai/gpt-oss-20b "Resolve every open ticket that is high priority, then tell me how many tickets are still open"

# Read-only question: the answer is returned in the report
python main.py --provider openai --model openai/gpt-oss-20b "How many resolved tickets are there, and who is the customer on TKT-1004?"


        natural-language goal
                   |
                   v
   +------------------------------+
   | UNDERSTAND                   |  open start URL, snapshot the page
   +------------------------------+  (visible text, interactive elements + selectors, blocked elements)
                   |
                   v
   +------------------------------+
   | PLAN          (LLM call)     |  goal + snapshot -> JSON plan, validated 
   +------------------------------+  with pydantic
                   |
                   v
   +------------------------------+       step ok        +-----------+
   | EXECUTE -> OBSERVE           | -------------------> | next step |
   | run step, check expect_text, |                      +-----------+
   | screenshot, log, record      |
   +------------------------------+
        | step failed
        v
     retry same step?  
        | still failing
        v
   +------------------------------+
   | RECOVER       (LLM call)     |  goal + history + error + FRESH page snapshot
   +------------------------------+  -> replacement steps 
                   |
          all steps done
                   v
   +------------------------------+
   | VERIFY        (LLM call)     |  final page + extracted data->verdict+ answer
   +------------------------------+  evidence must be a verbatim quote of what was observed
        | not achieved  -> RECOVER 
        | achieved      -> report




---

##  Technical & Design Decisions

* **Structured Output Contract via Pydantic:** Instead of sending LLM responses directly to the browser, we use Pydantic models (`Plan`, `Recovery`, `Verdict`) to validate them first. This ensures that invalid JSON or unsupported actions never reach the browser.
* **Stateful Error Handling & Recovery:** If a step fails (such as a blocking popup or a temporary network glitch), the agent doesn't just blindly repeat old steps. Instead, it takes a fresh snapshot of the live page and asks the LLM for a corrected recovery plan.
* **Outcome-Based Verification:** Simply checking whether "all steps run" does not mean the task was successful. The verifier checks the final page state and extracted data to confirm if the user's goal was truly achieved.

---

##  Assumptions Made

* **Mock Environment:** The system is designed specifically for a local sandbox environment (`mock_app.html`), so it doesn't handle complex multi-tab navigation, iframes, or heavy authentication/login flows like real enterprise apps.
* **API Availability:** It assumes that the user has a valid OpenAI-compatible API key (such as Groq) correctly configured in their local environment variables.

---

##  Tech Stack & Frameworks Used

* **Language:** Python 3.10+
* **Browser Automation:** Playwright (Chromium)
* **Data Validation:** Pydantic
* **LLM / APIs:** Groq API (`openai/gpt-oss-20b` via OpenAI-compatible client)
* **Environment:** Windows PowerShell, VS Code

##  Known Limitations

* **One-shot planning from one snapshot:** A plan cannot reference elements that do not exist yet (like a ticket row it is about to create); those are handled by recovery and verification.
* **Text-only perception:** The model sees extracted text and selectors, not pixels or complex canvas-heavy elements.
* **Single tab & origin:** No support for multiple tabs, iframes, or heavy authentication/MFA workflows.
* **Selector Brittleness:** Relies on clean data attributes (`data-testid`); real-world dynamic websites would require more flexible locators.

---

##  Future Improvements 

* **Adaptive Loop:** Re-observe after each step and let the model pick actions dynamically for higher stability on complex pages.
* **Accessibility-Tree Perception:** Use roles, names, and states instead of strict CSS selectors.
* **Risk-Aware Actions:** Tag actions as read/write/irreversible and add human approval checkpoints for sensitive tasks (e.g., payments or deletions).
* **Deterministic Verifiers:** Add programmatic/API assertions alongside the LLM verdict to guarantee 100% data correctness.
