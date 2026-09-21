---
name: review-ensemble
description: >
  Reviews a pull request with independent fresh-context reviewers, each on a distinct lens, dedups their claims, ranks them by risk for the operator to choose from, then verifies each chosen claim in a judge-run court where a prosecutor must produce valid evidence and a defender must cite the guard. Use when a single AI review is not trusted to be complete, when a PR is high-risk, or when the user asks for an ensemble, multi-pass, or "thorough" review.
metadata:
  origin: custom
---

# Review Ensemble

One AI review is one sample from a distribution. This skill takes many
samples with forced diversity, votes, lets the operator choose what is
worth verifying, verifies in court, and reports. The orchestrator never
reviews code itself: it is one sample too, and its only job is structure.

Invariants. Reviewers never see each other. Nobody in the court sees how
many reviewers raised a claim. The defender never sees the prosecutor's
work. Break any of these and the sampling means nothing.

## When To Activate

- User asks for an ensemble, multi-agent, multi-pass or thorough review of a PR
- User says a previous AI review missed things and wants a stronger pass
- PR touches concurrency, state machines, auth, or money, and the user asks
  for a review

## Inputs

- **PR number** (optional): detect from the current branch if absent.
- **PR URLs** (optional, two or more): set mode. The PRs may live in
  different repositories and may already be merged. See "Set mode".
- **lenses=** (optional): explicit list; explicit always wins over the profile.
- **N=** (optional): reviewers per lens, default 1. Raise for high-risk PRs;
  support count then becomes meaningful within a lens too.
- **parallel=** (optional): agents run at once, default 1. Sequential keeps
  token usage per unit of time flat; raise it only with budget to burn.
- **tier=high** (optional): judge and deep reviewers move to the `max`
  tier. Off by default; the strongest models are costly.

## Models

Roles name a tier, never a model. One table maps tiers to model names per
runtime; resolve through the column of the runtime you are running in. If
your runtime cannot set a model per subagent, run the role at the session's
model and say so in the report's metrics.

| tier       | meaning                                             | claude code | codex           | vibe           |
|------------|-----------------------------------------------------|-------------|-----------------|----------------|
| `standard` | reads and tallies, light judgement                  | sonnet      | gpt-5.6-luna    | mistral-small  |
| `strong`   | traces code, writes and runs tests, weighs evidence | opus        | gpt-5.6-terra   | mistral-medium |
| `max`      | strongest available; only with `tier=high`          | fable       | gpt-6-astra     | mistral-large  |

Column sources: Claude Code from the Agent tool's `model` values; Codex
from Codex CLI 0.155.1's own answer; Vibe from Mistral Vibe's own answer.
Re-ask when a runtime is upgraded.

| role                                                        | tier       | with `tier=high` |
|-------------------------------------------------------------|------------|------------------|
| orchestrator (the session itself)                           | `standard` | `standard`       |
| reviewers: spec, diff-hygiene, prior-review                 | `standard` | `standard`       |
| reviewers: control-and-error, state-transitions, callers-and-siblings, tests-as-spec, every conditional lens | `strong` | `max` |
| triage                                                      | `standard` | `standard`       |
| judge                                                       | `strong`   | `max`            |
| prosecutor                                                  | `strong`   | `strong`         |
| defender                                                    | `strong`   | `strong`         |

The `max` tier is never used unless the invocation says `tier=high`. A
profile may override any tier with a vendor name under `models:`; the repo
knows which runtime its team uses.

### Runtime notes

- **Claude Code.** Agent tool, `model` per call, fresh context by using a
  non-fork agent type. Concurrency is how many Agent calls go in one
  message; `parallel=1` means one per message.
- **Codex.** `spawn_agent({ model, fork_context: false, message })`.
  Concurrency via `[agents] max_concurrent_threads_per_session` in
  `config.toml`; set it to `parallel=`. Worktrees via `--worktree`.
- **Vibe.** `task` tool, no model parameter: every role runs at the session
  model, and the metrics must say so. `task` is synchronous, so
  `parallel=` above 1 has no effect. Subagents are read-only, which the
  court already accommodates: the prosecutor never writes files.

## Set mode

Several PRs that implement one change across repositories. Each PR is
reviewed on its own exactly as below, and the set as a whole gets three
cross lenses that no single-repo review can run. Differences from a single
PR:

- **Gate.** Skip the mergeable check; merged PRs are fine. Fetch every
  repository into its own scratch worktree at the PR head, base branch
  alongside. Each repository's own profile applies to its local lenses.
- **Pack.** One `context/<repo>/` per PR as in step 2, plus one
  `context/set.md`: the ticket, every PR with its role in one line, and
  the **contract map**, extracted mechanically from every diff: HTTP
  routes with request and response fields, message and event shapes,
  environment and config keys read or written, package and image
  versions pinned, database fields. For each entry: which PR produces
  it, which PR consumes it.
