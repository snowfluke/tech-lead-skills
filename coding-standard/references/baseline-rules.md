# Baseline rules

Propose these in step 2 unless the codebase already contradicts them. They are
stack-agnostic. Translate the examples into the project's language.

**File and function size**
- Cap files at **300 lines**; split proactively at **250**. A file that keeps growing is doing too many jobs. Rationale: small files are reviewable, testable, and navigable by humans and agents.
- The split strategy: when a file approaches the cap, extract cohesive pieces into a sibling submodule folder (e.g. `<name>-impl/` with one file per responsibility, or split a fat module into its endpoint/feature units) rather than dumping helpers into a shared `utils`. Keep the public entry point thin and re-export.
- Split a file that serves more than one feature, even under the cap.

**Complexity**
- Every function stays at cyclomatic complexity 10 or under **(enforced)**. Split an over-limit function by extracting a named function. Never suppress the rule. Rationale: a function with more than ten paths cannot be covered by a few tests.

**Tooling** (enforced by the linter, the formatter, and the hooks)

| Ecosystem | Lint | Format | Complexity rule |
| --- | --- | --- | --- |
| JS/TS | oxlint with the [anti-slop](https://github.com/dmmulroy/anti-slop) plugin | oxfmt | oxlint `complexity: ["error", { "max": 10 }]` |
| Go | golangci-lint | gofmt | the `cyclop` or `gocyclo` linter |
| Python | ruff | ruff format | ruff `C901` |
| Rust | clippy | rustfmt | clippy `cognitive_complexity` |

- Pre-commit hooks run the format check, the linter, and the type-checker on staged files. The aggregate check command runs before hand-off and in CI.
- Mark every rule a tool enforces as **(enforced)** and state the tool, so review time goes to what tools cannot catch.

**Type safety** (typed languages)
- No untyped escape hatches: no `any`, no unchecked casts (`as`), no non-null assertions (`!`). Use real narrowing and handle null/undefined explicitly. Rationale: each escape hatch is a place the type system stops protecting you.
- Explicit return types on exported functions. Prefer literal unions over loose enums where the language allows.
- Derive types from the validation schema (one source of truth) rather than declaring them twice.
- Shape data so that an illegal state cannot exist: a discriminated union or an enum, not a combination of booleans and optional fields. Rationale: a state the type cannot hold needs no check and no test.

**Structure and layering**
- Group files by feature: each feature folder holds its handler, service, repository, and tests. Group by layer only when the platform sets the structure, for example the OutSystems Architecture Canvas.
- Dependencies point one way: handler to service to repository. A lower layer never calls a higher one, and UI code never queries the data store.
- No business logic in route/handler/controller code; it belongs in a service layer.
- No direct data-store access outside a repository/data layer.
- Modules do not import each other's internals, and do not rely on another module's internal state, call order, or naming convention. They talk through a typed interface; cross-cutting needs go through a shared helper. Rationale: boundaries are what make a module extractable and testable later.

**Correctness hygiene**
- No magic numbers or strings; name them as constants.
- No dead code, no commented-out code; delete it (version control remembers).
- Each business rule lives in one place. Reuse a function only when it implements the same rule: code that only looks alike stays apart, because features that change for different reasons must not share a function. Before hand-off, check that one requirement change edits one place; if it edits many, fix the coupling or name it as debt in the PR.
- Validate input at the boundary (the handler, the form submit, the external call) and trust validated data inside it.
- Domain errors over raw throws; map them to the response envelope at the boundary. Never swallow an error into a bare log or hide it behind a fallback value such as `?? 0` or `catch { return [] }`.
- A wrapper translates data, enforces a policy, or isolates an external system. Delete a wrapper that only passes calls through.
- No config option, interface, or parameter that no current caller uses.
- Every multi-step write that must be atomic runs in a transaction; audit/side-effect writes share that transaction.
- A status that changes under business rules changes in one place: a transition function that checks the technical spec's transition table (the actor and the guard) and rejects every move the table does not list. Rationale: a status set by a plain assignment can reach a state no rule allows.

**Data access and performance**
- Select only the columns the code reads. Never load a full row or entity to read a few fields.
- Parse a URL once, where it enters the system, and pass the parsed fields on.
- Define regular expressions at module level, not inside functions or loops. Never call `test` or `exec` on a module-level regex with the `g` or `y` flag, because it keeps `lastIndex` between calls; use `matchAll` or `replace`, or create that regex inside the function.
- In PostgreSQL, a field that hot-path code reads, filters, or sorts on is a typed column. jsonb holds only data without a fixed schema, and stays small. Rationale: PostgreSQL stores large values out of line (TOAST) and must read and decompress the whole value to return one key.

**CSS** (projects with a web UI)
- Follow [Defensive CSS](https://defensivecss.dev): the layout holds when content is longer, shorter, or missing compared with the mockup. Flex rows that can overflow wrap; text-holding flex items get `min-width: 0` and grid tracks `minmax(0, 1fr)`; long words and URLs break or truncate; spacing comes from `gap` or margins, never `justify-content: space-between`; variable content gets `min-height`/`min-width`, not fixed sizes, and buttons a `min-width`.
- Images get `max-width: 100%`, and `object-fit: cover` at fixed dimensions; background images `no-repeat`; `var()` a fallback when the variable can be undefined; variable-count grids `auto-fill`.
- `overflow: auto`, not `scroll`, with `scrollbar-gutter: stable` where a scrollbar shifts the layout and `overscroll-behavior: contain` on scrollable panels. Sticky children of grid or flex get `align-self: start`; sticky or fixed positioning sits inside a `min-height` media query.
- Hover styles sit inside `@media (hover: hover)`. Inputs use `font-size` 16px or more, so iOS Safari does not zoom. Text on an image has a background or overlay. Vendor-prefixed selectors go in separate rules.

**No partial work in shipped code**
- No `// TODO` / `// FIXME` in merged code, except one explicitly sanctioned deferral pattern (e.g. a documented deferred-dependency marker) that the PR description lists. Rationale: a standard that tolerates open TODOs tolerates half-finished features.

**Configuration and secrets**
- No raw environment access scattered through the code; read config through one validated module. No secrets in the repo.

**Tests**
- Every test names one behaviour and fails when that behaviour breaks. Before a test is committed, break the implementation once (invert a condition, delete the branch) and watch the test fail.
- No tautological tests: no expected value produced by the code under test, no mock of the unit under test, no assertion inside a branch that may not run, no test that survives deleting the implementation body.
- No change-detector tests: no re-recorded snapshots, no assertions on internal calls or private structure. A test that fails while the behaviour is unchanged is rewritten or deleted.
- Arrange-Act-Assert. Mock only at the system edge (network, clock, third-party service), never the unit under test or the project's own modules.
- For a complex feature, the e2e tests cover realistic scenarios of medium or high complexity: several steps, real data shapes, and a failure or permission path. The simplest success case alone is not enough.
- For a bug fix, first check whether an existing behaviour test should have caught the bug. If so, fix that test. Add a new regression test only when no behaviour test covers the case. The regression test fails on the code before the fix, for the reported reason.
- One test owns each contract, at the strongest boundary that can see it. Another layer gets a test only for a failure the owner cannot reach. Extend a table case or a shared fixture instead of adding a near-duplicate test.
- No production seam for a test: no export, flag, wrapper, or injection hook that only a test calls. Test at the real boundary.
- A test does not match a junk pattern. A match stays only when it is the only guard of a public API, protocol, config, migration, storage, security, default, or release contract.
- State what must have a test: every error condition the service owns, and every acceptance criterion the card covers.

Junk patterns. A test that matches one is rejected in review:

1. It has no assertion.
2. It compares a value with itself or with a copy of itself.
3. It asserts a copied fixture, inventory, manifest, or export list against its source.
4. It greps for exact source text, an import, or a string. The exception: the cheapest guard of a user-facing key, byte, or path.
5. It tests a private predicate or a call shape that a boundary test already covers.
6. It repeats another test of the same contract.
7. It replays a shared helper's tests in a local copy.
8. It exists only to keep a test-only export, global, or wrapper alive.
9. It covers production code that only tests call. Delete the code and the test.
10. Its expected value comes from the helper or renderer under test.
11. Its mock implements the behaviour the test asserts, or one mock stands in for several different APIs.
12. Its fixture supplies what the code under test must produce, such as a receipt, an ordering, or a callback. Or it checks a store the code path never writes.
13. It restates a declared capability flag instead of exercising what the flag promises.
14. It is a negative test that passes for an unrelated reason, such as a refusal from a different guard.
15. Its name or fixture promises more than the test exercises.

The junk patterns are adapted from OpenClaw's test-audit skill (MIT, Copyright (c) 2026 OpenClaw Foundation).
- Local e2e runs stay light: capped workers, one headless browser, the running dev server reused, only the affected specs, heavy scenarios tagged and left to CI, traces and video only on failure. The suite stops every process it started.

**Comments**
- A comment states why the code exists. It never narrates what the code does or what a change did.

**Commits, branches, and pull requests**
- Conventional Commits, imperative mood, the card ID in the subject or body.
- Branch names carry the card: `<type>/<card-id>-<slug>`, for example `feat/BE-S2-05-csv-export`. Work from an issue with no card uses the issue number: `fix/123-login-timeout`.
- ASCII only in commit messages, PR titles, and PR bodies: no em or en dashes, no emoji, no arrows. Use `-`, `:`, and `->`. Source files may hold any language.
- No attribution: no `Co-Authored-By` trailer, no "Generated with" line.
- The PR body fills every section of the PR template.
- A `BE` or `FE` card's PR changes a test. A wiring card's PR changes an e2e test. A `fix` or `feat` issue's PR changes a test. A `TL` or `DB` chore may change none.
- A test that proves an AC names the AC ID in its title. Every AC ID in a PR body appears in a changed test.
- A PR is done when it opens. No to-do tags, skipped tests, or focused tests in the diff. Left-over work gets its own card or issue, and the PR names it.
- Language policy: code, commit messages, and PR text in English. UI copy and domain terms in the language of the business docs, verbatim.

These rules are enforced **(enforced)**. `10-comments-commits-and-docs.md` carries them in this block, which the hooks and CI read. Adjust the values to the project; keep every key.

```pr-hygiene
branch: ^(feat|fix|chore|docs|refactor|test)/(?P<kind>BE|FE|TL|DB)-S\d+-\d+-[a-z0-9-]+$
branch: ^(?P<kind>feat|fix|chore|docs|refactor|test)/\d+-[a-z0-9-]+$
commit-subject: ^(feat|fix|chore|docs|refactor|test|perf)(\([a-z0-9-]+\))?: \S.{0,70}$
forbidden: Co-Authored-By
forbidden: Generated with
forbidden: Generated by
pr-heading: ## Summary
pr-heading: ## Card
pr-heading: ## Tests
pr-heading: ## Checklist
test-files: (\.test\.|\.spec\.|/tests?/)
e2e-files: ^e2e/
tests-required: BE FE feat fix
ac-id: \bAC-\d+(?:\.\d+)*\b
```
