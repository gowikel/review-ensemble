---
name: review-ensemble
description: >
  Reviews a pull request with N independent fresh-context reviewers, each on a distinct lens, then dedups their claims, verifies each unique claim blind with a separate adversarial agent, and reports only what survives, with a coverage matrix and metrics. Use when a single AI review is not trusted to be complete, when a PR is high-risk, or when the user asks for an ensemble, multi-pass, or "thorough" review.
metadata:
  origin: custom
---

# Review Ensemble

One AI review is one sample from a distribution. This skill takes many
samples with forced diversity, votes, verifies, and reports. The orchestrator
never reviews code itself: it is one sample too, and its only job is
structure.

Two invariants. Reviewers never see each other. The verifier never sees how
many reviewers found a claim. Break either and the voting means nothing.

## When To Activate

- User asks for an ensemble, multi-agent, multi-pass or thorough review of a PR
- User says a previous AI review missed things and wants a stronger pass
- PR touches concurrency, state machines, auth, or money, and the user asks
  for a review

## Inputs

- **PR number** (optional): detect from the current branch if absent.
- **Lenses** (optional): explicit list; explicit always wins over the profile.
- **N** (optional): reviewers per lens, default 1. Raise for high-risk PRs;
  support count then becomes meaningful within a lens too.

## Repo profile

Generic core plus a per-repo profile. The core contains no path globs.
The profile lives at `.claude/review/profile.yaml` in the reviewed repo:

```yaml
version: 1
oracle:
  test: "npm test -- --silent"          # command that counts as ground truth
  lint: "make lint"
runs: .claude/review/runs               # per-run reports and metrics
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

### 1. Gate

```bash
/opt/homebrew/bin/gh pr view --json number,title,baseRefName,headRefName,mergeable,additions,deletions,files
/opt/homebrew/bin/gh pr diff <number> --name-only
```

Decide:

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

One fresh agent per lens (times N). Use the Agent tool with a
general-purpose agent, never a fork: a fork inherits this conversation and
correlates the samples. Each prompt contains: the pack path, the lens text
from `lenses.md` or the profile, the matrix cells it owns, and the finding
schema below. Reviewers read files as they need but must not run tests or
modify anything.

Finding schema, one JSON object per finding, in a `findings` array, plus a
`covered` array of cell ids with `found: true|false` for each cell it owns:

```json
{
  "file": "app/redux/slices/videoroom/videoroomSagas.js",
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

Before verification, so each unique claim is verified once.

- Key on file plus overlapping line range plus claim similarity.
- Cluster near-duplicates. Merge scenarios, keep the strongest evidence.
- `support` = number of distinct reviewers in the cluster.
- Record every cluster to `clusters.json` with its member reviewer ids.

### 5. Verify, adversarial and blinded

Per cluster, in a worktree so nothing touches the working tree. Every
verifier receives only: the claim, the scenario, the relevant files,
`tests.md`, the oracle command. None receives the support count, the
confidence, the reviewer's evidence, or any other cluster.

**Prosecutor.** Fresh agent. Must make the claim fail, in this order,
stopping at the first that works:

1. A failing test written against the PR branch and run with the oracle
   command. Attach the test body and the run output.
2. A concrete input trace through the code, line by line, ending in the
   wrong outcome.

Cannot do either: reports "no reproduction" with what was tried.

A failing test is deterministic. Prosecutor succeeds with a test: verdict
`confirmed`, no defender, done.

**Defender.** Fresh agent, blind to the prosecutor. Must prove the
scenario cannot occur: cite the guard, invariant or ordering by file:line,
and state which line of the scenario it breaks. Cannot: reports "no
defence".

Run prosecutor and defender in parallel unless the prosecutor's test
decided it. Then:

| prosecutor | defender | verdict |
|---|---|---|
| trace | no defence | `confirmed` |
| no reproduction | defence | `rejected` |
| trace | defence | judge |
| no reproduction | no defence | `plausible` |

**Judge.** Fresh agent, only on conflict. Receives both artifacts and the
files, nothing else. Must name the exact step where one side is wrong and
rule `confirmed` or `rejected`; if it cannot, `plausible` with both
artifacts attached. The judge never adds a new argument, it only ranks
the two given.

Every verdict carries its artifacts into the report: the test, the trace,
the defence, the ruling.

### 6. Filter and rank

- Keep every `confirmed`.
- Keep `plausible` only when `support >= 2` or severity is high.
- Drop `rejected` and everything that is purely style, unless the user
  asked for style.
- Rank by severity, then verdict, then support.

### 7. Report

Write `report.md` to the profile's `runs` directory, named
`<pr>-<yyyy-mm-dd>.md`, and print it. Sections in this order:

1. **Findings.** Each with file:line, claim, scenario, verdict, the
   verification artifact (test body or trace), support.
2. **Coverage.** The matrix: cells covered, cells with no owner output,
   reviewers that failed or timed out, lenses dropped at the gate.
3. **Discarded.** One line per rejected or filtered cluster, so a human can
   spot-check the filter.
4. **Metrics.** Per lens: findings, unique after dedup, verify pass rate,
   duplicates with other lenses, tokens, wall time. Totals.
5. **Profile suggestion.** Only when no profile existed.

## Rules

- Reviewers: fresh context, one lens, read-only, structured output only.
- Prosecutor and defender: blind to support, confidence and each other;
  artifact or nothing. Judge: ranks the two artifacts, adds none.
- Orchestrator: never judges content, only structure. If the pipeline
  cannot run a step, report the gap; do not fill it by reviewing yourself.
- Never post to the PR. The report is local; the user decides what to post.
