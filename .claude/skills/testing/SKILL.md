---
name: testing
description: How to write and run tests for the AI-Chess-Assistant project — unit tests for the analysis layer, integration tests for the API and LLM layers, and end-to-end tests for the full pipeline. Use this whenever the user wants to add tests, asks how to test a piece of code, wants to set up the test suite, asks about pytest, fixtures, mocking Chess.com or the LLM, or asks whether a change is covered. Consult it before writing any test file so tests land in the right place, at the right level, and stay fast and deterministic.
---

# Testing AI-Chess-Assistant

This project's architecture (see `feature-planning`) exists partly to make testing
possible. The three-layer pipeline maps directly onto three kinds of test:

```
Chess.com API  →  raw games  →  analysis  →  insights  →  LLM  →  exercises
   chess_api/                chess_analyser/          ai_toolbox/
   integration               unit                    integration
   └────────────────────── e2e (whole pipeline) ──────────────────────┘
```

The single most important rule: **match the test level to the layer.** Testing pure
analysis logic with a live network call is slow and flaky; testing the API wrapper with
a pure-function test misses the thing that actually breaks. Pick the level from the
layer, not from habit.

## Toolchain

This project uses plain `pip` + `requirements.txt` — keep testing tools consistent with
that. Standardise on **pytest** (not `unittest`, not `nose`). It is the de-facto Python
standard, has the least ceremony, and its fixtures are what the integration tests below
depend on.

First-time setup (nothing test-related is installed yet):

```bash
source venv/bin/activate
pip install pytest pytest-mock responses
cd src && pip freeze > requirements.txt
```

- **pytest** — the runner and assertion framework.
- **pytest-mock** — the `mocker` fixture, for patching the LLM and engine calls.
- **responses** — mocks the Chess.com HTTP layer without a live network call.

If the user prefers a lighter footprint, `pytest` alone plus `unittest.mock` covers most
of this; `responses` is the one genuinely worth adding, because hand-mocking `requests`
is tedious and error-prone.

Keep testing dependencies in the same `requirements.txt` for now. If the project grows,
splitting them into `requirements-dev.txt` is the natural next step — flag it, don't do
it silently.

## Layout

Tests live in a top-level `tests/` directory at the repo root, mirroring the package
structure under `src/`:

```
AI-Chess-Assistant/
├── src/
│   ├── chess_api/
│   ├── chess_analyser/
│   └── ai_toolbox/
└── tests/
    ├── conftest.py            <- shared fixtures (sample games, fake LLM)
    ├── data/                  <- saved Chess.com responses, sample PGNs
    ├── unit/
    │   └── test_analyser_*.py
    ├── integration/
    │   ├── test_chess_api_*.py
    │   └── test_ai_toolbox_*.py
    └── e2e/
        └── test_pipeline.py
```

Keeping `tests/` outside `src/` mirrors the package layout without cluttering it, and
makes "run everything" a single unambiguous target. Name files `test_*.py` and functions
`test_*` so pytest discovers them with no configuration.

Because the source lives under `src/`, make it importable from the tests. The simplest
approach that needs no path hacking is a minimal `pyproject.toml` or `pytest.ini` at the
repo root:

```ini
# pytest.ini
[pytest]
pythonpath = src
testpaths = tests
markers =
    e2e: end-to-end tests that exercise the whole pipeline (may be slow)
```

`pythonpath = src` lets `import chess_analyser` resolve in tests exactly as it does when
running `main.py` from inside `src/`. Without it you get `ModuleNotFoundError` and the
temptation to insert `sys.path` hacks — avoid those; they rot.

## The three levels

### Unit tests — `chess_analyser/`

This is where the project's value concentrates, so it is where test coverage should be
deepest. The analysis layer reasons over game data — no network, no LLM. A test here
should need only a saved game and no internet.

- Feed a known game (from `tests/data/`) in, assert on the insight that comes out.
- Cover the boring edges: a player with zero games, a single game, all wins, malformed
  PGN. These are where analysis logic actually breaks.
- **A network or LLM call in a `chess_analyser` test is a design smell** — those belong
  in the outer layers. Fix the code, not the test.

**The Stockfish exception.** `chess_analyser/` now uses Stockfish (via `python-chess`) to
find real mistakes, and the engine is a subprocess — a boundary, and a slow one. Keep the
pure move/statistics logic in functions that take *already-computed* evaluations, so most
unit tests never launch the engine and stay fast:

```python
# pure: no engine, tests instantly
def classify_move(eval_before, eval_after) -> str:
    if eval_before - eval_after > 3.0:
        return "blunder"
    ...
```

For the thin slice of code that does drive the engine, write a small number of tests that
mock the `python-chess` engine call (assert we ask it to analyse the right positions), and
optionally one real-engine test marked `slow` that runs Stockfish for a single position as
a smoke check. Skip the real-engine tests in the fast inner loop with `-m "not slow"`.
Don't let a real Stockfish subprocess into the bulk of the unit suite — that reintroduces
exactly the slowness the pure/boundary split is there to avoid.

```python
# tests/unit/test_analyser_weaknesses.py
from chess_analyser import find_time_trouble_losses

def test_flags_games_lost_on_time(sample_bullet_games):
    result = find_time_trouble_losses(sample_bullet_games)
    assert result.count == 2
    assert all(g.termination == "timeout" for g in result.games)
```

