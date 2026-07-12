---
title: Require pull requests and a merge queue on main
date: 2026-06-06
status: accepted
decision-makers: K. S. Ernest (iFire) Lee
tier: baseline
---

## Context and Problem Statement

The `manuals` repo had no branch protection on `main`. Changes landed by pushing
directly or by manually merging pull requests one at a time, with no serialization
between concurrent PRs. As the number of in-flight documentation PRs grows, two
PRs can each pass against an older `main` and then conflict or break the Quarto
build once both land. How should changes reach `main` so that every merge is
reviewable and tested against the tip it will actually join?

## Decision Drivers

- Every change to `main` should arrive through a pull request, not a direct push.
- Concurrent PRs should serialize so each is validated against the latest `main`.
- The team is small, so the process should add little ceremony (no mandatory
  second reviewer).
- The setting should be declarative and recorded, not click-ops folklore.

## Considered Options

- Leave `main` unprotected (status quo).
- Branch protection requiring a pull request only.
- A repository ruleset requiring a pull request plus a merge queue.

## Decision Outcome

Chosen option: "A repository ruleset requiring a pull request plus a merge queue",
because it serializes merges and keeps `main` PR-only without forcing a second
reviewer.

The ruleset targets the default branch with these rules:

- Block branch deletion and non-fast-forward pushes.
- Require a pull request, with `0` required approvals (review is allowed, not
  mandated) and all three merge methods permitted on the PR.
- Require a merge queue using the merge commit method, `ALLGREEN` grouping, up to
  5 entries built and merged per batch, and a 1-minute minimum wait.

### Consequences

- Good: `main` only changes through PRs, and the queue tests each entry against
  the branch tip before it lands.
- Good: the configuration is captured here and reproducible from the ruleset JSON.
- Bad: solo edits now need a PR and an enqueue step.
- Bad: with `0` required approvals, the rule enforces process but not review, so
  an unreviewed PR can still merge.

### Confirmation

The ruleset is active (id `17352485`). `gh api repos/v-sekai-multiplayer-fabric/manuals/rulesets`
lists it, and a direct push to `main` is rejected. Future PRs land via the queue.

## More Information

Created with `gh api -X POST repos/v-sekai-multiplayer-fabric/manuals/rulesets`.
To change the policy, edit the ruleset rather than protecting the branch through
the classic branch-protection API, so the two mechanisms do not overlap.

## Amendment (2026-07-12): auto-delete head branches after merge

Force branch deletion after merge by enabling the repository's **automatically
delete head branches** setting (`delete_branch_on_merge`). When a PR merges, its
source branch is removed automatically, so merged feature branches do not pile up.

Why this is a repo setting and not part of the ruleset:

- The merge queue rejects deletion at merge time — `gh pr merge --delete-branch`
  fails with *"Cannot use `-d` or `--delete-branch` when merge queue enabled"*, so
  the per-merge flag cannot do the cleanup. The repo-level `delete_branch_on_merge`
  setting fires after the queue lands the commit and is the correct mechanism.
- It does **not** conflict with the ruleset's *"block branch deletion"* rule. That
  rule protects the ruleset's target (the default branch) from being deleted; this
  setting deletes the **merged PR's source branch**. Different branches, different
  mechanisms.

Apply it declaratively:

```sh
gh api -X PATCH repos/v-sekai-multiplayer-fabric/<repo> \
  -F delete_branch_on_merge=true
```

### Confirmation

`gh api repos/v-sekai-multiplayer-fabric/<repo> --jq .delete_branch_on_merge`
returns `true`, and the source branch of a merged PR no longer exists.
