# Engineering Doctrine

This document defines **non-negotiable engineering rules** for all work planned and performed in this project.

* These rules are **defaults** and should be treated as **hard constraints** unless explicitly waived in writing.
* If a requirement conflicts with a user/business requirement, the conflict must be raised early and a resolution (or waiver) must be recorded.

---

## 0) Timeless End-State Rule (meta)

All names, headings, identifiers, comments, and documentation should describe the **intended end state** only.

Rules:

* Do not label anything as “new”, “old”, “legacy”, “temporary”, “v1/v2”, “phase1/phase2”, “step1/step2”, “rewrite”, “refactor”, “migration”, or similar process/chronology terms in identifiers or headings.
* Do not encode *how or when* something was created into names or structure.
* Prefer semantic naming that describes purpose and role in the final design (e.g., “public”, “internal”, “experimental”, “deprecated”) only when that distinction is truly part of the intended end state.

Notes:

* It is acceptable for dedicated logs (decision/change logs) to describe sequencing and evolution, but primary naming and structure must remain end-state oriented.

---

## 1) Architecture Doctrine: Custom N-Tier Organization

### 1.1 Layer definitions

This project follows a strict, custom N-tier architecture. Every module belongs to exactly one layer:

#### Service Layer

**Purpose:** External abstraction and ingress/egress translation.
**Responsibilities:**

* Accepts external inputs (e.g., events, requests, messages).
* Translates external formats into **internal types owned by the next layer**.
* Routes to the appropriate logic entrypoint.

**Constraints:**

* Service layer **ONLY calls the logic layer**.
* Service layer must not contain business decisions beyond routing and basic validation.

#### Logic Layer

**Purpose:** Business decision-making.
**Responsibilities:**

* Encodes the business rules and workflows.
* Decides **when** to retrieve/update/publish data.
* Orchestrates the sequence of operations.
* Emits domain-level outcomes.

**Constraints:**

* Logic layer **ONLY calls the data layer**.
* Logic layer must not know **how** data is retrieved/updated/published (no dependency details).
* Logic layer must not call external systems directly.

#### Data Layer

**Purpose:** Business data operations and shaping.
**Responsibilities:**

* Handles requests from logic layer to get/update/publish business data.
* Calls one or more dependencies to fulfill each request.
* Maps and formats dependency responses into **types owned by this layer**.
* Aggregates or composes multiple dependency calls if needed.

**Constraints:**

* Data layer **ONLY calls the dependency layer**.
* Data layer must not contain business “when/why” decisions—only data shaping and fulfillment.

#### Dependency Layer

**Purpose:** Downstream service abstraction.
**Responsibilities:**

* Encapsulates all downstream integration logic (e.g., databases, APIs, filesystems, sockets).
* Owns dependency-specific request/response formats, errors, retries, and pagination mechanics.
* Exposes narrow, stable interfaces to the data layer.

**Constraints:**

* Dependency layer must not depend on upper layers.
* Dependency layer must not contain business logic.

---

### 1.2 Directional call graph (strict)

**Each layer may ONLY call the layer immediately below it.**

Allowed:

* Service → Logic → Data → Dependency

Forbidden:

* Calling the same layer (e.g., Data → Data)
* Calling above (e.g., Data → Logic)
* Skipping layers (e.g., Logic → Dependency)
* Cross-layer sideways calls (e.g., Service → Data)

**Goal:** minimize blast radius of future changes. Replacing a dependency should primarily impact the dependency layer, with minimal ripple into higher layers.

---

## 1.3 Type boundaries and ownership (strict)

### 1.3.1 No “type teleportation” (no multi-layer jumps)

**Types must not be passed across more than one layer boundary.**

Meaning:

* Service must not accept/return dependency-layer types.
* Logic must not accept/return dependency-layer types.
* Service must not accept/return data-layer types.
* Any type that crosses a boundary must be owned by the receiving boundary layer.

If a type crosses more than one layer, the middle layer becomes an empty pass-through and the abstraction collapses.

### 1.3.2 Each layer owns its boundary types

