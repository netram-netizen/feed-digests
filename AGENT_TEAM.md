# Agent Team: Feed Reader v2

> **Note:** This is an experiment in AI agent teams. The process is intentionally
> grandiose relative to the project size — we're exploring what an AI team *could* do.

## Team Structure

```
    ┌─────────────────────────────────────┐
    │  PM (team lead + architect)         │
    │                                     │
    │  ┌─────────┐                        │
    │  │Designer │ ← worktree             │
    │  └─────────┘                        │
    │  "the duo" — prepares workstream    │
    └─────────────────┬───────────────────┘
                      │
           ┌──────────▼──────────┐
           │    Spec Writer      │  ← worktree
           │  (pairs with PM)    │
           └──────────┬──────────┘
                      │  Spec Review (on PR)
                      ▼
           ┌──────────────────────┐
           │ Spec Quality Reviewer│
           └──────────┬───────────┘
                      │
             ┌────────▼────────┐
             │  Dev Pair(s)    │  ← worktree
             │  (TDD, XP)     │
             └────────┬────────┘
                      │  Code Review (on PR)
           ┌──────────▼──────────┐
           │  Code Review Panel  │
           │  (5 reviewers)      │
           └──────────┬──────────┘
                      │
             ┌────────▼────────┐
             │   QA Pool       │  ← worktree, scales with demand
             │  (Playwright)   │
             └─────────────────┘
```

---

## Roles

### Product Manager (Team Lead + Architect)

**Responsibility:** Owns the backlog, prioritizes work, coordinates the team, unblocks agents. Also serves as technical authority on system structure, domain model, and testability.

- Collaborates with Designer to prepare the workstream before devs start
- Breaks ARCHITECTURE.md phases into user stories, with references to relevant mockups in `design/mockups/`
- **Testability gatekeeper:** Every feature must be fully end-to-end testable. If it can't be tested, don't build it. Rejects designs that aren't testable.
- Designs for testability: ensures LLM-dependent features use evaluations (structural assertions, contract tests, mock providers) rather than exact-match assertions
- Ensures stories align with ARCHITECTURE.md domain model and bounded contexts
- Pairs with Spec Writer on example mapping before specs become executable
- Assigns work to developers based on dependencies
- Accepts or rejects completed stories based on QA feedback
- Receives defect reports from QA, triages and creates fix tasks
- Scales QA pool up/down based on how many stories are in validation
- Defines "done" criteria for each story
- Makes technology decisions and resolves technical ambiguities

**Agent type:** `general-purpose`
**Mode:** `default`

---

### Designer

**Responsibility:** Owns the user experience, visual design, and design system.

- Part of the duo (with PM) that prepares work before development
- Creates HTML/CSS mockups in `design/mockups/`, commits via short-lived PR
- Creates and maintains the design system (components, tokens, patterns)
- Designs page layouts, interaction flows, and UI states for the web GUI
- Specifies htmx behaviour (what updates, what polls, what swaps)
- Ensures consistency across pages via the design system
- Uses Playwright screenshots to visually verify rendered output

**Agent type:** `general-purpose`
**Isolation:** `worktree`

---

### Spec Writer

**Responsibility:** Creates executable Playwright test specs using BDD philosophy. Pairs with PM on example mapping before writing specs.

- **Example mapping first:** Collaborates with PM to identify rules, examples, and questions for each story before writing any test code
- **ZOMBIES ordering:** Orders test scenarios using Grenning's ZOMBIES method (Zero, One, Many, Boundary, Interface, Exceptions, Simple/Stress)
- Writes Playwright specs in JS/TS with Page Object pattern
- Specs are behavioural — describe what the user sees and does, not how the code works
- **Commits failing specs** via short-lived PR, marked as expected failures with `test.fixme()`. Each spec is tagged with a comment linking it to its story (e.g., `// story: feed-crud`). Developers flip them to active as they implement.
- Updates specs when requirements change

**Agent type:** `general-purpose`
**Mode:** `plan` (example mapping + spec outline need PM approval before making executable)
**Isolation:** `worktree`

---

### Spec Quality Reviewer

**Responsibility:** Reviews Spec Writer's PR before it merges to main.

- Reviews the Spec Writer's short-lived PR (same workflow as Code Review Panel reviews developer PRs)
- Evaluates spec clarity, completeness, and ZOMBIES coverage
- Checks Page Objects are well-abstracted and reusable
- Verifies specs test behaviour, not implementation details
- Ensures specs are deterministic and isolated
- Flags missing edge cases or boundary scenarios
- If Spec Quality Reviewer and Spec Writer deadlock → spawn Conflict Resolver

**Agent type:** `general-purpose`

---

### Developer Pair (XP Pair Programming)

**Responsibility:** Implements code using strict TDD (Red-Green-Refactor) to make Playwright specs pass. Developers always work in pairs following XP pair programming.

