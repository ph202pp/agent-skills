---
name: TDD skill creation
overview: Create a personal Cursor skill at `~/.cursor/skills/tdd/SKILL.md` that codifies the RED/ORANGE/REVIEW/GREEN/BLUE TDD methodology used throughout the DCA implementation, written generically for any TypeScript project.
todos:
  - id: create-skill
    content: Create ~/.cursor/skills/tdd/SKILL.md with frontmatter, cycle overview, phase definitions, planning guidance, rules, anti-patterns, and example
    status: completed
isProject: false
---

# TDD Skill Creation

## What

A personal skill at `~/.cursor/skills/tdd/SKILL.md` that teaches the agent the five-phase TDD cycle developed during the advanced orders implementation. Generic TypeScript -- no NestJS/Kysely-specific patterns.

## Skill Design

### Frontmatter

- **name**: `tdd`
- **description**: "Develop features using a five-phase TDD cycle: RED (failing happy-path tests), ORANGE (failing error/edge-case tests), REVIEW (pause for human inspection), GREEN (minimal passing implementation), BLUE (refactor and lint). Use when the user says 'tdd', 'test driven', 'red green', 'write tests first', or asks to implement a feature with tests."

### Core Content

The SKILL.md will cover:

**0. Cycle 0: Architecture Checkpoint** -- Before any RED tests are written, document key architectural decisions that affect the entire feature: data types and storage format, API contracts, external dependencies, error strategy. This prevents costly late-stage rework (e.g., switching from floats to integer base units after 9 cycles). Cycle 0 is a one-time step at the start of a multi-cycle feature, not repeated per cycle.

**1. Cycle Overview** -- Mermaid flowchart showing RED -> ORANGE -> REVIEW -> GREEN -> BLUE, with a feedback loop from REVIEW back to RED/ORANGE when gaps are found. REVIEW is the quality gate -- the cycle only advances to GREEN when both agent and human are confident the test suite fully specifies the feature. After BLUE, show the two exit modes (autonomous vs supervised) and the transition to the next cycle.