Interfaces are defined by:

* public methods/functions
* their input types
* their output/result types
* their error types

Therefore **each layer must own the types that define its interface**.

Recommended structure (adapt as needed per language):

* `/src/service/types/`
* `/src/logic/types/`
* `/src/data/types/`
* `/src/dependency/types/`

Rules:

* A layer’s public API must use **its own layer types** (for inputs/outputs/errors).
* When calling down one layer, translate from the caller’s types into the callee’s types.
* When returning up one layer, translate back into the current layer’s types.

### 1.3.3 Mapping is mandatory at every boundary

Boundary calls must do explicit translation:

* Service ⇄ Logic
* Logic ⇄ Data
* Data ⇄ Dependency

No “just reuse that type” shortcuts.

---

## 1.4 Error boundaries and translation (strict)

### 1.4.1 Errors are part of the interface

Errors behave like types: passing an error upward without translating is equivalent to leaking the lower layer’s interface.

### 1.4.2 Error type must not jump layers

* A lower layer’s error type must not escape upward beyond the adjacent layer boundary.
* Each boundary must translate errors into the current layer’s error type.

### 1.4.3 Error message may propagate; typed error details may not

When translating an error upward:

* You MAY carry forward a human-readable message (and optionally a safe “cause chain” as strings).
* You MUST NOT pass the lower layer’s typed error as-is across the boundary.
* Upward layers should receive only the current layer’s error type.

Practical guidance:

* Preserve what matters to the upper layer: category, retryability, user-impact, and a readable message.
* Hide what does not matter: dependency-specific codes/shapes/pagination mechanics.

### 1.4.4 Layer-local handling

Each layer should handle errors “as it matters”:

* Dependency layer: dependency mechanics (timeouts, retries, backoff, paging failures)
* Data layer: normalize + shape into domain-relevant errors
* Logic layer: decide business outcomes (retry? compensate? fail?)
* Service layer: format into external response/event semantics

---

## 1.5 Minimal interfaces and information hiding

Interfaces between layers must be:

* **small** (minimal surface area),
* **stable** (don’t leak underlying mechanism),
* **expressive** (represent intent, not dependency details).

Upper layers must not be forced to understand downstream mechanics such as:

* pagination strategies,
* single vs batch retrieval differences,
* raw dependency error shapes,
* dependency response schemas,
* connection/session management.

**Example requirement**
If a downstream system supports:

* single get
* paginated batch get

Then the dependency layer should expose **one** method (e.g., `batch_get`) that:

* supports both single and batch usage,
* internally handles pagination/iteration,
* returns a unified result type.

---

### 1.6 Architecture enforcement checklist (must pass)

Before work is considered “done”, confirm:

* [ ] Every file/module is assigned to exactly one layer
* [ ] Dependencies flow strictly downward (no same-layer, no upward, no skipping)
* [ ] All boundary calls perform explicit type translation (no multi-layer type jumps)
* [ ] All boundary calls perform explicit error translation (no multi-layer error jumps)
* [ ] Interfaces do not leak downstream mechanics
* [ ] Data shaping occurs in the data layer, not logic
* [ ] External translation occurs in the service layer, not logic/data

---

## 2) Code Quality Doctrine

### 2.1 Functional, stateless, side-effect-minimizing style

Prefer code that is:

* **stateless** (avoid shared mutable state),
* **functional** (compose transformations),
* **side-effect-minimizing** (return values instead of mutating external state).

If side effects are required (e.g., persistence, publishing):

* isolate them as low as possible (often dependency layer),
* keep boundaries explicit and deterministic given explicit inputs.

### 2.2 Idempotency and determinism

* Code should be **idempotent whenever possible**.
* Methods should be **deterministic** (same input → same output), unless explicitly documented otherwise.

If nondeterminism is required (time, randomness, external state):

* inject it as an input dependency,
* record the rationale.

### 2.3 Immutability by default

* Variables should be immutable by default.
* Mutability is allowed only as an edge-case optimization or when required by the language/runtime.
* If mutability is used, keep it tightly scoped.