- **Fan-out.** Local lenses run per repository. The cross lenses in
  `lenses.md` run once each over every diff plus the contract map.
- **Dedup, triage, report.** Every claim carries a `repo` field; the
  triage table and the report show it. A cross claim lists every repo
  it touches.
- **Court.** The judge gets one worktree per repository in the set. A
  cross claim usually closes on a trace; the prosecutor tries a
  consumer-side test first, feeding the consumer the producer's new
  shape, and falls back to a trace across the repositories.
- **Cost.** Roughly one single-PR run per repository plus three cross
  reviewers. When the PRs were already reviewed individually, invoke
  with `lenses=contract-drift,rollout-order,config-propagation` to run
  only what nobody ran.

## Repo profile

Generic core plus a per-repo profile. The core contains no path globs.
The profile lives at `.claude/review/profile.yaml` in the reviewed repo:

```yaml
version: 1
oracle:
  test: "npm test -- --silent"          # command that counts as ground truth
  lint: "make lint"
runs: .claude/review/runs               # per-run reports and metrics
models:                                  # optional, vendor names, per tier
  strong: opus
context:                                 # extra sources for the context pack
  - kind: jira                           # jira | file | url | command
    key_from_branch: '^([A-Z]+-\d+)'     # regex on the branch name
  - kind: file
    path: CLAUDE.md
lenses:                                  # conditional lenses and their triggers
  - name: concurrency
    when:
      paths: ["**/*Sagas.js", "**/sagas.js"]
      diff:  ["takeLatest", "takeEvery", "yield call("]
    prompt: |
      ...
```

No profile: use language-level fallback triggers from `lenses.md`, and emit
a suggested `profile.yaml` at the end of the run as a side output. Never
write it into the repo without asking.

## Instructions

Open with one short line saying you are using the review-ensemble skill.
Work in the scratchpad; write into the repo only what the profile names.
Spawn agents with the Agent tool as fresh general-purpose agents, never
forks: a fork inherits this conversation and correlates the samples.
Respect `parallel=` everywhere agents are spawned.

### 1. Gate

```bash
/opt/homebrew/bin/gh pr view --json number,title,baseRefName,headRefName,mergeable,additions,deletions,files
/opt/homebrew/bin/gh pr diff <number> --name-only
```

Decide:

- **Set.** Two or more PR URLs: set mode, see above. Everything below
  then runs per repository, with the additions that section lists.
- **Scope.** Over ~800 changed lines or more than one unrelated concern:
  split into sub-scopes by directory or concern and run the pipeline once
  per sub-scope. Say so in the report.
- **Lenses.** All seven universal lenses from `lenses.md`, plus every
  conditional lens whose trigger matches (profile first, fallback table if
  no profile). The gate may add at most one extra lens, with a one-line
  reason that goes into the report. Explicit user lenses replace all of
  this.
- **Cap.** More than ten lenses: drop conditional ones with the weakest
  trigger match, report which were dropped.

### 2. Context pack

Build once, read-only, in the scratchpad as `context/`. Every reviewer reads
the same pack and nothing else is shared.

- `diff.patch`: full PR diff.
- `files/`: full current content of every changed file.
- `ticket.md`: ticket summary and acceptance criteria as a numbered list,
  from the profile's `context` sources. Absent: say "no ticket" in the pack.
- `related.md`: linked and related PRs (same ticket prefix, same files in
  the last 90 days) with their review comments.
- `comments.md`: every review thread on this PR, resolved or not.
- `tests.md`: test files that import or reference any changed module, and
  the oracle command.
- `matrix.md`: hunk list (file, start, end, one-line summary) crossed with
  the chosen lenses. Every cell is owned by every reviewer of its lens.

### 3. Fan-out

One fresh agent per lens (times N), spawned according to `parallel=`.
Each prompt contains: the pack path, the lens text from `lenses.md` or the
profile, the matrix cells it owns, and the finding schema below. Reviewers
read files as they need but must not run tests or modify anything.

Finding schema, one JSON object per finding, in a `findings` array, plus a
`covered` array of cell ids with `found: true|false` for each cell it owns:

```json
{
  "file": "src/state/sessionSagas.js",
  "lines": [120, 138],
  "claim": "one sentence, the defect only",
  "scenario": "concrete inputs and order of events that produce the wrong outcome",
  "evidence": ["what was read, file:line"],
  "confidence": 0.7,
  "lens": "concurrency",
  "severity": "high|medium|low"
}
```

"Covered, nothing found" is a required output per cell. Silence must be
distinguishable from not looked.

### 4. Normalise and dedup

- Key on repo, file, overlapping line range and claim similarity.
- Cluster near-duplicates. Merge scenarios, keep the strongest evidence.
- `support` = number of distinct reviewers in the cluster.
- Record every cluster to `clusters.json` with its member reviewer ids.