**2. Phase Definitions** -- Each phase gets a section with:
- **RED**: Write happy-path tests that call the function/method under test. Tests MUST fail (function doesn't exist yet or returns wrong values). Run tests to confirm failures. Commit nothing. **If a RED test passes unexpectedly**, stop and investigate -- either the test is wrong, the feature already exists, or the assertion isn't testing what you think. Do not treat unexpected passes as freebies.
- **ORANGE**: Write error-path and edge-case tests (bad input, missing dependencies, boundary values, external failures). Tests MUST fail. Run tests to confirm failures. Same rule as RED: **unexpected passes must be investigated**.
- **REVIEW**: A collaborative checkpoint between agent and human. The agent performs its own analysis first, then presents findings alongside the test tree for human approval. This phase can loop back to RED/ORANGE if gaps are found. Specifically:
  1. **Agent self-review**: The agent examines the full `describe`/`it` tree and asks itself: Are there missing scenarios? Logical gaps? Contradictory assumptions? Untested boundaries? Does the test set fully specify the feature?
  2. **Present findings**: Print the complete test description tree. If the agent identified gaps or concerns, call them out explicitly (e.g., "I notice we don't test what happens when X is null" or "Should Y also handle the case where Z?").
  3. **Human review**: The user inspects test descriptions, validates assumptions, and may request additions or changes.
  4. **Iterate**: If either the agent or human identifies missing tests, loop back to RED (for new happy-path gaps) or ORANGE (for new error/edge-case gaps). Write the additional tests, run them to confirm they fail, then return to REVIEW.
  5. **Gate**: Only proceed to GREEN when both agent and human are satisfied that the test suite is complete -- no logic gaps, no missing error paths, no unclear assumptions. This is the quality gate.
- **GREEN**: Write the minimal code to make all RED + ORANGE tests pass. No refactoring, no extra features, no cleanup. Run tests to confirm all pass.
- **BLUE**: Refactor for clarity, DRYness, and readability. Run linter. Run the **full** test suite (not just this cycle's tests) to catch cross-cycle regressions. This is the only phase where non-functional changes happen. **Surface forward concerns**: if the implementation reveals new issues that belong in a future cycle, note them explicitly (e.g., "this works but is not atomic -- consider a transaction cycle later") rather than scope-creeping the current cycle. After BLUE completes, the cycle ends with one of two modes:
  - **Autonomous mode**: The user has indicated trust in the flow (e.g., "continue autonomously", "keep going"). The agent commits the cycle and immediately starts the next cycle's RED phase without pausing.
  - **Supervised mode** (default): The agent presents a summary of what was implemented and tested in this cycle, then waits for the user to approve before committing. After the commit, the agent waits for the user to say "continue" before starting the next cycle.
  - The user can switch between modes at any time (e.g., "go autonomous" or "wait for me from now on"). The agent should ask which mode the user prefers at the start of a multi-cycle session if not already established.

**3. Test Structure as Documentation** -- Tests are the primary feature documentation. Enforce a deliberate hierarchy:
- **`describe` blocks** map to the unit under test (function, method, or class) -- these are chapter headings
- **Nested `describe` blocks** group by scenario category (happy path, error handling, edge cases, boundaries) -- these are section headings
- **`it` blocks** read as complete sentences starting with a verb: "returns", "throws", "rejects", "skips", "handles" -- these are the specification
- A reviewer should be able to read the `describe`/`it` tree alone (no code) and understand every behavior the feature supports, every error it handles, and every assumption it makes
- Test descriptions must be specific and include expected values: prefer `"returns '333' when 1000 is split across 3 remaining intervals"` over `"calculates fill amount correctly"`
- Group RED tests under a scenario `describe` like `"happy path"` or `"core behavior"`, and ORANGE tests under `"error handling"`, `"edge cases"`, `"boundary conditions"` -- this makes the REVIEW phase scannable

Example structure the skill should demonstrate:

```
describe("calculateFillAmount")
  describe("core behavior (RED)")
    it("divides total evenly across remaining intervals")
    it("returns '0' when total is fully filled")
    it("accounts for prior fills from execution history")
  describe("error handling (ORANGE)")
    it("returns '0' when remaining intervals is zero")
    it("throws when total_quantity is not a valid integer string")
  describe("boundary conditions (ORANGE)")
    it("handles values exceeding Number.MAX_SAFE_INTEGER")
    it("truncates toward zero on non-even division")
```

**4. Cycle Planning** -- How to break a feature into cycles:
- Each cycle targets one unit of behavior (one service method, one validation rule, one query)
- Cycles should be small enough that RED + ORANGE produce ~5-15 tests
- Create a todo list with one item per phase per cycle before starting

**5. Rules** -- Hard rules distilled from lessons learned:
- Never write implementation before RED tests exist
- Never skip ORANGE -- error paths catch more bugs than happy paths
- REVIEW is collaborative -- the agent must actively look for gaps, not just present the tree and wait
- REVIEW loops back to RED/ORANGE as many times as needed -- never rush past it
- GREEN must be minimal -- resist the urge to refactor during GREEN
- BLUE is for refactoring only -- no new behavior in BLUE
- Run tests after every phase transition; run the FULL suite in BLUE to catch regressions
- One cycle at a time -- finish BLUE before starting the next RED
- Test descriptions are specifications -- write them as if they will be read by someone who has never seen the code
- Unexpected passes in RED/ORANGE are bugs in the test -- investigate, do not ignore
- Architectural decisions belong in Cycle 0, not discovered mid-implementation
- BLUE surfaces forward concerns as backlog items, never scope-creeps the current cycle
- Commit messages reference the cycle: `feat(scope): description [cycle N]`

**6. Anti-patterns** -- Common mistakes:
- Writing tests and implementation together (defeats TDD)
- Skipping ORANGE and only testing happy paths
- Over-implementing in GREEN (adding features the tests don't require)
- Refactoring in GREEN instead of waiting for BLUE
- Making REVIEW a rubber stamp -- the agent must actively audit for gaps, not just list tests and ask "looks good?"
- Skipping the REVIEW loop -- if gaps are found, go back to RED/ORANGE instead of trying to sneak tests into GREEN
- Vague test names like "works correctly" or "handles errors" -- every `it` must say what input produces what output
- Flat test files with no `describe` grouping -- makes REVIEW impossible to scan

**7. Example Cycle** -- A short concrete example showing one cycle through all five phases for a generic `calculateDiscount(price, percentage)` function, with the full `describe`/`it` tree demonstrating the documentation structure.

## File Structure

```
~/.cursor/skills/tdd/
  SKILL.md    (main skill, ~150-200 lines)
```

No supporting files needed -- the skill is self-contained.
