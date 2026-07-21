---
name: dev-workflow
description: Development workflow and conventions for the AI-Chess-Assistant project — virtualenv activation, adding dependencies, project layout, module boundaries, and git conventions. Use this whenever setting up the project, running it, adding or updating a Python dependency, creating a new module or file, or committing changes. Consult it before running any pip or python command in this repo, so commands land in the right environment and the right directory.
---

# AI-Chess-Assistant dev workflow

This project runs in **WSL (Ubuntu)**, not Windows. Every command here assumes a bash
shell inside WSL. If a command is issued from a Windows shell, it will use the wrong
Python and the wrong paths.

## Layout

```
AI-Chess-Assistant/            <- git repo root
├── venv/                      <- virtualenv (not committed)
├── README.md
└── AI Chess Assistant/        <- all source lives here (note: name contains spaces)
    ├── main.py                <- entry point / orchestration
    ├── config.ini             <- API base URLs and settings
    ├── requirements.txt
    ├── chess_api/             <- fetching data from Chess.com
    ├── chess_analyser/        <- turning raw games into insights
    └── ai_toolbox/            <- LLM interaction, exercise generation
```

**The source directory name contains spaces.** Always quote it in shell commands:
`cd "AI Chess Assistant"`. Unquoted, bash reads it as three separate arguments and the
command fails confusingly. If the directory is ever renamed (e.g. to `src`), update
this file to match — a stale layout section is worse than none.

## Environment

Activate the virtualenv before running or installing anything:

```bash
cd ~/programming/AI-Chess-Assistant
source venv/bin/activate
```

The `(venv)` prefix in the prompt confirms it is active. Without it, `pip install`
writes to system Python and the package will appear to vanish next time — this is the
most common source of "but I installed it" confusion, so check the prefix when an
import fails unexpectedly.

Verify with `which python` — it should point inside `venv/`, not `/usr/bin/`.

## Dependencies

This project uses plain `pip` + `requirements.txt` (not Poetry, uv, or conda). Keep it
that way unless the user decides otherwise; mixing tools mid-project creates
environments that disagree with each other.

To add a package:

```bash
source venv/bin/activate
pip install <package>
cd "AI Chess Assistant" && pip freeze > requirements.txt
```

`requirements.txt` lives inside the source directory, not the repo root. Regenerate it
in the same step as installing — a dependency that works locally but is missing from
the file is invisible until the project is cloned somewhere else.

To rebuild the environment from scratch:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r "AI Chess Assistant/requirements.txt"
```

## Running

```bash
source venv/bin/activate
cd "AI Chess Assistant"
python main.py
```

Run from inside the source directory so the sibling packages (`chess_api`,
`chess_analyser`, `ai_toolbox`) resolve as imports.

## Module boundaries

Each package has one job. Respecting this keeps the project testable as it grows —
see the `feature-planning` skill for how to decide where new code belongs.

- **`chess_api/`** — talks to the Chess.com API. Network I/O only. Returns raw data;
  makes no judgements about it.
- **`chess_analyser/`** — pure logic over game data. No network calls, no LLM calls.
  Because it is pure, it is the easiest layer to test and the most valuable to get right.
- **`ai_toolbox/`** — LLM prompting and exercise generation. The only place that talks
  to a language model.
- **`main.py`** — wires the three together. Holds orchestration, not domain logic.

## Configuration

Settings live in `config.ini`, read with Python's `configparser`. Add new settings
there rather than hardcoding them.

**Secrets do not go in `config.ini`** — it is committed to git. API keys belong in a
`.env` file (already gitignored) or an environment variable. This matters as soon as
an LLM provider key enters the project.

## Git

The repo uses Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`). Example from
the history: `feat: creating initial skeleton`.

Commit only when asked. `venv/` must never be committed — it is large, machine-specific,
and rebuildable from `requirements.txt` in seconds.

## Context

The user is experienced with Python but new to Linux, Bash, and WSL. When running shell
commands, favour explaining what a command does over just issuing it, and prefer
standard tools over clever one-liners.
