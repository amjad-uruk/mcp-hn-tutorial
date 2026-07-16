# Build Your MCP Interactive Tutorial

A self-guided, hands-on tutorial for learning the **Model Context Protocol (MCP)** by
building a real MCP server — one chapter at a time, with an LLM acting as your tutor.

## What you'll build

A Python MCP server that lets an LLM crawl the public **Hacker News API** and populate a
**local SQLite database** with:
- **Stories** — the "activities."
- **The discussion roster** — the author plus everyone who commented on each story
  (found by walking the comment tree).

The MCP server is the "hands" (tools that fetch, parse, and store); a separate LLM is the
"brain" that drives an incremental, resumable crawl. The whole dev environment runs inside
**Docker/Podman**, insulated from your host.

## How it works

This repo doesn't ship the server code — **you** write it, guided by an LLM tutor. The
tutorial is a single **prompt** you paste into a capable LLM (e.g. Claude). The prompt
makes the model behave as a strict, hands-on tutor: you write all the real code, and it
explains, assigns one micro-task at a time, and reviews — never one-shotting the project.

Grab the prompt here:
[`docs/mcp-hn-tutorial-plan.md`](docs/mcp-hn-tutorial-plan.md)

## Chapters (~7.5–8.5 hours total, one per sitting)

0. Orientation & MCP mental model (~30 min)
1. Containerized dev env — Docker/Podman (~60 min)
2. Hello MCP + Inspector (~45 min)
3. HTTP client for the HN API (~45 min)
4. Fetch & parse stories (~75 min)
5. Discussion roster — who's on each story (~45 min)
6. Local database (~60 min)
7. Crawlable "skill" — LLM-driven crawl (~75 min)
8. Harden & ship (~60 min)

## Stack

Python 3.12 · FastMCP (official MCP Python SDK) · httpx · pydantic · SQLite ·
Docker/Podman · MCP Inspector.

## Data source

Public **Hacker News Firebase API** (`https://hacker-news.firebaseio.com/v0/`) — no auth,
pure JSON — plus the no-auth **Algolia HN Search API** for full-text search.