### 5. Triage, then stop

One fresh mid-tier agent reads every cluster and assigns `risk_if_true`:
high, medium or low, with one line of reasoning. It does not verify and
it does not judge whether the claim is true; it rates the damage if it
were.

Print a numbered table sorted by risk, then support:

```
#  risk    support  repo      file:lines                        claim
1  high    3        service   src/state/sessionSagas.js:120-138  ...
2  high    1        facade    ...
```

Stop and ask the operator which numbers proceed to court. Accept ranges
(`1-8`), lists (`1,3,7`), `all`, or `none`. Claims not chosen go to the
report's "not verified" section with their triage row, so the choice is
visible later. Do not spend a single verification agent before the answer.

### 6. Court, one claim at a time

For each chosen cluster, in triage order, the orchestrator creates one
**judge** (fresh agent, strongest model) and hands it the claim, the
scenario, the relevant files, `tests.md`, the oracle command and a worktree
on the PR branch, one per repository in set mode. Nothing else: no support count, no confidence, no
reviewer evidence, no other cluster. The judge runs the court and returns a
verdict with artifacts. Courts run one at a time unless `parallel=` says
otherwise; inside a court, everything is sequential.

**Prosecution.** The judge summons a fresh prosecutor with the same inputs.
The prosecutor must return the body and path of a test that fails because
of the claimed defect, or, if no test can express it, a line-by-line trace
of inputs and events ending in the wrong outcome. The prosecutor returns
text; it never writes to the worktree. The judge writes the test file and
runs it with the oracle command.

**Test validation.** A red test is not evidence until the judge says so.
Mechanical first: the judge runs the test on the PR branch (must fail) and
on the base branch (must pass; failing on both means pre-existing or
broken, not this PR). Then the checklist, each item answered yes or no:

1. The assertion states the wrong outcome named in the claim, not
   something adjacent.
2. The failure is on that assertion, not on setup, imports or fixtures.
3. The unit under test is real; mocks stand in only for collaborators.
4. The inputs match the scenario; nothing was invented to force the
   failure.
5. The test would pass if the defect were fixed.

Any no: the judge returns the test to the prosecutor once, naming the
failed item. The prosecutor fixes or withdraws. A second failure of the
checklist counts as no test. A trace goes through items 1, 4 and 5 the
same way, once.

Valid failing test: verdict `confirmed`, no defender, court closes.

**Defence.** Otherwise the judge summons a fresh defender, blind to the
prosecution, with the same inputs. The defender must cite the guard,
invariant or ordering by file:line that makes the scenario impossible and
name the scenario step it breaks. Cannot: "no defence".

**Ruling.** The judge now holds the prosecutor's artifact (trace or
nothing) and the defender's (guard or nothing) and decides:

- Evidence clearly for the prosecution: `confirmed`.
- Evidence clearly for the defence: `rejected`.
- Not clear: one clarification round, at most one request per side. The
  request names a specific step and demands a runnable check or a cited
  line for it, never "elaborate". Then the judge decides between
  `confirmed`, `rejected` and `plausible`. `plausible` is a legitimate
  outcome, not a failure; the operator reads the two artifacts.

The judge adds no argument of its own at any point; it validates, requests
and ranks. Every verdict carries its artifacts: test body and both runs,
trace, defence, clarification exchange, ruling with the deciding step
named.

### 7. Filter and rank

- Keep every `confirmed` and `plausible`, marked as such.
- Drop `rejected` from the findings; they appear in the discarded list.
- Rank by severity, then verdict, then support.

### 8. Report

Write `report.md` to the profile's `runs` directory, named
`<pr>-<yyyy-mm-dd>.md`, and print it. In set mode, write it to the
scratchpad and print it; the set has no single home repository. Sections in this order:

1. **Findings.** Each with repo, file:line, claim, scenario, verdict, the
   artifacts, support.
2. **Not verified.** The triage rows the operator did not choose.
3. **Coverage.** The matrix: cells covered, cells with no owner output,
   reviewers that failed or timed out, lenses dropped at the gate.
4. **Discarded.** One line per rejected cluster with the deciding step, so
   a human can spot-check the court.
5. **Metrics.** Per lens: findings, unique after dedup, sent to court,
   confirmed, rejected, plausible, tokens, wall time. Per court: agents
   spawned. Totals.
6. **Profile suggestion.** Only when no profile existed.

## Rules

- Reviewers: fresh context, one lens, read-only, structured output only.
- Triage: rates damage if true, never truth. The operator chooses.
- Judge: owns the court, validates evidence, requests once, rules; adds
  no argument. Prosecutor and defender: blind to support and to each
  other; artifact or nothing.
- Orchestrator: never judges content, only structure. If the pipeline
  cannot run a step, report the gap; do not fill it by reviewing yourself.
- Never post to the PR. The report is local; the user decides what to post.
