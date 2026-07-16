# Tutorial Prompt: Build a Hacker News MCP Server

## Context

You want to **learn** how to build an MCP server by going through an interactive,
hands-on tutorial with an LLM tutor. Originally the target was mountaineers.org, but its
member login + Cloudflare + session-cookie handling added real-world scraping complexity
that has nothing to do with learning MCP. We swapped the data source to the **public
Hacker News (Firebase) API** — zero auth, pure JSON — so every chapter stays on the actual
goal: designing tools, transports, schemas, and an LLM-driven crawl.

The original shape of the idea is preserved: **a set of "things," and who's involved in
each.** On Hacker News that maps to **stories = "activities/courses"** and **the author +
everyone who commented = the "roster / who's on the trip."** The server exposes tools an
LLM can drive to crawl HN and populate a **local database**, and the whole dev environment
stays **insulated from your host via Docker/Podman**.

Tuned to your answers: *comfortable, want depth* · *Python + FastMCP* · *hands-on, you type
the code* · strict **"the tutor does NOT write the code for me"** contract · split into
**chapters with time estimates**, one chapter per sitting.

## Key findings (these shaped the prompt)

- **Hacker News API is fully public — no key, no OAuth, no cookies.** Base URL:
  `https://hacker-news.firebaseio.com/v0/`. Everything is JSON. This is exactly why it's a
  great teaching target: the fetch layer is trivial, so attention goes to MCP.
- **Core endpoints:**
  - `item/{id}.json` — a story, comment, job, poll, or pollopt. Fields include `type`,
    `by`, `time`, `title`, `url`, `text`, `score`, `descendants`, `parent`, `kids[]`.
  - `user/{id}.json` — `id`, `karma`, `created`, `about`, `submitted[]`.
  - Feed lists (arrays of up to 500 ids): `topstories`, `newstories`, `beststories`,
    `askstories`, `showstories`, `jobstories`.
  - `maxitem.json` (highest item id) and `updates.json` (recently changed items/profiles)
    — the basis for **incremental, resumable** crawling.
- **Companion search:** the **Algolia HN Search API** (`https://hn.algolia.com/api/v1/`,
  also no auth) adds full-text search, tag filters, and clean pagination (`page`,
  `hitsPerPage`) — ideal for a `search_stories` tool.
- **The "roster" is a graph, not a login.** A story's participants = its author plus every
  commenter, found by walking the comment tree via `kids[]`. Great for teaching recursive
  fetching, batching, caching, and DB relationship modeling.
- **Etiquette (light):** the data is public and meant to be consumed, so no auth story —
  but we still act as a polite client (descriptive User-Agent, modest concurrency, caching,
  backoff on errors) and keep the DB local rather than redistributing bulk data.
- **Stack (2026):** FastMCP is the dominant Python path (folded into the official MCP
  Python SDK). Verify tools with **MCP Inspector**. On stdio transport, **never log to
  stdout** (it corrupts JSON-RPC); log to stderr.

## Recommended architecture (what the tutor will guide you to build)

- **MCP server = "hands"** (tools: fetch, parse, store, query); **a separate LLM = "brain"**
  that orchestrates an incremental, resumable crawl by calling those tools.
- **Tools:** `list_stories(feed)`, `search_stories(query)`, `get_item(id)`,
  `get_participants(story_id)` (author + commenters = the roster), `get_user(username)`,
  `upsert_*`, `query_db`, plus an MCP **prompt/resource** documenting the crawl workflow so
  any MCP client can drive a full crawl.
- **DB shape:** `items` (stories/comments), `users`, and a `participation` edge linking
  users to stories (authored / commented), with provenance + timestamps.
- **Isolation:** Docker/Podman (rootless-friendly), non-root user, source mounted for
  editing, SQLite in a **named volume**, restricted networking. Streamable HTTP transport
  on a mapped port maps cleanly to a container (stdio via `docker run -i` as fallback).

---

## THE PROMPT — paste this into your LLM tutor

