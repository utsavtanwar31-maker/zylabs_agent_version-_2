# Autonomous Company Research Agent

## Overview

This project implements an autonomous agent designed to research a company based on its name and optionally its website URL. The agent dynamically navigates the company's website, extracts offerings, target audiences, and case studies, and compiles a comprehensive Markdown profile.

## Architecture

The agent is built using **LangGraph** and **LangChain**.
It follows a state-machine architecture with the following nodes:
1. **init_research**: Initializes the research state, discovers the company website using Tavily search (if not provided).
2. **research_page**: Fetches a page's content, converts it to Markdown, and uses an LLM with structured output (Pydantic) to extract products, target audience, case studies, and new interesting links to explore.
3. **generate_profile**: Synthesizes all gathered evidence and citations into a final Markdown report.

**Agent Autonomy:**
The agent is not a static scraper. At each `research_page` node, the LLM analyzes the page content and extracts new URLs that look relevant (e.g., links to products, case studies, industries). The agent dynamically updates its `urls_to_visit` queue. It prioritizes URLs based on keywords (like 'product', 'customer', etc.) and determines when it has sufficient information by setting an `is_sufficient` flag, allowing it to halt early and proceed to `generate_profile`.

## Setup Instructions

1. Ensure you have Python 3.9+ installed.
2. Create and activate a virtual environment (optional but recommended).
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Configure API Keys:
   Create a `.env` file in the root directory and add your keys:
   ```env
   TAVILY_API_KEY=your_tavily_api_key_here
   # LLM provider: "huggingface" (default) or "openrouter"
   LLM_PROVIDER=huggingface
   HF_TOKEN=your_hugging_face_token_here          # needed for huggingface
   OPENROUTER_API_KEY=your_openrouter_api_key_here # needed for openrouter
   # Optional: override the provider's default model
   # (huggingface: nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B-BF16, openrouter: nvidia/nemotron-3-super-120b-a12b:free)
   LLM_MODEL=
   # Optional: max output tokens per call (default: 4000)
   LLM_MAX_TOKENS=4000
   # Optional token-budget controls
   PAGE_CHAR_LIMIT=8000   # characters of each fetched page shown to the LLM
   AGENT_MAX_STEPS=30     # agent step budget; the profile is written from saved evidence when reached
   ```
   Both providers expose an OpenAI-compatible API, so `langchain-openai`'s `ChatOpenAI` is pointed at [Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers) (`https://router.huggingface.co/v1`) or [OpenRouter](https://openrouter.ai) (`https://openrouter.ai/api/v1`). Whichever model you pick must support tool calling.

## Execution Instructions

Run the agent via the command line:

```bash
# With only company name (will search for website)
python agent.py "Stripe"

# With company name and website
python agent.py "Databricks" "https://databricks.com"
```

## Tools Available to the Agent

- **Tavily Search API**: Used in the `init_research` node to discover the company's official website if only the company name is provided.
- **HTTP/BeautifulSoup Scraper**: Used to fetch page content and convert it into markdown for the LLM to process.
- **LLM Structured Extraction**: The agent uses NVIDIA Nemotron via Hugging Face or OpenRouter with Pydantic structured outputs to extract structured data (offerings, target audiences, case studies) and intelligently identify new URLs to explore.

## Deliverables Included

1. **Source Code**: `agent.py` contains the LangGraph agent logic.
2. **Setup Instructions**: Provided above in this README.
3. **Tools & Autonomy Explanation**: Detailed in the Architecture & Tools sections.
4. **Example Runs & Profiles**: `stripe_profile.md`, `databricks_profile.md`, and `zapier_profile.md` represent 3 different test runs.
5. **Logs/Traces**: The `logs/` directory contains the CLI output traces showing the agent's decisions and page visits for each of the 3 runs.

## Known Limitations and Improvements

- **JavaScript Rendering**: The current tool uses `requests` and `BeautifulSoup`. It does not execute JavaScript. Using a tool like Playwright or Firecrawl would improve compatibility with heavily dynamic SPAs.
- **Context Limits**: Pages are currently truncated at 25,000 characters to avoid exceeding context windows.
- **Deeper Crawling**: The URL prioritization is heuristic-based. An LLM could be used to score and select the best next URL to visit for more intelligent navigation.
- **Checkpointing**: Adding LangGraph's `checkpointer` would allow the agent to pause, resume, and maintain state across sessions.
