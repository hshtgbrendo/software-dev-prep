---
tags: [git, tools]
aliases: [git stash, git cherry-pick, git bisect, git tag, git worktree]
---
# Stash, Cherry-pick and Other Tools

> [!summary] What you'll learn
> The specialist tools that solve specific, recurring problems:
> - **`stash`**: set uncommitted work aside to switch tasks.
> - **`cherry-pick`**: copy specific commits to another branch.
> - **`bisect`**: binary-search history to find which commit introduced a bug.
> - **`tag`**: mark releases.
> - **`worktree`**: check out a second branch in another folder at the same time.

Back to [[00 Git Index]] · Prev: [[09 Undoing Things Safely]] · Next: [[11 Recovery Cookbook]]

---

## git stash

**Problem it solves:** you're halfway through something, and you need a clean working directory *now*: to switch branches for an urgent review, pull, or test something on `main`. You don't want to commit half-done work.

`git stash` saves your uncommitted changes (staged and unstaged) onto a stack and resets your files to the last commit.

```sh
git stash push -m "half-done logging"   # save with a description (plain "git stash" also works)
git stash list
```

```text
stash@{0}: On feature/login: half-done logging
```

```sh
git switch main            # working directory is clean, so switching is free
# ...do the urgent thing...
git switch feature/login
git stash pop              # reapply the most recent stash and remove it from the stack
```

| Command | What it does |
|---|---|
| `git stash` / `git stash push -m "msg"` | Stash tracked changes (staged + unstaged) |
| `git stash -u` | Also stash **untracked** (new) files. Without `-u`, new files stay behind in your working directory |
| `git stash push -p` | Choose hunks interactively |
| `git stash push path/to/file` | Stash only certain files |
| `git stash list` | Show all stashes |
| `git stash show -p stash@{1}` | See a stash's diff |
| `git stash pop` | Apply the latest stash and drop it (kept if the pop conflicts) |
| `git stash apply stash@{1}` | Apply a stash but **keep** it on the stack |
| `git stash drop stash@{1}` | Delete one stash |
| `git stash branch new-branch` | Create a branch from where the stash was made and apply it there (avoids conflicts) |

**Conflicts on pop:** if the code changed since you stashed, `pop` can conflict. Resolve it like any conflict ([[07 Resolving Merge Conflicts]]). The stash is **not** dropped when a pop conflicts, so run `git stash drop` once you're done.

> [!tip] Stash vs WIP commit
> Stashes are easy to forget: they don't belong to a branch, and after a few weeks `stash@{3}` means nothing. For anything longer than a quick context switch, a **WIP commit on your branch** is better: `git commit -am "WIP: logging"`, then later either keep working and squash it, or `git reset HEAD~` to un-commit it. Commits are visible, pushable and backed up.

**Lost a dropped stash?** Dropped stashes become dangling commits: `git fsck --lost-found`, then `git show` the candidates, then `git stash apply <sha>`.

---

## git cherry-pick

**Problem it solves:** you need **one specific commit** (or a few) from another branch, without merging everything else on that branch.