### Integration tests — `chess_api/` and `ai_toolbox/`

These layers cross a boundary — the network for `chess_api`, the LLM provider for
`ai_toolbox` — so the test's job is to verify *our* code against a *controlled* stand-in
for that boundary. **Never hit the real Chess.com API or a real paid LLM in the test
suite.** It makes tests slow, non-deterministic, and (for the LLM) costs money on every
run.

**Chess.com (`chess_api/`)** — mock the HTTP layer with `responses`. Save one real
response into `tests/data/` once (see the `curl` in `feature-planning`), then replay it:

```python
# tests/integration/test_chess_api_client.py
import responses
from chess_api.client import ChessApiWrapper

@responses.activate
def test_get_players_games_parses_response(sample_api_json):
    responses.add(
        responses.GET,
        "https://api.chess.com/pub/player/hikaru/games/2026/06",
        json=sample_api_json,
        status=200,
    )
    games = ChessApiWrapper().get_players_games("hikaru", 2026, 6)
    assert len(games) > 0
```

Test the unhappy paths too: a 404 for an unknown player, a 429 rate-limit, a malformed
body. That error handling is exactly what an integration test should pin down.

**LLM (`ai_toolbox/`)** — patch the provider client with the `mocker` fixture so no real
call goes out. Assert on **what we send** (the prompt is built correctly from the
insights) and on **how we handle what comes back**, not on the model's wording, which is
non-deterministic and not ours to test.

```python
# tests/integration/test_ai_toolbox_exercises.py
def test_builds_prompt_from_insights(mocker, sample_insights):
    fake = mocker.patch("ai_toolbox.client.messages.create")
    fake.return_value = fake_exercise_response()

    generate_exercises(sample_insights, rating=1400)

    prompt = fake.call_args.kwargs["messages"][0]["content"]
    assert "1400" in prompt          # rating is passed through
    assert "time trouble" in prompt  # the real weakness drives the prompt
```

When the project does wire up a real LLM, consult the `claude-api` skill for the exact
client shape to patch.

### End-to-end tests — the whole pipeline

One or two e2e tests prove the layers actually connect: given a fake Chess.com response
and a fake LLM, does `main.py`'s orchestration produce exercises? Mock only the two outer
boundaries (network in, LLM out) and let everything real in between run. Mark them `e2e`
so they can be skipped for a fast inner loop:

```python
import pytest

@pytest.mark.e2e
def test_full_pipeline_produces_exercises(mocker, mock_chess_api, mock_llm):
    exercises = run_assistant(username="hikaru", month="2026-06")
    assert len(exercises) > 0
```

Keep e2e tests few. They are the most valuable for catching wiring bugs and the most
expensive to maintain, so a handful covering the main path beats exhaustive coverage
here — put the exhaustive cases in unit tests.

## Fixtures and test data

Put shared fixtures in `tests/conftest.py`; pytest injects them by name automatically.
The high-value ones for this project:

- **Real saved data**, not hand-written dicts. Fetch one genuine Chess.com response and
  commit it under `tests/data/`. The project's design is downstream of what the API
  actually returns (see `feature-planning`), so tests built on a guessed shape test the
  guess, not the code.
- **A fake LLM response** helper, so integration and e2e tests share one definition of
  "what the model returns."

```python
# tests/conftest.py
import json, pathlib, pytest

DATA = pathlib.Path(__file__).parent / "data"

@pytest.fixture
def sample_api_json():
    return json.loads((DATA / "hikaru_2026_06.json").read_text())
```

## Running tests

```bash
source venv/bin/activate    # tests import the project; the venv must be active
pytest                      # everything, from the repo root
pytest tests/unit           # just the fast pure-logic tests
pytest -m "not e2e"         # skip the slow end-to-end tests (fast inner loop)
pytest -k weakness -v       # tests matching a name, verbose
pytest --lf                 # only what failed last run
```

Run `pytest` from the **repo root**, not from inside `src/` — `pytest.ini` sets
`pythonpath` and `testpaths` relative to the root. This is different from running
`main.py`, which must be run from inside `src/`; the mismatch trips people up, so check
where you are when imports fail.

A green run means the analysis logic holds, the boundaries are handled, and the wiring
connects — in that order of how often each will actually save you.

## Writing tests as you build

Prefer growing the suite alongside the code, not in a big batch afterwards. The natural
rhythm for this project:

1. New pure logic in `chess_analyser/` → add a unit test with a saved game in the same
   step. This is cheap and catches the most bugs.
2. New boundary code in `chess_api/` or `ai_toolbox/` → add an integration test with a
   mocked boundary.
3. New wiring in `main.py` → extend or add the single e2e test.

Don't write tests for code that is still a `return`-only stub (as several modules
currently are). Test behaviour once there is behaviour — a test asserting that an empty
function returns `None` only pins down that it is empty.

## Working style

The user is a professional Python developer, so pytest idioms and fixtures need no
explanation — but they are building this project to learn, so prefer a few clear,
readable tests they can follow over a large generated suite. When you add tests, say
which level each one is and why it belongs there, so the mapping from layer to test level
becomes second nature. Explain any shell mechanics (venv, working directory) rather than
just issuing the command — they are new to Linux, Bash, and WSL.