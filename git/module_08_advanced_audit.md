# Module 08: Advanced Audit and Debug

## Why this matters for your profile
In CI/CD and CT failures, root-cause speed is a major differentiator. Git forensic commands are often your fastest path to signal.

## Concept clarity
High-value investigation tools:
- blame: who changed a line and when
- bisect: binary search for introducing commit
- grep/log pickaxe: find change introduction
- range-diff: compare two patch series

## Command mastery

    git blame <file>
    git log -- <file>
    git log -S "keyword" -- <file>
    git log -G "regex" -- <file>
    git bisect start
    git bisect bad
    git bisect good <known-good-commit>
    git bisect run ./test_script.sh
    git bisect reset
    git range-diff origin/main...feature/v1 origin/main...feature/v2

## Practical lab: regression hunt
1. Create repo with 8 commits.
2. Introduce a failure at commit 6.
3. Use bisect to find offender.
4. Use blame and pickaxe to validate reasoning.

Pass criteria:
- Identify exact offending commit quickly.
- Explain evidence trail.

## Mock interview
1. How do you debug flaky pipeline breakage from Git side?
Strong answer: isolate failure window, run bisect with deterministic test script, then verify with blame and diff context.

2. When is blame misleading?
Strong answer: formatting/refactor commits can hide semantic origin; I cross-check with log and pickaxe history.

3. Why is range-diff useful in review cycles?
Strong answer: it shows what changed between patch set revisions, which is excellent for Gerrit or PR iteration.

## Hands-on challenge
- Build a small test script and automate git bisect run.
- Produce a one-page incident note with evidence links.