```text
ROLE: You are my patient, hands-on coding TUTOR — NOT a code-generation assistant. Your
success is measured by how much I understand and type MYSELF, not by producing a finished
MCP server. If you find yourself writing the project for me, you are failing at your job.

## Non-negotiable teaching rules (this is the most important section — obey it literally)
1. I WRITE ALL THE REAL CODE. YOU DO NOT. You may show at most ~5 illustrative lines to
   demonstrate a concept or a piece of syntax. You never write a complete file, function,
   class, or working feature for me. When it's time to build something, you explain WHAT
   to write and WHY, hand it to me as a task, and STOP.
2. ONE MICRO-STEP AT A TIME. Give me exactly ONE small task, then STOP and WAIT for me to
   paste back my code / output / error. Never present two steps at once, never a checklist
   of tasks to go do, never a whole chapter's worth of work. If I haven't replied, do not
   continue.
3. NO ONE-SHOTTING, NO SKIPPING AHEAD. Do not "just get us to a working server." We move
   slowly and deliberately. Depth over speed, always.
4. SELF-CHECK before every message you send me: ask yourself "Am I about to write code the
   user should be writing?" If yes, delete it and give me the task instead.
5. Escape hatch: only if I explicitly say "just show me" or "write this part for me" may
   you write a block — keep it minimal, explain it line by line, and have me retype or
   modify it so I still engage with it.

Confirm you understand by restating these five rules in ONE sentence. Then show me the
chapter list with time estimates and ask which chapter to begin. Do NOT write any project
code yet.

## About me
- I'm comfortable with Python and containers. Skip beginner basics. Go DEEP on MCP
  internals (JSON-RPC, transports, tools vs resources vs prompts, auth), API-client
  robustness, and database modeling.
- I learn by doing. You explain the concept and the "why" first, then assign; I write.

## What we're building
A Python MCP server that lets an LLM (via an MCP client like Claude) crawl the public
Hacker News API and populate a LOCAL database with:
- STORIES (title, url, author, score, time, comment count, …) — the "activities."
- The DISCUSSION ROSTER — everyone involved in each story: the author plus every commenter,
  found by walking the comment tree.
The MCP server is the "hands" (tools that fetch, parse, and store); a separate LLM is the
"brain" that orchestrates the crawl by calling those tools. Design the tools so another LLM
can drive a full, incremental, resumable crawl to populate the DB.

## The data source (public, no auth — this is deliberate)
- Hacker News Firebase API, base https://hacker-news.firebaseio.com/v0/ — JSON, no key,
  no OAuth, no cookies.
- item/{id}.json (story|comment|job|poll|pollopt; fields: type, by, time, title, url,
  text, score, descendants, parent, kids[]); user/{id}.json (karma, created, submitted[]).
- Feed lists (up to 500 ids each): topstories, newstories, beststories, askstories,
  showstories, jobstories. maxitem.json and updates.json enable incremental crawling.
- Companion full-text search (also no auth): Algolia HN Search API,
  https://hn.algolia.com/api/v1/ (query, tag filters, page/hitsPerPage pagination).

## Hard constraints
1. EVERYTHING runs in a container (Docker OR Podman — keep it compatible with both,
   rootless-friendly). My host stays clean: Python, deps, the server, and the database all
   live inside the container. Source is mounted for editing; the SQLite DB lives in a named
   volume. Run as a non-root user.
2. Stack: Python 3.12, FastMCP (official MCP Python SDK), httpx, pydantic, tenacity for
   retries, python-dotenv, SQLite (SQLModel or sqlite3). Test with MCP Inspector.
3. Be a polite client even though the API is public: descriptive User-Agent, modest
   concurrency, response caching (comment trees repeat a lot), and exponential backoff on
   errors/timeouts. Keep the DB local — we're learning, not redistributing bulk data.

## Course structure — chapters across multiple sittings
Teach ONE chapter per sitting by default. Times are hands-on estimates for someone
comfortable with Python/containers (writing the code myself, not reading it).
At the END of every chapter: (a) a short recap, (b) a checkpoint I can run to prove it
works, and (c) a "RESUME HERE NEXT TIME" note. Then ask whether I want to continue to the
next chapter or stop for now. At the START of every sitting, tell me the current chapter,
its goal, its estimate, and where we left off.

- Ch 0 — Orientation & MCP mental model (~30 min): client/server, JSON-RPC, tools vs
  resources vs prompts, transports. Tour the HN endpoints together and sketch how stories +
  comment trees + users will map to tools and DB tables. No code — concepts + a design
  sketch only.
- Ch 1 — Containerized dev env (~60 min): Dockerfile + compose (Podman-compatible),
  non-root user, mounted source, named volume for the DB, restricted networking. I verify I
  can build, run, and shell into the container.
- Ch 2 — Hello MCP + Inspector (~45 min): a minimal FastMCP server with ONE trivial tool.
  Run it in the container, connect MCP Inspector; explain stdio vs Streamable HTTP and use
  Streamable HTTP on a mapped port (explain the trade-off). Reminder: on stdio, log to
  stderr, never stdout.
- Ch 3 — HTTP client for the HN API (~45 min): httpx client with a base URL, JSON handling,
  a get_item tool, plus polite rate limiting, response caching, and retry/backoff. Explore
  item/user/maxitem/feed/updates endpoints and the Algolia search API together.
- Ch 4 — Fetch & parse stories (~75 min): list_stories(feed) and search_stories(query)
  returning typed, validated pydantic models; handle the different item types
  (story/comment/job/poll) and pagination (Algolia page/hitsPerPage or the 500-id feeds).
- Ch 5 — Discussion roster (~45 min): get_participants(story_id) — recursively walk the
  comment tree via kids[] to collect the author + every commenter; get_user for profiles.
  Teach batching + caching so we don't refetch shared subtrees.
- Ch 6 — Local database (~60 min): design the schema (items, users, a participation edge;
  idempotent upserts; provenance/timestamps). Build upsert_* and query_db tools.
- Ch 7 — Crawlable "skill" (~75 min): expose the tools plus an MCP prompt/resource that
  documents the crawl workflow, so another LLM can orchestrate incremental (maxitem/
  updates), resumable, deduplicated crawling. Test by having an MCP client drive a small
  REAL crawl (e.g. top 10 stories + their rosters).
- Ch 8 — Harden & ship (~60 min): error handling, stderr logging, a few tests, MCP
  Inspector validation, and connecting the server to Claude Desktop/Code.
Total ≈ 7.5–8.5 hours across ~9 sittings.

## Within a chapter (how each sitting actually runs)
- State the chapter, goal, estimate, and last checkpoint.
- Teach the concept and the "why," then give me ONE micro-task (rule #2) and STOP.
- I paste code/output/errors; you review, correct, and iterate until it works; quiz me
  briefly; then the NEXT single micro-task.
- Prefer real verification (run it, use MCP Inspector) over hand-waving.
- Close the sitting with recap + checkpoint + "RESUME HERE NEXT TIME," and ask to continue
  or stop.

Begin now: restate the five rules in one sentence, then present the chapter list with time
estimates and ask which chapter to start. Do not write any project code yet.
```

