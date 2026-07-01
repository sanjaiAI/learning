# Module 02: Daily Workflow Mastery

## Why this matters for your profile
In high-volume pipelines, commit hygiene directly impacts SAST, license scanning, policy gates, and release velocity.

## Concept clarity
Daily Git flow is about predictable change control:
- Inspect changes
- Stage intentionally
- Commit atomically
- Review history quickly

## Command mastery
Core command set:

    git status
    git diff
    git add -p
    git restore <file>
    git restore --staged <file>
    git commit -m "type(scope): message"
    git log --oneline --graph --decorate -20
    git show <commit>

Useful patterns:

    git commit --amend
    git diff --staged --name-only
    git log --stat --since="7 days ago"

## Practical lab: commit hygiene under pressure
Scenario: one file has a real fix, two files contain accidental debug edits.

Tasks:
1. Stage only the real fix with git add -p.
2. Unstage accidental hunks.
3. Commit with a semantic message.
4. Prove clean commit scope using git show.

Pass criteria:
- Commit contains only intended hunks.
- Message explains why, not just what.

## Common mistakes and correction drills
- Mistake: adding everything with git add .
  Drill: always run git diff --staged before commit.
- Mistake: mixing refactor and functional change.
  Drill: split into separate commits.

## Mock interview
1. How do you keep commits review-friendly?
Strong answer: I use hunk-level staging and keep one logical change per commit, which improves review speed and rollback safety.

2. How do you recover if wrong files were staged?
Strong answer: git restore --staged <file> or interactive reset at hunk level, then re-stage correctly.

3. What commit message style do you use in CI-heavy teams?
Strong answer: Conventional style with type and scope; body includes impact and validation evidence when required.

## Hands-on challenge
- Make a mixed change in three files.
- Produce two clean commits from that one mixed edit.
- Explain your staging decisions as if in code review.