Common real cases:
- A bug fix landed on `main` and must also go onto `release/2.4` (a backport).
- You committed to the wrong branch and want to copy the commit to the right one ([[11 Recovery Cookbook#I committed to the wrong branch]]).
- A teammate's branch has one commit your work depends on, and their PR isn't merged yet.

```sh
git switch release/2.4
git cherry-pick a1b2c3d
```

```text
[release/2.4 ef1e21d] Fix crash when cart is empty
 Date: Sun Nov 8 01:57:00 2026 -0800
 1 file changed, 1 insertion(+)
```

Like rebase, cherry-pick **copies**: it applies the commit's changes on top of your branch as a **new commit with a new ID** (`ef1e21d` vs the original `a1b2c3d`). The original stays on its branch.

```sh
git cherry-pick a1b2c3d e4f5a6b      # several commits, applied in order
git cherry-pick a1b2c3d^..e4f5a6b    # a range, including a1b2c3d (without ^ the first is excluded)
git cherry-pick -x a1b2c3d           # add "(cherry picked from commit a1b2c3d)" to the message (good for backports)
git cherry-pick --no-commit a1b2c3d  # apply the changes but don't commit (to combine or tweak)
git cherry-pick -m 1 <merge-sha>     # cherry-pick a merge commit (relative to parent 1)
```

**Conflicts:** fix the files, `git add`, then `git cherry-pick --continue`, or bail out with `git cherry-pick --abort`.

**When not to cherry-pick:** as a substitute for merging. If you cherry-pick many commits between long-lived branches, the same changes exist twice with different IDs, which confuses later merges. Prefer merging or rebasing for whole branches. Cherry-pick is for **individual, targeted commits**.

---

## git bisect

**Problem it solves:** "It worked two weeks ago and it's broken now. There are 200 commits in between." Bisect does a **binary search**: it checks out the middle commit, you say good or bad, and it halves the range. 200 commits take about 8 steps.

```sh
git bisect start
git bisect bad                 # the current commit is broken
git bisect good v1.4.0         # this tag/commit was known to work
# Git checks out a commit in the middle:
#   "Bisecting: 100 revisions left to test after this (roughly 7 steps)"
# test it (run the app or a test), then tell Git:
git bisect good                # or: git bisect bad
# ...repeat until Git prints "a1b2c3d is the first bad commit"
git bisect reset               # IMPORTANT: return to your original branch
```

**Automate it** if you can write a check that exits 0 for good and non-zero for bad:

```sh
git bisect start HEAD v1.4.0
git bisect run npm test -- --testNamePattern="cart total"
git bisect reset
```

Git runs the command at each step and finds the first bad commit by itself. Use `git bisect skip` for a commit that can't be tested (for example, it doesn't build). This is one reason small, working commits matter.

---

## git tag

Tags are **labels that don't move**, used to mark releases (`v1.4.0`). Unlike branches, they stay on one commit forever.

```sh
git tag                                   # list tags
git tag -a v1.4.0 -m "Release 1.4.0"      # annotated tag (has author, date, message): use for releases
git tag v1.4.0-rc1                        # lightweight tag (just a name): fine for private markers
git tag -a v1.3.1 a1b2c3d -m "Hotfix"     # tag an older commit
git push origin v1.4.0                    # tags are NOT pushed by default
git push --follow-tags                    # push commits plus the annotated tags pointing at them
git tag -d v1.4.0                         # delete locally
git push origin --delete v1.4.0           # delete on GitHub
git switch --detach v1.4.0                # look at the code as of a release (detached HEAD)
```

Most teams use **semantic versioning**: `MAJOR.MINOR.PATCH`. Bump MAJOR for breaking changes, MINOR for new features, PATCH for fixes. GitHub Releases are built on tags.

---

## git worktree

**Problem it solves:** you're mid-feature with a running dev server and uncommitted changes, and you need to review a PR or hotfix `main` *without* stashing and switching. A worktree gives you **another working directory for the same repo**, on a different branch.

```sh
git worktree add ../app-hotfix main            # check out main in a sibling folder
git worktree add -b fix/crash ../app-crash origin/main   # new branch in a new folder
cd ../app-hotfix                               # work, commit, push as normal
git worktree list
git worktree remove ../app-hotfix              # clean up when done
```

All worktrees share one `.git` (same commits, branches and stashes), so they don't take a full clone's space or need a separate fetch. One rule: the same branch can't be checked out in two worktrees at once.

---

## A few more you'll occasionally need

| Command | What it's for |
|---|---|
| `git shortlog -sn` | Commit counts per author |
| `git log --oneline --graph --all` | The big picture of all branches (alias it: see [[03 The Everyday Loop#Exploring history]]) |
| `git grep "TODO"` | Fast search of tracked files (`git grep "x" v1.4.0` searches an old version) |
| `git rev-parse HEAD` | Print the full ID of a commit (handy in scripts and CI) |
| `git describe --tags` | Name the current commit relative to the latest tag, like `v1.4.0-12-ga1b2c3d` |
| `git submodule update --init --recursive` | If a repo uses submodules (other repos nested inside), fetch them after cloning |
| `git lfs` | Git Large File Storage for big binaries (design files, videos); some repos require it |
| `git sparse-checkout` | Only check out part of a huge monorepo |

---

## Quick self-test
- [ ] When would you stash instead of making a WIP commit, and vice versa? How do you stash new untracked files?
- [ ] What happens to a stash when `git stash pop` conflicts?
- [ ] Give two legitimate cherry-pick use cases. Why does the cherry-picked commit get a new ID? Why shouldn't you use cherry-pick instead of merging?
- [ ] Walk through finding a bug with `git bisect`. How do you automate it?
- [ ] What's the difference between an annotated and a lightweight tag? Why doesn't `git push` send your tags?
- [ ] What problem does `git worktree` solve?

Next: [[11 Recovery Cookbook]]
