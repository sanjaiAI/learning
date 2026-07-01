# Module 06: History Surgery

## Why this matters for your profile
Production DevSecOps needs safe correction of mistakes: wrong commit, wrong branch, accidental secret, or bad merge.

## Concept clarity
Key tools:
- reset: move branch pointer (local rewrite)
- revert: create inverse commit (safe for shared branches)
- amend: fix last commit
- cherry-pick: copy specific commit to another branch
- reflog: recover lost references

Golden rule:
Prefer revert over reset on shared/protected branches.

## Command mastery

    git commit --amend
    git reset --soft HEAD~1
    git reset --mixed HEAD~1
    git reset --hard HEAD~1
    git revert <commit>
    git cherry-pick <commit>
    git reflog
    git checkout -b recovered <hash>

## Practical lab: controlled mistake recovery
1. Create three commits.
2. Introduce a bad commit.
3. Revert it on main.
4. Use reflog to recover a dropped commit in a side branch.

Pass criteria:
- You can recover work from reflog.
- You use revert for shared history correction.

## Mock interview
1. reset vs revert in team environments?
Strong answer: reset rewrites history and is local-safe; revert preserves history and is collaboration-safe.

2. How do you recover after accidental hard reset?
Strong answer: inspect reflog, locate previous HEAD, create branch from that hash, then re-integrate safely.

3. When do you cherry-pick in release engineering?
Strong answer: to move a narrowly scoped, approved fix from main to release without pulling unrelated commits.

## Hands-on challenge
- Simulate accidental hard reset and recover all lost commits via reflog.
- Demonstrate both revert and cherry-pick in one scenario.