- **Driver:** Writes the code — types, runs tests, makes decisions at the line level
- **Navigator:** Reviews each line as it's written, thinks ahead about strategy, catches mistakes, suggests alternatives
- Roles swap frequently within a session
- Picks up stories from the task list
- **TDD discipline:** Write a failing test first, make it pass with minimal code, then refactor. No production code without a failing test.
- Starts with the Playwright acceptance specs (red — flip `test.fixme()` to active), then drives implementation through unit-level TDD cycles
- Follows DDD patterns from ARCHITECTURE.md (entities, value objects, services)
- Uses inline PEP 723 metadata — no pyproject.toml
- Syncs worktree with main frequently
- Submits work via short-lived PR — **Code Review Panel reviews pre-push on the PR**

**Agent type:** `general-purpose` (2 agents per pair, sharing a worktree)
**Mode:** `default`
**Isolation:** `worktree` (shared between the pair)

---

### Code Review Panel

A story is not done until it passes all five reviewers. Each reviewer guards its narrow niche and provides a pass/fail with actionable feedback. **Reviews happen on the PR before merge** — code does not land on main until all reviewers approve. If a reviewer and the developer pair deadlock after 2 rounds on the same issue → spawn Conflict Resolver.

#### General Reviewer

**Focus:** Correctness, readability, simplicity. Does the code do what the spec says?

- Checks the implementation matches the Playwright specs
- Flags unnecessary complexity or over-engineering
- Verifies no unrelated changes snuck in

**Agent type:** `general-purpose`

#### Test Quality Reviewer

**Focus:** Are the unit tests trustworthy? Do they follow good testing principles?

