---
tags: [git, index, moc]
aliases: [Git Guide, Git Study Guide]
---
# Git: From Commit to Team Workflow

> [!summary] What this guide is for
> You can stage, commit and push. This guide fills in everything else, so you can work on a team **without fear of losing work**: branching, merging, pull requests, resolving conflicts, rebasing, undoing mistakes, and recovering from the scary situations.
>
> It's written to **teach**, not just list commands. Each note explains what Git is actually doing, why you'd use each command, what can go wrong, and how to get back. All terminal output comes from real runs, and the [[13 Practice Lab]] lets you safely cause and fix every problem on your own machine.
>
> **Every note follows the same shape:** a summary of what you'll learn → explanations with diagrams and real examples → pitfalls → a **self-test checklist** at the bottom.

> [!note] About the code blocks
> Commands are in `sh` blocks on purpose, so the Execute Code plugin shows no Run button on them. This vault is itself a Git repo, so accidentally running something like `git reset --hard` here would be a real problem. Type commands into a terminal in the right folder.

---

## The one idea that removes most of the fear

> [!tip] Committed work is almost never lost
> Git doesn't delete commits when you reset, rebase, amend or delete a branch. It just moves labels, and it keeps a log of where everything used to point (the **reflog**) for about 90 days. What *can* be lost is work Git **never saw**: uncommitted edits and untracked files. So: **commit early and often on a branch**, run `git status` before anything you're unsure of, and use `--abort` when a merge or rebase gets confusing. Details in [[01 How Git Thinks#Why committed work is very hard to lose]].

---

## Study path

| # | Note | What you'll be able to do |
|---|---|---|
| 1 | [[01 How Git Thinks]] | Picture commits, the staging area, branches and HEAD, which makes every other command make sense |
| 2 | [[02 Setup and Your First Commit]] | Configure Git safely; start a project from `git init` to the first push; master `.gitignore` |
| 3 | [[03 The Everyday Loop]] | Review changes with `diff`, build focused commits with `add -p`, write good messages, explore history |
| 4 | [[04 Branches and Merging]] | Create and switch branches confidently; understand fast-forward vs merge commits |
| 5 | [[05 Remotes - Fetch, Pull, Push]] | Understand `origin/main`, fetch vs pull, rejected pushes, and force-pushing safely |
| 6 | [[06 Team Workflow and Pull Requests]] | Run the full feature-branch → PR → review → merge → cleanup cycle like a professional |
| 7 | [[07 Resolving Merge Conflicts]] | Read conflict markers, resolve any conflict type, and know your escape hatches |
| 8 | [[08 Rebasing]] | Rebase onto main, clean up commits with interactive rebase, use `--onto`, and undo any rebase |
| 9 | [[09 Undoing Things Safely]] | Pick the right undo (`restore`, `reset`, `revert`), and recover anything with the reflog |
| 10 | [[10 Stash, Cherry-pick and Other Tools]] | Use stash, cherry-pick, bisect, tags and worktrees |
| 11 | [[11 Recovery Cookbook]] | Step-by-step fixes for 20+ real "oh no" situations |
| 12 | [[12 Git Cheat Sheet]] | Every command on one page, with danger levels |
| 13 | [[13 Practice Lab]] | Hands-on: cause and fix conflicts, rebases and "lost" commits in a throwaway repo |

### Suggested plan
- **Session 1:** notes 1–3, then set up the recommended config from note 2.
- **Session 2:** notes 4–6. This is the team workflow, the core of working professionally.
- **Session 3:** notes 7–8, then Practice Lab exercises 1–3.
- **Session 4:** notes 9–11, then Practice Lab exercises 4–8.
- **Ongoing:** keep [[12 Git Cheat Sheet]] open while you work, and use the [[11 Recovery Cookbook]] when something goes wrong.

---

## Which note do I need right now?

| Situation | Go to |
|---|---|
| Starting a new project | [[02 Setup and Your First Commit#Starting a brand-new project]] |
| Starting a ticket on a team | [[06 Team Workflow and Pull Requests#The full lifecycle of a change]] |
| "CONFLICT (content): Merge conflict in ..." | [[07 Resolving Merge Conflicts#The resolution procedure]] |
| My PR says "This branch has conflicts" | [[07 Resolving Merge Conflicts#Scenario 3 - My PR has conflicts with main]] |
| Push rejected | [[11 Recovery Cookbook#My push was rejected]] |
| Committed to the wrong branch or to main | [[11 Recovery Cookbook#I committed to main instead of a branch]] |
| "My commits are gone" | [[11 Recovery Cookbook#I reset or rebased and my commits are gone]] |
| Need to clean up messy commits before review | [[08 Rebasing#Interactive rebase for cleaning up your commits]] |
| Something merged to main broke production | [[11 Recovery Cookbook#A merged PR broke main]] |
| Stuck mid-merge or mid-rebase | [[11 Recovery Cookbook#I'm stuck in the middle of a merge or rebase]] |

---

## Best external resources
- **Pro Git** book (free at git-scm.com/book): the definitive reference. Chapters 2, 3 and 7 pair with this guide
- **Learn Git Branching** (learngitbranching.js.org): interactive visual puzzles for branching, rebasing and cherry-picking. Excellent for building the mental model
- **Oh Shit, Git!?!** (ohshitgit.com): short recipes for common mistakes
- **GitHub Docs: About pull requests** and **About merge methods**
- `git help <command>` (for example `git help rebase`): the official manuals, which are more readable than their reputation

## ✅ Progress tracker
- [ ] 01–03: fundamentals and everyday loop
- [ ] 04–06: branching, remotes and team workflow
- [ ] 07: merge conflicts
- [ ] 08: rebasing
- [ ] 09–11: undo, tools and recovery
- [ ] Practice Lab: all 8 exercises done without looking
- [ ] Recommended config applied on my machine
