# context.md — Autonomous Company Research Agent

> Running log for the internship assignment. Updated after every work session.
> Last updated: 2026-09-24 (Session 3: Phase 0 setup)

---

## 1. Goal (one line)
Build an agent that takes a company name (+ optional website), decides for itself how to research it, and outputs a cited Markdown **Company Profile**: what they sell, who they sell to (industry / size / geography), and representative case studies.

## 2. What the evaluators actually grade (ranked by weight)
1. **Agent design.** It must be a real agent: plans, picks tools, reacts, changes strategy. A fixed scrape-then-summarize pipeline fails.
2. **Research quality.** Offerings and customers are correctly identified.
3. **Case study discovery.** It finds them even under "Resources", "Stories", "Customers", and similar.
4. **Evidence.** Every claim is traceable (URL + title + snippet). It says "unclear" instead of guessing.
5. **Robustness.** Handles JS sites, redirects, 404s, missing pages, tool failures.
6. **Efficiency.** No full crawl, no duplicate visits, controlled context.
7. **Code quality.** Clean, production-ish.

## 3. Required deliverables (checklist)
- [ ] Source code
- [ ] README (setup + run)
- [ ] Tool descriptions
- [ ] Explanation of how autonomy is implemented
- [ ] ≥3 example runs on differently structured sites
- [ ] Generated Markdown profiles for those runs
- [ ] Logs/traces of agent decisions + tool calls
- [ ] Known limitations + improvements

**Bonus targets** (pick the cheap, high-signal ones): checkpoint/resume, page cache, subagents in parallel, Pydantic-validated output, confidence levels, conflict detection, stopping criteria, token/cost tracking, retry/fallback between fetchers.

## 4. Platform decision
**Recommended: local Python repo built with Claude Code. Add an optional Colab notebook only as a "try it" demo.**

Reasons:
- The deliverable is a repo (code, README, logs, outputs), not a notebook.
- Playwright (needed for JS-heavy sites) is awkward in Colab because of browser installs and async event-loop clashes.
- Checkpointing, SQLite cache, and trace files persist locally; Colab wipes them when the session ends.
- Claude Code can run the agent against real websites and iterate. The chat sandbox has restricted internet, so real test runs must happen on your machine.

Status: ✅ **Confirmed.** Built with Claude Code. Utsav pastes prompts from `claude_code_prompts.md`; `CLAUDE.md` holds the spec.

## 5. Proposed architecture (draft)
- **Framework:** LangChain **Deep Agents** on LangGraph. It is named in the brief and provides planning (todo list), subagents, and a virtual file system for notes out of the box.
- **Orchestrator agent:** reads the goal, writes a research plan, dispatches subagents, checks coverage gaps, and decides when to stop.
- **Subagents** (isolated context, can run in parallel):
  - `offerings_researcher`: products, services, platforms, product families
  - `customer_researcher`: industries, company size, geography
  - `case_study_hunter`: finds and extracts customer stories
- **Tools:**
  - `web_search(query)`: Tavily / DuckDuckGo. Used for website discovery and `site:` searches.
  - `fetch_page(url)`: httpx + trafilatura first, Playwright fallback for JS pages. Returns a clean summary plus a list of links.
  - `get_sitemap(domain)`: robots.txt → sitemap.xml → filtered URL list.
  - `find_links(url, keywords)`: ranks the links on a page by relevance.
  - `save_evidence(claim, url, title, quote)`: writes to the evidence store and returns an evidence ID.
  - `research_status()`: shows visited URLs, filled vs. missing fields, and budget used.
- **State/infra:** SQLite page cache (dedup), LangGraph SqliteSaver (checkpoint/resume), JSONL trace log, token/cost counter.
- **Output:** Pydantic `CompanyProfile` in which every field references evidence IDs, rendered to Markdown. A validator rejects any claim without evidence.
- **Stopping rule:** stop when all required fields have evidence or are explicitly marked unverifiable, or when the page/token budget is exhausted.

## 6. Build sequence
Deadline: **2026-09-24 EOD IST**. Total estimate is about 9 h of Claude Code time.

