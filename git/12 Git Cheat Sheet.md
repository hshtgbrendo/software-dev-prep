---
tags: [git, cheat-sheet, reference]
aliases: [Git Commands, Git Quick Reference]
---
# Git Cheat Sheet

> [!summary]
> Every command from this guide on one page, grouped by task, with a danger marker. 🟢 = safe, 🟠 = can discard uncommitted work (check `git status` first), 🔴 = can destroy work or affect teammates. Each section links to the note that explains it.

Back to [[00 Git Index]] · Prev: [[11 Recovery Cookbook]] · Next: [[13 Practice Lab]]

---

## Setup ([[02 Setup and Your First Commit]])

| Command | Does |
|---|---|
| `git config --global user.name "Name"` / `user.email` | Identify yourself |
| `git config --global pull.rebase true` | Pull rebases instead of making merge commits |
| `git config --global merge.conflictStyle zdiff3` | Conflict markers show the original code |
| `git config --global push.autoSetupRemote true` | No more "no upstream branch" errors |
| `git config --global rerere.enabled true` | Reuse conflict resolutions |
| `git init` | Create a repo in the current folder |
| `git clone <url>` | Copy a remote repo, with full history |
| `git remote add origin <url>` / `git remote -v` | Connect a remote / list remotes |
| `git rm --cached <file>` | Stop tracking a file, keep it on disk |
| `git check-ignore -v <file>` | Which `.gitignore` rule matches |

## Everyday ([[03 The Everyday Loop]])

| Command | Does | |
|---|---|---|
| `git status` / `git status -s` | What's changed, staged, which branch | 🟢 |
| `git diff` | Unstaged changes | 🟢 |
| `git diff --staged` | What the next commit will contain | 🟢 |
| `git add <file>` / `git add .` | Stage | 🟢 |
| `git add -p` | Stage hunk by hunk | 🟢 |
| `git commit -m "msg"` | Commit staged changes | 🟢 |
| `git commit --amend` | Replace the last commit (force-push needed if it was pushed) | 🟢 local |
| `git log --oneline --graph --all` | History of all branches | 🟢 |
| `git log -p <file>` / `git log -S "text"` | File history / find when code appeared | 🟢 |
| `git show <sha>` | One commit's message and diff | 🟢 |
| `git blame <file>` | Last commit to touch each line | 🟢 |

## Branches and merging ([[04 Branches and Merging]])

| Command | Does | |
|---|---|---|
| `git branch` / `-a` / `-vv` | List branches / include remote / show tracking | 🟢 |
| `git switch -c <name>` | Create and switch | 🟢 |
| `git switch <name>` / `git switch -` | Switch / switch back | 🟢 |
| `git merge <branch>` | Merge `<branch>` **into the current branch** | 🟢 |
| `git merge --abort` | Cancel a conflicted merge | 🟢 |
| `git branch -d <name>` | Delete if merged | 🟢 |
| `git branch -D <name>` | Force delete (recoverable via reflog) | 🟠 |
| `git branch -m <new>` | Rename current branch | 🟢 |

## Remotes ([[05 Remotes - Fetch, Pull, Push]])

| Command | Does | |
|---|---|---|
| `git fetch` / `git fetch --prune` | Download; never changes your branches | 🟢 |
| `git pull` / `git pull --rebase` | Fetch + integrate | 🟢 |
| `git push` / `git push -u origin <branch>` | Upload / first push of a new branch | 🟢 |
| `git push --force-with-lease --force-if-includes` | Overwrite **your own** remote branch after a rebase or amend | 🟠 |
| `git push --force` | Overwrite the remote, no checks | 🔴 never |
| `git push origin --delete <branch>` | Delete a remote branch | 🟠 |
| `git log --oneline origin/main..main` | Commits you haven't pushed | 🟢 |

## Team flow ([[06 Team Workflow and Pull Requests]])

```sh
git switch main && git pull                    # 1. latest main
git switch -c feature/thing                    # 2. branch
# ...edit, git add -p, git commit...           # 3. work
git fetch && git rebase origin/main            # 4. stay current (or: git merge origin/main)
git push -u origin feature/thing               # 5. push, open a PR
# ...review: commit + push fixes...            # 6-7. iterate
# merge on GitHub                              # 8. merge
git switch main && git pull && git branch -d feature/thing   # 9. clean up (-D after a squash merge)
```

## Conflicts ([[07 Resolving Merge Conflicts]])

| Step | Command |
|---|---|
| See what's conflicted | `git status` |
| Understand each side | Read the markers (`<<<<<<<` ours, `\|\|\|\|\|\|\|` base, `>>>>>>>` theirs) |
| Take one side for a file | `git restore --ours <file>` / `--theirs` (**swapped during a rebase**) |
| Start a file over | `git restore --merge <file>` |
| Check for leftover markers | `git diff --check` |
| Mark resolved | `git add <file>` |
| Finish | `git commit` / `git rebase --continue` / `git cherry-pick --continue` |
| Bail out | `git merge --abort` / `git rebase --abort` / `git cherry-pick --abort` |
| Lockfile conflict | Resolve `package.json`, then `npm install`, then `git add` both |

## Rebasing ([[08 Rebasing]])

| Command | Does | |
|---|---|---|
| `git rebase origin/main` | Replay your commits on top of the latest main | 🟢 own branch |
| `git rebase -i origin/main` | Edit your commits: pick, reword, squash, fixup, drop, edit | 🟢 own branch |
| `git commit --fixup <sha>` + `git rebase -i --autosquash origin/main` | Fold fixes into earlier commits | 🟢 |
| `git rebase --onto <new> <old> <branch>` | Move commits to a new base | 🟢 |
| `git rebase --continue` / `--skip` / `--abort` | During a rebase | 🟢 / 🟠 / 🟢 |
| `git reset --hard ORIG_HEAD` | Undo the rebase you just did | 🟠 |
| Golden rule | Never rebase commits others have built on | 🔴 |

## Undo ([[09 Undoing Things Safely]])

| Situation | Command | |
|---|---|---|
| Unstage a file | `git restore --staged <file>` | 🟢 |
| Discard edits to a file | `git restore <file>` | 🟠 permanent |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` | 🟢 |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` | 🟢 |
| Throw away commits and changes | `git reset --hard <sha>` | 🟠 |
| Undo a pushed commit | `git revert <sha>` | 🟢 |
| Undo a pushed merge | `git revert -m 1 <merge-sha>` | 🟢 |
| Find lost commits | `git reflog`, then `git branch rescue <sha>` | 🟢 |
| Find lost staged content or dropped stashes | `git fsck --lost-found` | 🟢 |
| Delete untracked files | `git clean -n` (dry run) then `git clean -fd` | 🔴 |

## Tools ([[10 Stash, Cherry-pick and Other Tools]])

| Command | Does |
|---|---|
| `git stash -u -m "msg"` / `git stash pop` / `git stash list` | Shelve and restore uncommitted work (including new files) |
| `git cherry-pick <sha>` / `-x` | Copy a commit onto the current branch |
| `git bisect start` / `good` / `bad` / `run <cmd>` / `reset` | Binary-search for the commit that broke something |
| `git tag -a v1.0.0 -m "msg"` / `git push --follow-tags` | Mark and publish a release |
| `git worktree add ../dir <branch>` | Second working folder on another branch |

## Panic button ([[11 Recovery Cookbook]])

1. `git status`: where am I, what's in progress?
2. Mid-operation? `--abort` returns you to the start.
3. Worried? `git branch backup/now` labels your current state.
4. Lost commits? `git reflog`.
5. Never `--force` push or `clean -f` while panicking.
