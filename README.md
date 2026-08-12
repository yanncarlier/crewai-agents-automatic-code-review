# CrewAI Agents — Automatic Code Review

An automated pull-request code review system built with [CrewAI](https://crewai.com). A crew of three specialized AI agents — a **Senior Developer**, a **Security Engineer**, and a **Tech Lead** — analyzes a set of code changes and produces a final review decision: approve, request changes, or escalate to a human.

The workflow is **configuration-driven**: agent roles/goals and task instructions are defined in plain Markdown files, so the entire review pipeline can be tuned without touching Python code.

## How it works

```
                    ┌─────────────────────────────┐
                    │  code_changes.txt (PR diff) │
                    └──────────────┬──────────────┘
                                   ▼
┌─────────────────────────┐  ┌──────────────────────────┐
│   Senior Developer      │  │   Security Engineer      │
│   Analyze Code Quality  │  │   Review Security        │
│   bugs / style /        │  │   vulnerabilities + risk │
│   maintainability       │  │   (SerperDevTool +       │
│   (critical vs minor)   │  │    ScrapeWebsiteTool,    │
└───────────┬─────────────┘  │    OWASP guidance)       │
            │                └───────────┬──────────────┘
            └───────────────┬────────────┘
                            ▼
                 ┌──────────────────────┐
                 │      Tech Lead       │
                 │  Make Review Decision│
                 │  approve / changes / │
                 │  escalate            │
                 └──────────┬───────────┘
                            ▼
                 Final report (printed) +
                 results.dill (serialized)
```

## Repository layout

| Path | Purpose |
|---|---|
| `agents_automatic_code_review_modular.py` | Main entry point: loads configs, builds agents/tasks, kicks off the crew |
| `agent_definitions/` | Markdown prompts for the three agents (`senior_developer.md`, `security_engineer.md`, `tech_lead.md`) |
| `task_definitions/` | Markdown prompts for the three tasks (`analyze_code_quality.md`, `review_security.md`, `make_review_decision.md`) |
| `code_changes.txt` | Input: the code changes (PR diff) to review |
| `results.dill` | Output: serialized CrewAI run result |
| `utils.py` | Helpers for loading `.env` and API keys |
| `patch.py` | Disables SSL verification for constrained/proxy environments (lab use) |
| `unittests.py` | Unit tests validating agents, tasks, tools, and crew wiring |
| `example.env` | Template for required environment variables |

## The agents

- **Senior Developer** — Reviews the diff for bugs, style, and maintainability issues, classifying each as `critical_issues` (must fix) or `minor_issues` (suggested). No tools.
- **Security Engineer** — Scans for security vulnerabilities, classifies them by risk (Critical/High/Medium/Low), and determines whether they block approval. Uses `SerperDevTool` (targeting owasp.org) to find security best practices and `ScrapeWebsiteTool` to read them.
- **Tech Lead** — Consumes both reviews and issues the final decision: approve, request changes, or escalate for human review.

## The tasks

| Task | Agent | Expected output |
|---|---|---|
| Analyze Code Quality | Senior Developer | JSON: `critical_issues`, `minor_issues`, `reasoning` |
| Review Security | Security Engineer | JSON: `security_vulnerabilities`, `blocking`, `highest_risk`, `security_recommendations` |
| Make Review Decision | Tech Lead | Report: final decision, required changes, escalation reasoning |

## Setup

1. Install dependencies:

   ```bash
   pip install crewai crewai-tools python-dotenv dill requests
   ```

2. Create a `.env` file (see `example.env`) with at least:

   ```bash
   OPENAI_API_KEY=your_api_key
   OPENAI_API_BASE=http://localhost:8080/v1   # or your OpenAI-compatible endpoint
   SERPER_API_KEY=your_serper_api_key
   DLAI_SERPER_BASE_URL=https://google.serper.dev
   ```

3. Put the code changes to review in `code_changes.txt`.

## Run

```bash
python agents_automatic_code_review_modular.py
```

The script will:
1. Load agent/task definitions from the Markdown files.
2. Read `code_changes.txt` and inject it into each task description via the `{code_changes}` placeholder.
3. Run the crew and print the Tech Lead's final review report.
4. Save the full serialized result to `results.dill`.

## Run tests

```bash
python unittests.py
```

## Notes

- `patch.py` globally disables SSL verification and patches the OpenAI/httpx clients — intended for lab/proxy environments only, not production.
- The workflow assumes an OpenAI-compatible LLM endpoint (see `OPENAI_API_BASE`).
- Outputs are in JSON format for the analysis tasks, making them easy to parse or feed into downstream tooling.