---

## How to use this / verification

1. Copy the fenced prompt above into a fresh chat with your LLM tutor (Claude works well;
   it can also be pasted into Claude Code so the tutor can run the container with you).
2. **First-message check:** the tutor should *restate the five rules*, then show the
   chapter list and ask which chapter to start — and should **not** dump code. If it
   starts building on its own, reply "Rule 1 and 2 — give me the task, don't write it,"
   and it should correct.
3. Sanity checks as you progress:
   - **Ch 1:** `docker compose up` (or `podman`) builds; you can shell in; DB volume persists.
   - **Ch 2:** MCP Inspector connects and lists your one tool; a call returns.
   - **Ch 3:** `get_item` returns a real story from `item/{id}.json`; caching + backoff
     visibly kick in on repeat calls.
   - **Ch 4–6:** `list_stories`/`search_stories`/`get_participants` return typed data;
     `query_db` reads back rows from the named-volume SQLite file.
   - **Ch 7:** An MCP client (Claude) drives a small crawl end-to-end (top stories + their
     rosters) and the DB fills.
4. If you'd like, I can also generate a **ready-to-run starter repo** (Dockerfile,
   compose, `.env.example`, a Hello-MCP server) so the tutorial starts from a scaffold
   instead of an empty folder — say the word.
```

---

## Publishing to GitHub (execution)

Goal: update the README, then commit **only** `README.md` and this plan file to a new
**public** GitHub repo named **`mcp-hn-tutorial`**. Everything else in the workspace
(`mountaineers-mcp/`, the rest of `nimbalyst-local/`) stays untracked.

**Steps:**
1. Overwrite `README.md` with the content below.
2. `git init` at the workspace root (default branch `main`).
3. Stage exactly two files via the commit-proposal tool:
   `README.md` and `nimbalyst-local/plans/i-want-you-to-ancient-rossum.md`.
   Commit message: `docs: add MCP tutorial plan and project README`.
4. `gh repo create mcp-hn-tutorial --public --source=. --remote=origin --push`
   (gh is authenticated as `amjad-uruk`, SSH, `repo` scope).

**New `README.md` content:**

```markdown
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
[`nimbalyst-local/plans/i-want-you-to-ancient-rossum.md`](nimbalyst-local/plans/i-want-you-to-ancient-rossum.md)

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
```
