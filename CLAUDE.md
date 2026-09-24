# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This subproject sits inside the Forge (`projects_2026`) repo — the root `CLAUDE.md` and
`FORGE_GOVERNANCE.md` still apply (git staging/push approval, no `--no-verify`/`--force`
without approval, etc.). This file adds only what's specific to this project.

## Project overview

"Ice Breaker" — a Flask + LangChain app that generates personalized conversation starters
by looking up a person's LinkedIn (and optionally Twitter/X) profile, scraping their data,
and running it through LLM chains to produce a summary, interests, and ice breakers.

## Commands

Dependency management is Pipenv (Python 3.10, see `Pipfile`).

```bash
pipenv install              # install dependencies
pipenv shell                # activate the virtualenv
pipenv run python app.py    # run the Flask app (http://localhost:5000)
pipenv run pytest .         # run tests (no test files currently exist in the repo)
pipenv run pylint <path>    # lint (pylint is a listed dependency)
pipenv run black .          # format
pipenv run isort .          # sort imports
```

There is no build step — this is a plain Flask server rendering `templates/index.html`.

Running any module standalone (several have `if __name__ == "__main__":` blocks for manual
testing) requires the `.env` file to be populated first, e.g.:

```bash
pipenv run python third_parties/linkedin.py
pipenv run python third_parties/twitter.py
```

## Environment

Copy `.env.example` to `.env` and fill in real keys before running anything that hits a
live API. Required: `OPENAI_API_KEY`, `SCRAPIN_API_KEY` (LinkedIn scraping via scrapin.io —
note `.env.example` has a legacy `PROXYCURL_API_KEY` name, the code actually reads
`SCRAPIN_API_KEY`), `TAVILY_API_KEY` (profile-URL search). Twitter keys and
`LANGCHAIN_*` (LangSmith tracing) are optional — if `LANGCHAIN_TRACING_V2=true` is set,
`LANGCHAIN_API_KEY` must also be valid or the app throws.

## Architecture

Request flow, driven by `ice_break_with()` in `ice_breaker.py` (called from the
`/process` route in `app.py`):

1. **Agents locate profile URLs** (`agents/linkedin_lookup_agent.py`,
   `agents/twitter_lookup_agent.py`) — each spins up its own ReAct agent
   (`langchain.agents.create_react_agent` + `AgentExecutor`, pulling the prompt from
   `langchain.hub`) with a single Tavily search tool
   (`tools/tools.py::get_profile_url_tavily`) to find the person's profile URL/username
   from their name.
2. **Third-party scrapers fetch raw data** (`third_parties/linkedin.py`,
   `third_parties/twitter.py`) — `scrape_linkedin_profile()` calls the scrapin.io API (or a
   fixed mock Gist URL when `mock=True`); `scrape_user_tweets_mock()` is what
   `ice_breaker.py` actually calls (pulls a hardcoded Gist of sample tweets rather than
   live Twitter — swap in `scrape_user_tweets()` for real Twitter API calls).
3. **Chains transform data into structured output** (`chains/custom_chains.py`) — three
   independent `PromptTemplate | llm | PydanticOutputParser` pipelines (summary,
   interests, ice breakers), each invoked separately with the same
   `{information, twitter_posts}` inputs. Two LLM instances exist at module scope:
   `llm` (temperature 0, factual) and `llm_creative` (temperature 1, currently unused by
   any chain).
4. **Output schemas** (`output_parsers.py`) define the three Pydantic models
   (`Summary`, `IceBreaker`, `TopicOfInterest`), each paired with a
   `PydanticOutputParser` whose `format_instructions` get injected into the chain's
   prompt template, and each with a `to_dict()` used to build the Flask JSON response.
5. **Flask app** (`app.py`) exposes `/` (renders `templates/index.html`) and `/process`
   (POST, takes `name` form field, returns JSON with summary/interests/ice_breakers/photo
   URL).

Key coupling to know before changing code: swapping `scrape_user_tweets_mock` for
`scrape_user_tweets` in `ice_breaker.py` requires valid Twitter API credentials in `.env`;
the chains all expect `information` to be a dict-like LinkedIn profile and `twitter_posts`
to be a list of `{text, url}` dicts — changing either scraper's return shape requires
updating the prompt templates in `chains/custom_chains.py` accordingly.