| # | Phase (prompt) | Output | Est. | Status |
|---|-------|--------|------|--------|
| 0 | Setup + Deep Agents API check | skeleton repo, API cheat sheet | 20m | ✅ (live key check pending on Utsav's PC) |
| 1 | Tool layer: fetch (+Playwright fallback), search, sitemap, links, cache, budgets | `fetch.py`, `tools.py`, tests | 1.5h | ⏳ |
| 2 | Evidence store (quote verification) + Pydantic schema | `evidence.py`, `schema.py` | 45m | ⏳ |
| 3 | Agent: orchestrator + 3 subagents, tracing, checkpointer | `agent.py`, `prompts.py`, `tracing.py` | 2h | ⏳ |
| 4 | Finalize (evidence-only) + validator + Markdown + CLI + resume | `finalize.py`, `report.py`, `cli.py` | 1.5h | ⏳ |
| 5 | Runs: Freshworks, Temenos, Tiger Analytics, name-only Chargebee | `outputs/*` | 2h | ⏳ |
| 6 | README, autonomy explanation, limitations, final commit | submission | 1h | ⏳ |

## 6a. Deep Agents API cheat sheet (verified against installed deepagents 0.7.18, langgraph 1.2.12, langchain 1.4.2)
1. `from deepagents import create_deep_agent` → `create_deep_agent(model=None, tools=None, *, system_prompt, middleware=(), subagents=None, skills, memory, permissions, backend, interrupt_on, response_format, state_schema, context_schema, checkpointer, store, debug, name, cache)` → `CompiledStateGraph`.
2. `model` must be a `str` ("provider:model") or a `BaseChatModel`. **A `.with_fallbacks()` Runnable crashes it** (`resolve_model` calls `.partition` on it).
3. Fallback for agents = `langchain.agents.middleware.ModelFallbackMiddleware(fallback_model)` in `middleware=[...]`. Also available: `ModelRetryMiddleware`, `ModelCallLimitMiddleware`, `ToolCallLimitMiddleware`.
4. Custom tools: `tools=[...]` accepts `BaseTool`, plain typed functions with docstrings, or dicts.
5. Subagents: `subagents=[SubAgent]`, a TypedDict: `name`, `description` (required); `system_prompt`, `tools`, `model`, `middleware`, `response_format`, `mode` ("isolated" default | "fork"), `interrupt_on`, `skills`, `permissions` (optional).
6. Subagent `tools` omitted → inherits main tools. `model` omitted → inherits main model. **`middleware` is NOT inherited**, so add `ModelFallbackMiddleware` to each subagent spec.
7. The orchestrator calls subagents via the built-in `task(description, subagent_type=name)` tool; the subagent's last AIMessage (or `structured_response`) comes back as the ToolMessage.
8. A `general-purpose` subagent is auto-added unless you define one with that name or disable it via a `HarnessProfile(general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False))`.
9. Built-ins: `write_todos`, virtual FS (`ls`, `read_file`, `write_file`, `edit_file`, …) backed by state by default (`backend=`).
10. Checkpointer: `from langgraph.checkpoint.sqlite import SqliteSaver` (pkg `langgraph-checkpoint-sqlite`); `with SqliteSaver.from_conn_string("x.sqlite") as cp: create_deep_agent(..., checkpointer=cp)`; invoke with `config={"configurable": {"thread_id": run_id}}`.

## 7. Open questions
- Confirm the Gemini free-tier requests-per-minute limit on Utsav's key. It sets how long each run takes. `LLM_RPM=8` is the default for now.

## 7a. Issues / problems
- `docs/assignment.md` is missing from the repo folder. Needs to be added.
- Phase 0 ran in Claude's cloud workspace, which is **blocked from Gemini/Groq/Cerebras/Tavily** and has no shell on Utsav's PC. So the live tool-call check and `playwright install chromium` haven't run yet. They run via `setup.ps1` on Utsav's machine.
- `init_chat_model` has no `cerebras` provider in langchain 1.4. `llm.build_model` constructs `ChatCerebras` directly for `cerebras:` specs.
- `.with_fallbacks()` is incompatible with `create_deep_agent` (see cheat sheet #2). So `llm.py` exposes both `get_llm()` (a fallback chain for direct calls such as finalize) and `get_models()` + `get_fallback_middleware()` for agents.

## 7b. agent_assignment/ Implementation Status
A preliminary/alternative implementation has been created in the `agent_assignment/` directory. It uses a single-agent LangGraph state machine with OpenAI `gpt-4o-mini` and `requests`.
- **What is done there:** Accepts name/URL inputs, extracts structured data via Pydantic (offerings, audiences, case studies), performs basic autonomous navigation via LLM-selected URLs, applies heuristic link scoring, limits crawling, and produces the required Markdown profiles (tested on Stripe, Databricks, Zapier).
- **What is missing there:** Does not handle JS-heavy sites (no Playwright/Firecrawl), lacks subagents (uses a monolithic extraction prompt), does not capture verbatim page titles/quotes for rigorous citations, and has no persistent state/checkpointer.

## 8. Decisions log
- 2026-09-24: Project started. Plan drafted.
- 2026-09-24: Switched to FREE models only. Primary: Gemini 2.5 Flash (Google AI Studio). Fallback: Groq or Cerebras `gpt-oss-120b`. Calls are rate-limited and fall back automatically on 429 errors.
- 2026-09-24: Stack locked: Claude Code; Tavily search; Deep Agents on LangGraph; SQLite for cache, evidence and checkpoints.
- 2026-09-24: Anti-hallucination approach: `save_evidence` only accepts quotes that match cached page text, and the final profile is built from the evidence store only.
- 2026-09-24: Fallback plan: if the Deep Agents API blocks progress, switch to the LangGraph ReAct agent with subagents exposed as tools (rescue prompt).
- 2026-09-24: Agent-level fallback uses `ModelFallbackMiddleware` (on the main agent and each subagent). `.with_fallbacks()` is only used for direct calls. SDK `max_retries=1` so a 429 hands over to the fallback fast. Rate limits are env-configurable (`LLM_RPM`, `LLM_FALLBACK_RPM`).
- 2026-09-24: Packaging: `pyproject.toml` (src layout, `pip install -e .[dev]`); added `rank-bm25` for `read_page_section`; `setup.ps1` for Windows setup and verification.

## 9. Session log
- **S1 (2026-09-24):** Read the brief, drafted the plan and architecture, created context.md.
- **S2 (2026-09-24):** Confirmed the stack and deadline. Wrote CLAUDE.md (spec) and claude_code_prompts.md (7 phase prompts + rescue prompts). Next: Utsav runs Prompt 0.
- **S3 (2026-09-24): Phase 0.** Scaffolded the repo (14 stub modules plus real `config.py` and `llm.py`), `pyproject.toml`, `.env.example`, `.gitignore`, git init. Installed the stack in a venv. Read the installed deepagents source and wrote the cheat sheet (§6a). Built `llm.py`. Offline checks: `pytest` shows **3 passed** (rate limiter attached to all 3 providers, `get_llm()` is a fallback chain, a simulated 429 on the primary returns the fallback's answer). A deep agent built with `ModelFallbackMiddleware`, a subagent and a checkpointer compiled OK. `scripts/verify_phase0.py` runs end to end, but all live calls fail with `403 proxy` (sandbox network block, not a code error). **Next:** Utsav fills in `.env` and runs `powershell -ExecutionPolicy Bypass -File setup.ps1`, then pastes the verify summary here. Then Phase 1.
- **S4 (2026-09-24):** Reviewed the `agent_assignment/` folder against the assignment. It contains a functional V1 baseline using a monolithic LangGraph approach, OpenAI, and simple HTTP requests. It meets basic requirements but lacks the robustness (Playwright), deep traceability (exact quotes/titles), and Deep Agents architecture outlined in the plan above. Created `evaluation_findings.txt` summarizing gaps and updated this context document.
