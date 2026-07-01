# Module 07: Remote Collaboration at Scale

## Why this matters for your profile
Your work involved governance-heavy environments with enterprise controls, which requires strong remote and branch protection discipline.

## Concept clarity
Important remote concepts:
- origin/main is a remote-tracking branch
- fetch updates remote-tracking refs without changing local branch
- pull = fetch + merge/rebase
- push updates remote refs based on policy

Safe collaboration baseline:
- fetch first
- review incoming commits
- use pull --ff-only where possible
- avoid force-push on protected branches

## Command mastery

    git remote -v
    git fetch --all --prune
    git branch -vv
    git pull --ff-only
    git push -u origin feature/x
    git push --force-with-lease origin feature/x

Tracking and cleanup:

    git branch --set-upstream-to=origin/main main
    git remote prune origin

## Practical lab: two-clone collaboration
1. Clone same repo into cloneA and cloneB.
2. Create diverging commits.
3. Practice fetch, compare, integrate.
4. Simulate rejected push and fix safely.

Pass criteria:
- You can resolve non-fast-forward errors without panic.
- You can explain force-with-lease safety.

## Mock interview
1. Why prefer pull --ff-only?
Strong answer: it avoids accidental merge commits and enforces explicit integration decisions.

2. Why force-with-lease instead of force?
Strong answer: it prevents overwriting unseen remote updates by validating expected remote state.

3. How do branch protections help DevSecOps?
Strong answer: they enforce review and pipeline gates, reducing unauthorized or non-compliant changes.

## Hands-on challenge
- Reproduce and resolve one non-fast-forward push rejection.
- Document exact fix steps and risk checks.