# Mutation Testing

Mutation testing injects small code changes (mutants) and checks whether the
test suite catches them. A surviving mutant means a test exists that covers
the line but doesn't assert on what the line actually does — a genuine
specification gap, not a coverage gap.

Use it after achieving the line/branch coverage targets, to find tests that
pass for the wrong reasons.

---

## Tool (Python): mutmut

```bash
uv add --dev mutmut
uv run mutmut run
uv run mutmut results
uv run mutmut show <id>   # inspect a specific surviving mutant
```

Configure in `setup.cfg`:

```ini
[mutmut]
# Core logic modules only — see "What to mutate" below
paths_to_mutate=src/mypackage/core.py,src/mypackage/logic.py

# Run without coverage instrumentation — faster, and mutmut doesn't need it
runner=uv run python -m pytest tests/ -x -q --no-header --no-cov

# Suppress equivalent mutations — see "Equivalent mutations" below
do_not_mutate=
    logger.
    raise RuntimeError(f"Required
```

---

## What to mutate

**Include**: pure logic modules — anything that takes inputs and returns
outputs without touching external systems.

**Exclude**:
- Infrastructure boundary modules: HTTP clients, database layers, queue
  consumers, anything that calls an external API
- Entry points (`main.py`, CLI handlers)
- Integration test scaffolding

Surviving mutations in boundary modules almost always reflect mock fidelity
gaps, not specification gaps. Chasing them produces tests that assert on
implementation details of the mock rather than observable behaviour.

---

## Equivalent mutations

Some mutations change code in ways that cannot affect observable behaviour.
Killing them would require testing implementation details. Suppress them with
`do_not_mutate` (comma-separated substrings; mutmut skips any line containing
one of these strings):

| Pattern | Why it's equivalent |
|---------|---------------------|
| `logger.` | Log messages are not assertions; silencing them breaks nothing observable |
| Solver/LP variable name strings | Solvers ignore internal variable names |
| `raise SomeError(f"message text` | Tests should check exception type, not message text |
| Optional env var name mutations | When the var isn't set, the default is returned either way |
| String formatting in `__str__` / `__repr__` | Display strings are rarely asserted |

---

## Score targets

| Context | Target |
|---------|--------|
| Core logic modules (pure functions) | ≥ 65% overall; aim for 75–80% per module |
| Config / boundary code | Lower is acceptable — don't chase it |

A low score on a module is sometimes a structural signal, not a testing gap.
If a module mixes pure computation with I/O (e.g. cost calculations tangled
with HTTP calls), surviving mutations are the test telling you to extract the
pure logic into a separate, fully testable module. Fix the structure before
adding more tests.

---

## Categorising surviving mutants

Before writing tests, categorise each survivor:

**Category 1 — Equivalent mutation**: the change cannot affect observable
behaviour. Annotate with `do_not_mutate` and move on.

**Category 2 — Genuine spec gap**: a real behaviour the tests don't pin down.
Write a test that catches it. Good signals:
- Arithmetic constants (slot duration, efficiency exponents, unit conversions)
- Boundary conditions (threshold comparisons, off-by-one in ranges)
- Sign conventions (cost vs revenue, charge vs discharge)
- Default field values on dataclasses

Run `uv run mutmut show <id>` to inspect each mutant before deciding.

---

## CI integration

Mutation testing is too slow to run on every PR (~30–60 min for 500 mutants).

**Recommended approach: `workflow_dispatch` only.**

Add a `mutation` job to the project's CI workflow, gated on
`github.event_name == 'workflow_dispatch'`. Trigger it manually after closing
spec-gap issues to take a score snapshot. Upload `mutmut results` as an
artifact.

```yaml
  mutation:
    name: mutation
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - uses: astral-sh/setup-uv@<sha>
      - run: uv sync --frozen
      - run: uv run mutmut run
      - run: uv run mutmut results > mutation-results.txt
      - uses: actions/upload-artifact@<sha>
        if: always()
        with:
          name: mutation-results
          path: mutation-results.txt
```

On **private repos**, Actions minutes are limited (2,000/month free). A weekly
schedule would consume 120–240 minutes/month. Manual dispatch keeps this at
zero until you deliberately want a snapshot.

On **public repos**, minutes are unlimited — a weekly schedule is fine.

Once the score stabilises above the target, consider adding a score-floor
check (parse `mutmut results` and exit non-zero below threshold) to catch
regressions on new code.

---

## Workflow for a new project

1. Achieve ≥80% line coverage first.
2. Add mutmut, configure `paths_to_mutate` for core logic only.
3. Run `uv run mutmut run` locally.
4. Categorise survivors: annotate equivalents, note genuine gaps.
5. Write tests for genuine gaps; re-run to confirm they're killed.
6. Target ≥65% overall; ≥75% on the most critical modules.
7. Add `workflow_dispatch` mutation job to CI.
