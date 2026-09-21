# Lenses

A lens is a mutation operator: it must hunt a bug class the other lenses
do not. Each reviewer gets exactly one lens text below, verbatim, plus its
matrix cells.

## Universal (always on)

### spec

Take each acceptance criterion from `ticket.md` in turn. For each, name
where in the diff it is satisfied and where it is tested. Report a finding
for every criterion with no implementation, no test, or a partial
implementation. No ticket: report a single low finding saying the scope
could not be checked, and stop.

### control-and-error

Walk every branch, early return, catch, retry and default in the diff.
Hunt: empty or swallowing catch blocks, rejected promises nobody awaits,
error state set and never cleared, retries without a bound, defaults that
hide a missing value.

### state-transitions

List the states the diff reads or writes and the events that move between
them. For each ordering the code assumes but nothing enforces, produce the
scenario that violates it: event arrives twice, arrives before
initialisation, never arrives, arrives after teardown.

### callers-and-siblings

For every function, component, action or selector changed, find every call
site and every place that does the same job and was not changed. Hunt:
callers broken by the new contract, sibling code paths with the same bug
the PR fixes in only one place, dead callers left behind.

### tests-as-spec

Read only the tests in the diff and in `tests.md`. For each, say what
behaviour it pins versus merely executes. Report every assertion that would
still pass if the change were reverted, every branch in the diff with no
test reaching it, and every fixture whose shape differs from the real
runtime data.

### diff-hygiene

Hunt in the diff only: leftover debug output, dead code, unrelated
reformatting, renamed symbols with stale references in strings, docs or
config, changelog or docs not updated for a user-visible change, TODOs
standing in for work.

### prior-review

Read `comments.md` and `related.md` only. For every review comment, past
or present, check it is addressed in the current diff and not regressed.
Report each unresolved or regressed comment with the thread it came from.

## Conditional (fallback triggers when no profile)

Trigger is file extension or an import in the diff. A profile replaces this
table with repo-specific globs and prompts.

### concurrency

Trigger: `async`, `await`, `Promise`, generators, threads, goroutines,
`yield`, effects libraries. Hunt: two events racing on shared state,
cancellation not propagated, cleanup not run on unmount or exit, stale
closures over mutable values, latest-wins vs every-wins mismatches.

### data-shape

Trigger: reducers, serializers, DTOs, API clients, transforms, migrations.
Hunt: optional field treated as present, case or naming boundary crossed
without transform, mutable vs immutable mix, fixtures not matching the
production shape, nullable columns.

### ui-contract

Trigger: component or view files, stylesheets, templates. Hunt: every prop
or state combination that renders, missing role or label, keyboard path
missing, measured dimensions vs the design source, both light and dark or
both product modes when they exist.

### http-boundary

Trigger: route handlers, controllers, middleware, views. Hunt: new route
without auth, input reaching a query or a shell unchecked, CSRF on
state-changing routes, response leaking internals, status codes not
matching the outcome.

### config-and-deploy

Trigger: env templates, config files, CI, Dockerfiles, task definitions,
feature flags. Hunt: key added in one environment and not the others,
flag default that turns the feature on, order of deploy vs migration, old
clients during rollout.

### performance

Trigger: loops over collections in render paths, selectors, ORM queries.
Hunt: new reference returned on every call, N+1, work done per render that
could be memoised, unbounded lists.

### security

Trigger: auth, token, secret, crypto, session, permission in paths or diff.
Hunt: trust boundary crossed without validation, secret in logs or client
bundle, permission checked on the client only, token lifetime or scope
widened. Only on trigger; otherwise it dilutes the other lenses.

## Cross (set mode only)

Each runs once over every diff in the set plus `set.md`. They never run on
a single PR.

### contract-drift

Walk the contract map. For every route, payload field, event, config key
or schema element a PR changes on the producing side, find every consumer
in the other PRs and in the unchanged code of their repositories. Hunt:
renamed on one side only, type or nullability changed, field removed
while still read, field added and never read, default that differs
between sides. Cite both sides by repo and file:line.

### rollout-order

For every pair of repositories that talk, derive which must deploy first
from the contract map, then produce the scenario for the window between:
old consumer on new producer and new consumer on old producer. Hunt:
required field the old side does not send, route the old side does not
serve, version pin that the other repository has not yet published. State
the safe order explicitly and every pair where no order is safe.

### config-propagation

Every environment variable, config key, secret name, image tag or version
added, renamed or removed anywhere in the set: list where it is set and
where it is read across all repositories. Hunt: set and never read, read
and never set, set under one name and read under another, present in one
environment template and missing in the others. Follow the chain end to
end; a break anywhere is a finding.