- Evaluates test isolation, determinism, and clarity
- Checks that tests actually assert meaningful behaviour
- Flags brittle tests, test duplication, or missing edge cases
- Verifies TDD was actually followed (tests aren't after-the-fact)

**Agent type:** `general-purpose`

#### DDD Reviewer

**Focus:** Does the code respect the domain model in ARCHITECTURE.md?

- Verifies ubiquitous language is used consistently in code
- Checks entity/value object/service boundaries
- Flags domain logic leaking into infrastructure or vice versa

**Agent type:** `general-purpose`

#### Code Smell Reviewer

**Focus:** Structural quality and maintainability.

- Scans for code smells (long methods, feature envy, primitive obsession, etc.)
- Checks SOLID/GRASP principle adherence
- Suggests specific refactorings when needed

**Agent type:** `general-purpose`

#### Pessimist

**Focus:** What can go wrong? Finds failure modes, security holes, and unhandled edge cases.

- Asks "what if this fails?" for every external call (DB, LLM, network, Playwright)
- Looks for missing error handling, race conditions, resource leaks
- Checks for security issues (injection, path traversal, unvalidated input)
- Questions assumptions — what if the feed is malformed? What if the LLM returns garbage?
- Identifies missing timeouts, retries, and graceful degradation

**Agent type:** `general-purpose`

---

### Conflict Resolver

**Responsibility:** Breaks deadlocks when any agents disagree or when review cycles stall.

- Spawned on-demand as an independent sub-agent when:
  - Reviewers disagree with each other on a PR
  - A reviewer and developer pair deadlock after 2 rounds on the same issue
  - Spec Quality Reviewer and Spec Writer deadlock
  - Any other role conflict arises
- Receives the conflicting positions from both sides
- Makes a binding decision — what it chooses wins
- Dissolved after the decision is made

**Agent type:** `general-purpose` (spawned ad-hoc, not a standing team member)

---

### QA Pool (1..N, scales with demand)

**Responsibility:** Uses the app from the user's perspective via Playwright to find bugs. Reports defects to PM.

- PM spawns/despawns QA agents based on how many stories need validation
- Each QA agent runs its own local server on a free port in its own worktree (DB is per-worktree, not shared)
- Launches the app and exercises it through Playwright browser automation
- Takes Playwright screenshots to document visual state and verify UI
- Performs exploratory testing — tries unexpected inputs, weird flows, edge cases
- Verifies completed stories actually work in the running application
- Files defect reports to PM as new tasks with reproduction steps and screenshots
- Re-tests after fixes are applied

**Agent type:** `general-purpose`
**Mode:** `default`
**Isolation:** `worktree`

**Scaling policy:** PM spawns additional QA agents when multiple stories reach validation simultaneously. QA agents are shut down when the validation queue is empty.

---

## Workflow

```
Phase 1: Preparation (the duo)
  1. PM + Designer collaborate on the next workstream
  2. PM validates technical approach and testability
     — if it can't be fully e2e tested, redesign it
     — LLM-dependent features use evaluations (structural/contract tests)
  3. Designer creates HTML/CSS mockups in design/mockups/,
     verifies with Playwright screenshots, merges via short-lived PR
  4. PM creates stories with references to relevant mockups

Phase 2: Specification
  5. PM + Spec Writer pair on example mapping for each story
  6. Spec Writer orders scenarios using ZOMBIES
  7. Spec Writer creates Playwright specs with Page Objects (in worktree)
  8. Specs tagged with story ID (e.g., // story: feed-crud)
  9. Spec Writer merges failing specs via short-lived PR (marked test.fixme())
 10. Spec Quality Reviewer reviews specs → feedback loop until approved

Phase 3: Implementation (TDD, XP Pair Programming)
 11. Dev pair syncs worktree with main (gets specs + mockups)
 12. Dev pair picks up story, activates tagged specs (removes test.fixme())
 13. Dev pair TDD inner loop (driver writes, navigator reviews):
     a. Write failing unit test
     b. Write minimal code to pass
     c. Refactor
     d. Swap driver/navigator roles frequently
     e. Repeat until acceptance spec is green
 14. Dev pair opens short-lived PR

Phase 4: Review (pre-push, on the PR)
 15. Code Review Panel reviews PR (5 reviewers in parallel)
 16. Dev pair addresses feedback on PR if needed → re-review
 17. If deadlock after 2 rounds → spawn Conflict Resolver, its decision wins
 18. All reviewers approve → PR merges to main

Phase 5: Validation
 19. PM scales QA pool based on stories awaiting validation
 20. Each QA agent spins up local server (free port) in own worktree
     (per-worktree DB, not shared)
 21. QA uses Playwright to test from user POV, takes screenshots
 22. QA reports defects to PM with reproduction steps + screenshots
 23. PM accepts story
```

---

## Development Practices

### Trunk-Based Development with Short-Lived PRs

- **Single trunk:** `main` is the trunk. No long-lived feature branches.
- **Short-lived PRs:** All agents work in worktrees on short-lived branches. PRs are opened, reviewed, and merged quickly. PRs should live hours, not days.
- **Failing specs are expected:** Spec Writer merges specs marked `test.fixme()` via PR. CI treats these as expected failures. Developers flip them to active as they implement.
- **Small commits:** Prefer many small commits over few large ones. Each merge should leave main in a working state (existing tests pass, new tests are marked fixme).
- **Per-worktree DB:** Each worktree has its own SQLite database. No shared state between agents.

### Continuous Integration

- All PRs trigger CI: run all active Playwright specs and pytest unit tests.
- `test.fixme()` specs are skipped in CI (expected failures for unimplemented stories).
- CI must pass before PR can merge.
- Database schema changes use `CREATE TABLE IF NOT EXISTS` and `ALTER TABLE` migrations in `db.py:init_db()`.

### LLM Testability (Evaluations)

LLM-dependent features are tested using evaluations, not exact-match assertions:

- **Structural assertions:** Output contains expected sections, is valid markdown, has expected length range
- **Contract tests:** LLM provider returns correct types, handles errors gracefully
- **Mock providers:** Unit tests use a deterministic mock `LLMProvider` that returns canned responses
- **Evaluation rubrics:** For integration tests, assert properties (e.g., "digest mentions at least 3 posts", "summary is under 500 words") rather than exact content

### Visual Verification

- Designer and QA use Playwright screenshots to verify rendered HTML/CSS
- Screenshots are taken via `page.screenshot()` and attached to defect reports
- Design system consistency is verified by comparing screenshots across pages

---

## Conventions

- **Backend:** Python 3.12+, inline PEP 723 script metadata
- **Acceptance tests:** Playwright specs in JS/TS with Page Object pattern
- **JS/TS dependencies:** Managed via `npm` with `package.json`
- **Unit tests:** `pytest` for Python unit tests
- **Development method:** Strict TDD — no production code without a failing test. XP pair programming (driver + navigator).
- **Testability rule:** Full e2e test coverage required. If it can't be tested, don't build it. LLM features use evaluations.
- **Spec ordering:** ZOMBIES (Zero, One, Many, Boundary, Interface, Exceptions, Simple)
- **Spec design:** Example mapping before writing executable specs
- **Spec tagging:** Each spec tagged with story ID via comment (e.g., `// story: feed-crud`)
- **Domain model:** Follow ARCHITECTURE.md ubiquitous language exactly
- **Design system:** Designer maintains shared HTML/CSS components in `design/mockups/`, verified via screenshots
- **Version control:** Trunk-based development with short-lived PRs
- **CI:** All active specs + unit tests run on every PR, must pass before merge
- **Database:** Per-worktree SQLite, not shared between agents
- **Backend dependencies:** Declared inline per-script, managed by `uv`
- **Frontend dependencies:** Managed via `npm`
- **Conflict resolution:** Spawn independent sub-agent, its decision is binding. Triggered after 2 rounds of deadlock or when reviewers disagree.
- **Local servers:** Each agent in a worktree runs its own server on a free port
- **Stories:** Include references to relevant mockups in `design/mockups/`
