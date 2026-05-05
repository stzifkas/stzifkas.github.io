---
layout: post
title: "Sieve: The Same Failure, Smaller"
date: 2026-05-05
categories: [llm-agents, developer-tools, compression, coding-agents]
image: /assets/sieve-cover.png
---

![Sieve: The Same Failure, Smaller](/assets/sieve-cover.png)

There’s a weird little failure mode in coding agents that doesn’t look dramatic at first.

The agent runs a test.

The test fails.

The terminal returns a wall of output.

The agent reads it, makes a change, runs the test again, and then the same wall comes back with one tiny difference hiding somewhere inside it.

Again.

And again.

After a few turns, the conversation context starts looking less like an engineering process and more like a filing cabinet full of duplicate police reports. Same traceback. Same pytest header. Same plugin list. Same failing test name. Same summary. Maybe one line changed. Maybe nothing changed. Maybe the whole thing is noise wearing a useful hat.

That’s the problem Sieve tries to solve.

Sieve is transparent feedback compression middleware for LLM coding agents. It sits between an agent and its tools. When the tool returns output, Sieve parses it before the text enters the model’s context. Then it emits a smaller version that keeps the useful facts and drops the repeated junk.

Simple idea.

Annoyingly useful.

The whole design lives in [Sieve on GitHub](https://github.com/stzifkas/sieve) repository, but the short version is this: coding agents are drowning in observations.

A JetBrains / NeurIPS 2025 result says that 83.9% of tokens in coding-agent trajectories are tool observations. That’s a ridiculous amount of context spent on terminal output. Worse, most of it gets re-read on later turns because the transcript keeps growing.

A failed command doesn’t just cost tokens once. It lingers.

It becomes part of the room.

And sometimes, after the fifth identical traceback, you start hearing the failure speak backwards.

## The blob problem

Most agent systems treat tool output as a blob.

Run command. Get stdout. Get stderr. Append everything. Let the model figure it out.

That works until it doesn’t.

A `pytest` result has shape. It has failed node IDs, assertion lines, file locations, expected values, actual values, captured logs, and summary counts.

A Python traceback has frames, line numbers, exception types, messages, and a final cause.

A `pip` failure usually has some real reason buried inside a lot of resolver noise.

A TypeScript compiler run has diagnostic codes, file paths, ranges, symbols, and messages.

A compiler error has a location, an error class, and often a repeated cascade that only exists because the first thing broke.

So Sieve doesn’t just cut text. It parses.

That matters.

Truncation says: “Here are the first or last N lines. Good luck.”

Parsing says: “Here’s the failure. Here’s where it happened. Here’s what changed since last time.”

Those are very different promises.

## Compression can lie

I’m suspicious of compression in agent loops.

It can hide the one line that matters. It can flatten a failure until the model sees something neat and wrong. It can turn a real debugging session into a bedtime story.

So Sieve has to be boring in a very specific way.

If the compressed output would be larger than the raw output, Sieve passes the raw text through unchanged. That happens with small `mypy` outputs, terse ESLint messages, and some tiny generic logs. There’s no need to force the machinery just to make a chart prettier.

The invariant is simple: never return something larger than the original.

Even when the visible output passes through unchanged, Sieve can still extract structured items behind the scenes. That gives it memory for later. If the same thing appears again, the next output can be compared against the previous one.

That’s where the real value starts showing up.

## The first numbers

There’s a small fixture benchmark in `tests/fixtures/`.

Run it with:

```bash
uv run python -m benchmarks.run
```

Current result:

| Category | Samples | Raw chars | Compressed chars | Reduction |
|---|---:|---:|---:|---:|
| pytest | 7 | 13,475 | 1,292 | 90.4% |
| pip | 2 | 9,505 | 138 | 98.5% |
| runtime | 6 | 3,150 | 1,068 | 66.1% |
| gcc | 1 | 1,318 | 721 | 45.3% |
| tsc | 2 | 1,264 | 708 | 44.0% |
| generic | 1 | 480 | 433 | 9.8% |
| eslint | 1 | 844 | 780 | 7.6% |
| mypy | 2 | 758 | 756 | 0.3% |
| total | 22 | 30,794 | 5,896 | 80.9% |

Total reduction: 80.9%.

The big wins are where you’d expect. `pytest` is chatty. `pip` can be absurd. Runtime traces usually have enough structure to compress well.

The low numbers are fine. Actually, I like them. They mean the compressor isn’t trying to perform a magic trick on text that’s already small.

A tool like this has to know when to shut up.

## The repeated pytest failure

The most interesting case is the repeated failure.

Here’s the kind of raw test output we all know too well:

```text id="vhlyoc"
============================= test session starts ==============================
platform linux -- Python 3.12.0, pytest-8.1.1, pluggy-1.4.0
... [40 lines of header + per-test output] ...
=================================== FAILURES ===================================
________________________________ test_user_update ________________________________
    def test_user_update(self):
        ...
>       assert response.status_code == 200
E       AssertionError: assert 403 == 200
tests/test_views.py:89: AssertionError
... [equivalent block for test_user_delete] ...
=========================== short test summary info ============================
FAILED tests/test_views.py::TestUserViewSet::test_user_update - AssertionError
FAILED tests/test_views.py::TestUserViewSet::test_user_delete - AssertionError
========================= 2 failed, 140 passed, 0 warnings ====================
```

That’s 1,818 characters in the fixture.

Sieve turns it into this:

```text id="5tngfk"
PYTEST: 2 failed, 140 passed (142 total)
FAIL tests/test_views.py::TestUserViewSet::test_user_update (test_views.py:89)
  expected 200, got 403
FAIL tests/test_views.py::TestUserViewSet::test_user_delete (test_views.py:102)
  expected 204, got 403
Pattern: All failures return 403 in test_views.py
```

297 characters.

83.7% smaller.

More readable, too.

Then the agent fixes one test and runs the suite again. Now Sieve can emit a delta:

```text id="0gnoqn"
PYTEST DELTA (turn 2)
PASS tests/test_views.py::TestUserViewSet::test_user_update now passes
STILL FAIL tests/test_views.py::TestUserViewSet::test_user_delete (line 102) - expected 204, got 403
Result: 1 failed, 141 passed (142 total)
```

That’s the useful thing.

The model sees progress. One failure went away. One remains. Same file. Same status-code pattern. Keep going.

In a 5-turn delta scenario where the same pytest failure repeats, cumulative compression reaches 86.3%. Turn 1 is 297 chars. Turns 2–5 collapse to 238 chars each.

That’s the part I care about.

A coding agent shouldn’t have to re-read the same traceback like a cursed bedtime story.

## End-to-end run: SWE-bench Lite with Cursor

Fixture numbers are nice. Real agent runs matter more.

Sieve has paired baseline-vs-Sieve runners for SWE-bench Lite using Cursor CLI / Composer-2. The scoring goes through the official `swebench.harness.run_evaluation` Docker harness.

Small run, four scored instances:

| Profile | Instances scored | Resolved | Resolve rate |
|---|---:|---:|---:|
| baseline | 4 | 2 | 50.0% |
| sieve | 4 | 2 | 50.0% |

Same resolved count.

Now the context numbers:

| Metric | baseline | sieve |
|---|---:|---:|
| patch chars | 21,242 | 19,956 |
| agent-facing chars | 47,688 | 11,613 |
| raw chars | 47,688 | 40,416 |
| compression ratio | 0% | 71.3% |

Agent-facing context dropped from 47,688 chars to 11,613 chars.

That’s a 75.6% reduction.

The repair rate stayed the same in this small trial. I’m happy with that result because the first goal here is safety. Compress the observation channel. Keep the repair signal. Don’t break the agent.

To reproduce:

```bash id="j1n9ea"
bash scripts/run_cursor_swe_bench_profiles.sh --resume \
  --eval-with-harness --harness-namespace none

PYTHONPATH=src python3 -m benchmarks.swe_bench_compare \
  --baseline artifacts/cursor-swe-bench-lite.baseline.jsonl \
  --sieve    artifacts/cursor-swe-bench-lite.sieve.jsonl
```

The runner builds each SWE-bench instance harness container, mounts the workspace at `/testbed`, runs the agent, writes predictions, and lets the official harness fill in the authoritative `resolved` value.

There’s also a trajectory-only benchmark that replays `.traj` files for token counts. Useful for measuring transcripts. For repair scoring, use the harness.

## CI logs are worse

CI logs are where this problem gets ugly.

A GitHub Actions failure can include workflow YAML, setup logs, cache output, dependency installation, compiler output, test output, shell wrappers, warnings, and several thousand lines of “almost useful” text.

A human skims it with instinct. We jump to the end, search for `Error`, scroll back to the first real failure, ignore the repeated garbage, and build a mental model.

An agent gets text.

Lots of it.

Sieve includes a benchmark path for CI-Repair-Bench, built from real GitHub Actions failures. Each observation is workflow plus flattened logs, with the gold diff excluded. So the measurement is about diagnostic bulk, without patch leakage.

Run:

```bash id="o0b53t"
uv sync --group swe-eval
uv run python -m benchmarks.ci_repair_bench --compare --json
```

That covers all 567 rows of the `ci-benchmark-user/ci-repair-bench` dataset.

For repair scoring, use the upstream paper’s harness. Sieve’s measurement here is narrower: how much noisy diagnostic material can be reduced before it hits the model?

That’s already a big question.

## What’s inside

Sieve currently has parsers for:

| Area | Tools |
|---|---|
| testing | `pytest` |
| Python failures | tracebacks |
| typing | `mypy`, `tsc` |
| linting | `eslint` |
| native builds | `gcc`, `clang` |
| packaging | `pip` |
| fallback | generic text |

Output formats:

| Format | Use |
|---|---|
| plain | readable compressed text |
| structured | JSON-style output for downstream use |
| XML | useful for tagged agent contexts |
| minimal | smallest practical form |

The library itself has no dependencies. The MCP proxy needs the optional `mcp` extra. Python 3.11 or newer.

## Direct usage

The basic API is small:

```python id="c15aoa"
from sieve import CompressSession

session = CompressSession()
result = session.compress(
    command="pytest tests/",
    stdout=raw_stdout,
    stderr=raw_stderr,
    exit_code=1,
)

print(result.text)
print(result.stats.compression_ratio)
```

`CompressSession` keeps state, so later calls can emit deltas against earlier observations.

There’s also a decorator:

```python id="cur8cp"
import subprocess
from sieve import wrap_tool

@wrap_tool
def run_bash(command: str) -> tuple[str, str, int]:
    p = subprocess.run(command, shell=True, capture_output=True, text=True)
    return p.stdout, p.stderr, p.returncode
```

Now `run_bash(...)` returns compressed output. The decorator holds the session.

Configuration looks like this:

```python id="qun68y"
from sieve import CompressConfig, CompressSession, OutputFormat

session = CompressSession(CompressConfig(
    format=OutputFormat.STRUCTURED,   # plain | structured | xml | minimal
    delta_mode=True,
    include_pattern_hints=True,
    max_raw_lines=50,
))
```

That’s the whole idea at library level. Keep the tool interface familiar. Clean up the observation before it becomes context.

## MCP proxy

The MCP proxy is probably the cleanest integration.

`sieve.integrations.mcp` wraps any upstream MCP server. The agent talks to the proxy as if it were the original server. The proxy forwards `tools/list` and `tools/call`, then compresses each returned `TextContent` block through one shared `CompressSession`.

Install:

```bash id="9d8sqz"
pip install 'sieve[mcp]'
```

Example config:

```json id="tszu20"
{
  "mcpServers": {
    "sieve-demo": {
      "command": "python",
      "args": [
        "-m", "sieve.integrations.mcp",
        "--",
        "npx", "-y", "@modelcontextprotocol/server-everything"
      ]
    }
  }
}
```

For regular tools, Sieve uses the tool name as a parser hint. For shell-like tools with a `command`, `cmd`, or `shellCommand` argument, it forwards the real command string into parser detection. So `pytest`, `mypy`, `pip`, and friends can be recognized properly.

The agent sees the same tools.

The text comes back cleaner.

Fire walk with middleware.

## Why I built it this way

I don’t think the next step for coding agents is always more agency.

Sometimes the agent is already doing the right loop. Read, edit, run, inspect. The weak point is the channel between the tool and the model.

Right now that channel is too raw.

Terminals were made for humans. Humans are good at skipping. We see a pytest header and ignore it. We see the same traceback twice and compare the important bits. We search visually. We develop little debugging reflexes.

Models don’t get that for free. They receive the text we give them. If we give them repeated terminal output for fifteen turns, we shouldn’t be shocked when the context turns into a swamp.

Sieve is a filter for that swamp.

It keeps the failure. It keeps the location. It keeps the changed state. It keeps the pattern when there is one. It lets raw output through when compression would add no value.

Or, to put it in the language of a very strange night drive: it doesn’t solve the mystery, it just stops every road sign from screaming at the detective.

It’s a small layer, but small layers matter in agent systems. The prompt matters. The tool schema matters. The diff format matters. The order of files matters. The phrasing of test failures matters. The observation channel matters too.

Maybe more than we’ve been treating it.

## Current status

The current results are early:

Fixture corpus: 80.9% total reduction.

Repeated pytest delta scenario: 86.3% cumulative compression.

Small SWE-bench Lite paired Cursor Composer-2 run: same scored resolve rate, 75.6% less agent-facing context.

CI-Repair-Bench support: compression measurement over 567 real GitHub Actions failures, without gold diff leakage.

That’s enough to make the idea feel real.

The next step is more runs, more parsers, more agents, and more failure cases. Cursor. Codex CLI. MCP clients. More SWE-bench Lite rows. More CI logs. More checks for whether the compressed output preserves the repair signal.

The interesting question isn’t only how much text can disappear.

The interesting question is how little the agent needs to see before it still makes the right next move.

That’s the line Sieve is trying to find.

A thin layer between tools and the model.

A parser with memory.

A way to stop the same failure from haunting every turn.