### 2.4 Control flow clarity

* Avoid nested conditions and `else` chains.
* Prefer **guard clauses / early exits**.
* Prefer clear linear flows over deep branching.

### 2.5 No magic numbers or strings

* No unexplained literals in logic.
* Use named constants.
* Include units in names where not enforced by types (e.g., `TIMEOUT_PERIOD_S`).

### 2.6 Naming and typing requirements

* Variables must be clearly named.
* Avoid nullable parameters whenever possible.
* Always use typed variables (avoid “any”-like types; prefer explicit domain types).

### 2.7 Performance-first mindset

Write code to be performant from the beginning:

* avoid unnecessary heap allocations,
* avoid expensive calls in hot paths,
* prefer predictable, efficient data structures.

Performance decisions must not compromise correctness or clarity. If a tradeoff is necessary, document it via the comments policy below.

---

## 2.8 Comments policy (strict)

**Default:** no comments.

Comments are ONLY allowed when a doctrine rule must be broken or a non-obvious constraint cannot be expressed cleanly in code.

### Comment content rules

When you write a comment:

* keep it **short** (1–2 lines preferred),
* explicitly state what the code is doing “wrong” relative to the normal style,
* explain **why** it is necessary,
* and only if needed, briefly explain **what** it does.

**Do not reference this doctrine explicitly in the comment.**
Instead, phrase it as a local exception explanation.

### Comment format (recommended)

Use one of these patterns:

* `# Mutating <thing> because <reason>.`
* `# Using <non-ideal approach> because <reason>.`
* `# Allowing <exception> to avoid <worse outcome>.`

### Allowed comment categories (only as exceptions)

* Non-obvious invariants that cannot be encoded cleanly
* Performance caveats/constraints forcing a non-idiomatic approach
* Safety/footgun warnings where misuse is likely
* Local exception notes for unavoidable complexity

---

## 2.9 Interface minimalism & compatibility policy (strict)

This section exists to prevent:

* excessive parameters in CLIs/APIs/interactions,
* unnecessary backwards compatibility burdens,
* interface bloat that reduces clarity and increases maintenance cost.

### 2.9.1 Minimal parameter rule (strict)

Any public interaction surface (CLI args, API request fields, event payload fields, public method parameters) must be **minimal**.

Rules:

* Do not add a parameter unless it is required to meet an explicit acceptance criterion.
* Prefer **one configuration object / request type** over long parameter lists.
* Prefer deriving values from existing context over new knobs.
* Prefer sensible defaults over mandatory flags.

When proposing new parameters, you MUST justify:

* What user-visible capability it enables
* Why existing inputs cannot express it
* Why it must be configurable (vs default)

### 2.9.2 No speculative flexibility

Do not add parameters “for future use” or “just in case”.
Future needs must be introduced when they are real, not predicted.

### 2.9.3 Backwards compatibility is not automatic

Do not preserve backwards compatibility unless there is a concrete need.

Compatibility must be explicitly chosen based on:

* whether the interface is already depended on (internal/external),
* cost/risk of breaking change,
* migration feasibility,
* rollout requirements.

If compatibility is required:

* prefer a short, explicit migration plan,
* keep both behaviors for a limited period if necessary,
* remove the old path once migration completes.

If compatibility is NOT required:

* change the interface cleanly and remove the old behavior immediately.

### 2.9.4 Compatibility decision record (required)

Every time an interface changes, record a short decision note (in planning docs) stating:

* Is backwards compatibility required? (Yes/No)
* Who depends on it?
* What is the migration plan (if any)?
* What is the removal plan (if any)?

### 2.9.5 Interface review checklist (must pass)

For any change that adds/modifies an interaction surface:

* [ ] Is every parameter necessary for an acceptance criterion?
* [ ] Are there any parameters added “for future use”? (must be removed)
* [ ] Are defaults used instead of additional knobs?
* [ ] Is the interface expressed as a minimal typed request object where appropriate?
* [ ] Was backwards compatibility explicitly decided and documented?
* [ ] If compatibility is kept, is there a defined removal plan?

