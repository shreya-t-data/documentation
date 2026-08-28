# DE Career Copilot — Agentic RAG + MCP on Databricks

## Problem
Job searching generates a lot of scattered evidence — a resume, several project READMEs, case studies, design decisions — that's hard to search or reason over in the moment, e.g. mid-interview-prep, when tailoring a resume bullet, or when trying to remember which project actually demonstrates a specific skill a JD is asking for. This project builds a personal RAG system over exactly that evidence, exposed as an MCP server so Claude Desktop can query and act on it directly, rather than building a standalone chatbot.

## Architecture
A local corpus (resume, project READMEs, case studies, design docs) is chunked and embedded locally using an open-source model, indexed in Databricks Vector Search, and queried by a local Python MCP server. That server exposes four tools — semantic search, full case-study retrieval, deterministic skills-gap checking against a pasted JD, and read/write access to a Lakebase-backed job-application tracker — to Claude Desktop as an MCP client. A Lakeflow Job automates re-chunking and re-embedding whenever the corpus changes.

## Tech Stack
Databricks Free Edition (Unity Catalog, Delta Lake, Vector Search, Lakeflow Jobs, Lakebase) · sentence-transformers (local embeddings) · Python `mcp` SDK, packaged as a Desktop Extension · Claude Desktop as MCP client

## Key Decisions
1. **Local embeddings over pay-per-token APIs** — kept the entire project's cost at genuinely $0, consistent with the constraint driving every tool choice in this build.
2. **Deterministic tools for factual checks, Claude for judgment** — `get_skills_gap` does plain keyword matching; observed in testing that Claude layered its own reasoning on top, correctly identifying JD requirements the fixed list didn't cover, confirming the intended division of labor.
3. **Fresh Postgres connections per call** — avoids Lakebase's 24-hour idle connection timeout affecting a server that may sit unused between sessions.
4. **Environment-panel dependencies over inline pip installs** — discovered mid-build that `%pip install` + `restartPython()` raced against automated "Run All"/Job execution, causing intermittent `ModuleNotFoundError`s; switching to Databricks' Environment side panel removed the race condition entirely and is the platform's documented pattern for reproducible execution.

## Results
A working agentic RAG system: real multi-tool chaining observed live in Claude Desktop (skills-gap check → automatic application logging, unprompted), a self-refreshing corpus via Lakeflow, and passing CI on every push.

## What I Learned
Real production-relevant debugging, not just following steps: a Vector Search index silently collapsing 11 rows into 2 due to a non-unique primary key, caught by noticing `indexed_row_count` didn't match the source table rather than any error message; a copy-pasted browser URL in an environment variable causing every API call to silently redirect to a login page instead of failing with a clear auth error; and a genuine platform migration — Claude Desktop's local MCP installation method moving from hand-edited JSON config to packaged Desktop Extensions — discovered and adapted to mid-build rather than assumed from outdated documentation.

## GitHub
https://github.com/shreya-t-data/de-career-copilot
