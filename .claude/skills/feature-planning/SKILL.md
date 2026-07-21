---
name: feature-planning
description: How to plan work on the AI-Chess-Assistant project before writing code — tracing the pipeline from Chess.com data through Stockfish analysis to LLM-generated chess exercises, deciding which module owns a piece of logic, and slicing work into small verifiable steps. Use this whenever the user wants to add a feature, asks how to structure or approach something, says they are not sure where to start, asks what to build next, or requests a plan. Prefer planning with the user over jumping straight to code on this project — they are building it to learn, so the design reasoning is part of the deliverable.
---

# Planning work on AI-Chess-Assistant

## What this project is

An assistant that fetches a player's recent games from Chess.com, analyses them using Stockfish to find
that player's actual weaknesses, and uses an LLM to generate personalised chess
exercises targeting those weaknesses, calibrated to their rating.

Keep the end goal in view when planning: the output is **exercises a specific human
should practise**, not a generic analysis dump. A feature that does not eventually feed
better exercises is probably a detour worth naming as one.

## The pipeline

Every feature lands somewhere on this path. Locating it is the first planning step:

```
Chess.com API  →  raw games  →  analysis  →  insights  →  LLM  →  exercises
   chess_api/                chess_analyser/          ai_toolbox/
```

Deciding where code belongs:

- Does it make a **network call to Chess.com**? → `chess_api/`
- Does it **reason about game data** (patterns, weaknesses, statistics)? → `chess_analyser/`
- Does it **build a prompt or call an LLM**? → `ai_toolbox/`
- Does it **connect the above in sequence**? → `main.py`

The boundary that matters most is keeping `chess_analyser/` free of network and LLM
calls. Pure functions over game data can be tested with a saved game file and no
internet, which is what makes the analysis layer improvable later. Once an LLM call
leaks into it, every test becomes slow, non-deterministic, and costs money.

## Before writing code

**1. Look at the real data first.**
This project's design is downstream of what Chess.com actually returns. Guessing the
shape of that response and building on the guess is the fastest way to write code that
must be thrown away. Fetch one real response and inspect it before designing anything
that consumes it.

```bash
curl -s "https://api.chess.com/pub/player/<username>/games/2026/06" | head -c 2000
```

Note that Chess.com returns games as **PGN text inside JSON** — parsing that into
positions is real work, and usually the step people underestimate. Plan for it
explicitly rather than treating it as a detail.

**2. Name the smallest useful slice.**
Prefer one thin vertical slice that runs end to end over a complete layer that cannot
be executed yet. "Fetch one month of games and print how many were losses" is a better
first step than a fully designed analysis module, because it proves the API works,
surfaces the real data shape, and produces something observable.

**3. Surface the open decisions rather than silently picking.**
This project has genuine unmade choices. When one is load-bearing for the task at hand,
raise it instead of quietly deciding:

- Which LLM provider and SDK (nothing is installed yet — `requirements.txt` has only
  `requests`). Consult the `claude-api` skill before writing any LLM integration code.
- Whether to store fetched games locally or refetch each run.
- Whether a chess engine (e.g. Stockfish via `python-chess`) is needed to find real
  mistakes, or whether Chess.com's own accuracy data is enough. This is the biggest
  architectural fork in the project — engine analysis is far more powerful and far more
  work.
- What "weakness" concretely means (openings? endgames? tactical motifs? time trouble?).
  Vague here means the analysis layer cannot be specified.

## Producing a plan

Keep plans proportional — a few steps for a small feature, not a document. A useful
plan for this project states:

1. **The goal**, in terms of the pipeline above.
2. **Which module(s)** change, and why that boundary.
3. **The steps**, each small enough to run and see working.
4. **How to verify** — what command to run and what output means success.
5. **Open questions** that need the user's answer before starting.

Then confirm the plan before implementing. The user is learning this codebase as they
build it, so a plan they have agreed to is worth more than code that arrives unexplained.

## Working style

The user is a professional Python developer, so Python idioms need no explanation. They
are new to Linux, Bash, and WSL — so explain shell commands and environment mechanics.

Prefer small, reviewable changes over large generated files. Code the user has not
followed is code they cannot maintain or debug, which defeats the point of the project.