---

## 3) Testing Doctrine

Testing strategy is explicitly tiered. Choose the tier based on what you are validating.

### 3.1 Unit tests (file-level behavior)

**Goal:** validate a single file/module’s behavior.

Rules:

* Unit tests should test the **file/module interface** and behavior (inputs → outputs).
* **Mock everything that leaves the file/module** (any external calls, IO, network, DB, time).
* Keep tests deterministic and fast.
* Tests should remain resilient to refactors.

Notes:

* If you cannot test a behavior without calling non-public internals, restructure so the behavior is reachable via a public module API (do not test private helpers directly).

### 3.2 Local E2E tests (system-level with external mocks)

**Goal:** validate end-to-end behavior of the system while controlling external dependencies.

Rules:

* Exercise the system through its normal public entrypoints.
* Mock **service-external systems** (e.g., filesystem, external APIs, databases) so tests are repeatable and can run locally.
* Validate “real wiring” between internal layers while keeping external dependencies stable.

### 3.3 Integration tests (no mocks)

**Goal:** validate real integration with real dependencies.

Rules:

* No mocks.
* Run the system “as is” and interact with it like any other client/system would.
* Focus on catching contract mismatches, environment issues, and real dependency behavior.

### 3.4 General testing rules (apply to all tiers)

* Tests should avoid coupling with implementation details.
* Tests should assert explicit inputs and explicit outputs.
* Tests should ONLY call public methods/APIs of the relevant scope (module/system).
* Tests must provide high confidence.
* Every time a bug is found:

  * investigate why tests didn’t catch it,
  * add/improve tests to prevent recurrence,
  * update doctrine/patterns if there is a systemic gap.

---

## 4) Linting, static analysis, and build/compile-time quality gates (strict)

We lean toward being as strict as possible. Code quality issues should be caught:

* as early as possible,
* automatically,
* and preferably at build/compile time.

### 4.1 Default posture: strict by default

Rules:

* Enable the strictest reasonable linting/static analysis for the language/tooling used.
* Prefer configurations that treat warnings as errors in gated checks.
* Prefer configurations that fail fast and produce actionable messages.

### 4.2 No blanket suppression

Disallowed:

* global “disable all” configurations,
* broad file-level ignore directives,
* wildcard ignores.

Allowed only as an exception:

* a narrow suppression for a specific rule on a specific line/symbol,
* with a short exception comment (per the comments policy) stating what is being done “wrong” and why.

### 4.3 Lint rule selection philosophy

Prefer rules that enforce:

* correctness (unused/unsafe constructs, unreachable code, invalid states),
* determinism/idempotency patterns where possible,
* type safety and explicitness,
* interface minimalism (where supported),
* clarity (naming, dead code, complexity),
* performance footguns (allocations, expensive operations) where available.

### 4.4 Gatekeeping requirements

Any planned implementation must define build/compile-time quality gates, typically including:

* formatting checks (no drift)
* lint/static analysis checks
* compile/build checks
* test execution checks (appropriate to the change)

These gates must be:

* deterministic,
* fast enough to run routinely,
* required to pass before merging.

### 4.5 “Fix, don’t waive” rule

If a lint/gate fails:

* default action is to refactor/fix code to satisfy the rule,
* only waive if the rule is truly incompatible with a required behavior/performance constraint.

Any waiver must:

* be narrow (smallest possible scope),
* be explained via a short local exception comment (“doing X wrong because Y”),
* be recorded as a waiver in planning documentation.

### 4.6 Complexity caps (recommended)

To prevent runaway complexity:

* prefer code that stays under reasonable complexity thresholds (branching, nesting, cyclomatic complexity) where enforceable.
* if a complex construct is necessary, isolate it and justify with a short exception comment.

---

## 5) Waivers (exceptions process)

If a doctrine rule must be broken:

* Record a waiver with:

  * what rule is being violated,
  * scope (where),
  * rationale,
  * risk,
  * mitigation,
  * rollback plan (if applicable).

Waivers must be explicit and discoverable (e.g., in the planning log for the task).
