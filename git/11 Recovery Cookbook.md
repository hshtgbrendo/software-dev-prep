---
tags: [git, recovery, troubleshooting]
aliases: [Git Recipes, Git Problems, Oh Shit Git, Fix Git Mistakes]
---
# Recovery Cookbook

> [!summary] What you'll learn
> Step-by-step fixes for the situations that actually happen, each with **the commands, why they work, and how risky they are**. Find your problem in the list below.
>
> Before any fix: **run `git status`** and read it. Then, if you're nervous, **make a safety label**: `git branch backup/before-fix`. It costs nothing and means that whatever happens next, your current commits have a name you can return to.

Back to [[00 Git Index]] · Prev: [[10 Stash, Cherry-pick and Other Tools]] · Next: [[12 Git Cheat Sheet]]

---

## Index of problems

**Commits in the wrong place**
- [[#I committed to the wrong branch]]
- [[#I committed to main instead of a branch]]
- [[#I made commits in detached HEAD]]
- [[#I branched from the wrong place]]

**Fixing commits**
- [[#I need to change the last commit message]]
- [[#I forgot to add a file to the last commit]]
- [[#I want to undo my last commit but keep the changes]]
- [[#I need to fix a commit further back]]
- [[#I committed a huge file or node_modules]]
- [[#I committed a secret]]

**"I lost my work"**
- [[#I reset or rebased and my commits are gone]]
- [[#I deleted a branch I needed]]
- [[#My uncommitted changes disappeared]]

**Remote and team problems**
- [[#My push was rejected]]
- [[#My first push was rejected because the GitHub repo has a README]]
- [[#Pull made a weird merge commit]]
- [[#I want my local branch to exactly match GitHub]]
- [[#A teammate force-pushed and my pull is a mess]]
- [[#A merged PR broke main]]
- [[#I need to test a teammate's PR locally]]

**Being stuck**
- [[#I'm stuck in the middle of a merge or rebase]]
- [[#I'm stuck in Vim]]
- [[#Every line in a file shows as changed]]

---

## Commits in the wrong place

### I committed to the wrong branch

You meant to commit on `feature/a` but were on `feature/b`. The commit isn't pushed.

```sh
git log --oneline -3            # note the ID of the misplaced commit, e.g. a1b2c3d
git switch feature/a
git cherry-pick a1b2c3d         # copy it to the right branch
git switch feature/b
git reset --hard HEAD~1         # remove it from the wrong branch (check git status first!)
```

**Why it works:** cherry-pick copies the commit's changes to the correct branch, and reset moves the wrong branch's label back one commit. The commit isn't lost: it now exists on `feature/a`.
**If you'd already pushed `feature/b`:** use `git revert a1b2c3d` on `feature/b` instead of reset (or reset plus force-push with lease if it's your own branch).

For several commits, cherry-pick a range (`git cherry-pick a1b2c3d^..e4f5a6b`) and reset by that many (`HEAD~3`).

### I committed to main instead of a branch

Very common: you forgot to create a branch, and now local `main` has commits that should be in a PR. As long as you haven't pushed (protected `main` usually rejects the push anyway):

```sh
git status                      # make sure there's no uncommitted work (stash it if there is)
git branch feature/my-work      # 1. put a new label on your current commit: your work is now safe on it
git reset --hard origin/main    # 2. move main back to match GitHub
git switch feature/my-work      # 3. continue on the branch
git push -u origin feature/my-work
```

**Why it works:** step 1 makes `feature/my-work` point at your latest commit, so those commits belong to that branch. Step 2 only moves the `main` label back to where GitHub's `main` is; the commits aren't touched. Run `git fetch` first if you want `origin/main` to be current.
**Risk:** low, provided `git status` was clean. The `--hard` only affects uncommitted changes.

### I made commits in detached HEAD

You checked out a commit or tag, made commits, and `git status` says `HEAD detached at ...`:

```sh
git switch -c rescue/experiment     # create a branch right here: done, the commits are safe
```

**Already switched away and the commits seem gone?** Git warned you with their ID when you left, and they're in the reflog:

```sh
git reflog                          # find the last commit you made while detached
git branch rescue/experiment <sha>
```

### I branched from the wrong place

Your branch was created from another feature branch instead of `main`, so the PR includes someone else's commits:

```sh
git rebase --onto origin/main wrong-base-branch my-branch
git push --force-with-lease --force-if-includes
```

This replays only the commits after `wrong-base-branch` onto `main`. See [[08 Rebasing#Moving a branch to a different base with --onto]].

---

## Fixing commits

### I need to change the last commit message

```sh
git commit --amend -m "Correct message"
```

**Not pushed:** done. **Pushed to your own branch:** also run `git push --force-with-lease --force-if-includes`. **Pushed to a shared branch or `main`:** leave it. A typo in a message isn't worth rewriting shared history.

### I forgot to add a file to the last commit

```sh
git add forgotten.js
git commit --amend --no-edit
```

Same push rules as above.

### I want to undo my last commit but keep the changes

```sh
git reset --soft HEAD~1    # changes stay staged
# or
git reset HEAD~1           # changes stay in your files, unstaged
```

**Why:** reset moves the branch label back one commit, and `--soft`/`--mixed` leave your files alone. If it was already pushed to a shared branch, use `git revert HEAD` instead.

### I need to fix a commit further back

The bug is in a commit three back on your branch, and you want the fix to live in that commit:

```sh
git add the-fix.js
git commit --fixup <sha-of-the-commit-to-fix>
git rebase -i --autosquash origin/main     # save and close the todo list as is
git push --force-with-lease --force-if-includes   # if already pushed
```

Or `git rebase -i origin/main`, mark the commit `edit`, make the change, `git commit --amend`, then `git rebase --continue`. See [[08 Rebasing#Fixup commits and autosquash]].

### I committed a huge file or node_modules

**Last commit, not pushed:**

```sh
echo "node_modules/" >> .gitignore
git rm -r --cached node_modules     # untrack it; keeps the files on disk
git add .gitignore
git commit --amend --no-edit
```

**A few commits back, not pushed:** interactive rebase, `edit` the commit that added it, do the same `git rm --cached`, then `git commit --amend` and `git rebase --continue`. Fixing only the latest commit leaves the big file in history, and GitHub rejects files over 100 MB even if a later commit deletes them.

**Already pushed:** removing it in a new commit stops tracking it, but history (and the repo size) keeps it. Purging it from history means rewriting shared history (`git filter-repo`). Coordinate with the team before doing that.

### I committed a secret

API key, password, `.env` file, private key.

1. **Rotate the secret first.** Revoke the key and generate a new one, *now*. If it was pushed, assume it's compromised: bots scrape GitHub for keys within minutes. **This is the real fix**; everything else is cleanup.
2. Stop tracking the file: add it to `.gitignore`, run `git rm --cached .env`, and commit.
3. **Not pushed yet:** remove it from your local commits (amend, or interactive rebase with `edit`) so it never reaches GitHub.
4. **Pushed:** purging it from history requires rewriting it with [`git filter-repo`](https://github.com/newren/git-filter-repo) (or BFG Repo-Cleaner), force-pushing every affected branch, and having teammates re-clone. GitHub may also retain cached views. That's why step 1 matters most. At a company, tell your lead or security team right away; it's a routine incident when reported fast.

Prevention: `.gitignore` your `.env` from day one, check `git diff --staged` before committing, and turn on GitHub secret scanning / push protection.

---

## "I lost my work"

### I reset or rebased and my commits are gone

They're almost certainly not gone. The commits are still in the repository, and the reflog records where your branch used to point.

```sh
git reflog --date=relative
```

Find the entry from just **before** the mistake: the line under `reset: moving to ...`, or the last entry before `rebase (start)`. Inspect it:

```sh
git log --oneline -5 <sha>
```

Then restore it:

```sh
git branch rescue <sha>         # safest: label it first and look around
git reset --hard <sha>          # or move your current branch back there (check git status first)
```

Right after a bad merge, rebase or reset, `git reset --hard ORIG_HEAD` is a shortcut. Full explanation: [[09 Undoing Things Safely#The reflog is your safety net]].

### I deleted a branch I needed

```text
$ git branch -D feature/login
Deleted branch feature/login (was aeb8a56).
```

Git even printed the commit ID. Recreate the label:

```sh
git branch feature/login aeb8a56
```

Lost that output? Search the reflog for the last time you were on that branch:

```sh
git reflog | grep "feature/login"
```

Or, if it still exists on GitHub: `git switch feature/login` recreates it from `origin/feature/login`.

### My uncommitted changes disappeared

Work through these in order:

1. **Wrong branch?** `git status`. Uncommitted changes travel with you when you switch, so check which branch you're on and which branches you've visited (`git reflog`).
2. **Stashed?** `git stash list`. A pull with autostash, or a tool, may have stashed them. `git stash show -p stash@{0}` to inspect, then `git stash pop`.
3. **Were they ever staged (`git add`)?** Then the contents exist as dangling blobs: `git fsck --lost-found`, then `git show <sha>` each blob to find your code.
4. **Never staged, then wiped by `reset --hard`, `restore` or `checkout --`?** Git can't help. Check your **editor's local history**: VS Code has "Timeline" in the Explorer and "Local History: Find Entry to Restore" in the command palette, and JetBrains IDEs have Local History. Many people have been saved by these.

**Lesson:** commit or stash before anything risky. Committed work is always recoverable.

---

## Remote and team problems

### My push was rejected

```text
 ! [rejected]        main -> main (fetch first)
```

Someone pushed before you. **Don't force-push.** Integrate their work, then push:

```sh
git pull --rebase       # replay your commits on top of theirs; resolve conflicts if any
git push
```

If the rejection says `(non-fast-forward)` right after you amended or rebased your **own** feature branch, a force push with lease is the right call. Details: [[05 Remotes - Fetch, Pull, Push#Why pushes get rejected]].

### My first push was rejected because the GitHub repo has a README

You created the GitHub repo with a README (or license), so GitHub has an "Initial commit" your local repo doesn't share. A plain merge-pull refuses:

```text
fatal: refusing to merge unrelated histories
```

Fix by rebasing your commits on top of GitHub's (tested):

```sh
git pull --rebase origin main
git push -u origin main
```

(Alternative: `git pull origin main --allow-unrelated-histories` creates a merge commit joining the two.) Next time, create the GitHub repo empty.

### Pull made a weird merge commit

You see `Merge branch 'main' of github.com:org/repo` in your log after a `git pull`. That's harmless, just noise. If it hasn't been pushed, undo it and redo the pull as a rebase:

```sh
git reset --hard ORIG_HEAD     # back to before the pull (ORIG_HEAD = pre-merge position)
git pull --rebase
```

Prevent it with `git config --global pull.rebase true`.

### I want my local branch to exactly match GitHub

Local branch is a mess and you want to throw away your local commits and changes:

```sh
git fetch
git status                       # last chance: anything uncommitted here will be destroyed
git branch backup/mess           # optional: keep a label on your local commits, just in case
git reset --hard origin/main     # or origin/<your-branch>
```

### A teammate force-pushed and my pull is a mess

A teammate rebased a branch you both use, so your local copy has the **old** commits. Pulling tries to combine old and new versions, producing duplicates and conflicts.

- **No local commits of your own on it:** abort anything in progress (`git rebase --abort` or `git merge --abort`), then `git fetch` and `git reset --hard origin/the-branch`.
- **You have your own new commits on it:**
  ```sh
  git fetch
  git rebase --onto origin/the-branch <last-old-commit-you-had-from-them> my-branch
  ```
  Or simply cherry-pick your own commits onto a fresh copy: `git switch -c fresh origin/the-branch`, then `git cherry-pick <your commits>`.

Then agree as a team to coordinate before rebasing shared branches ([[08 Rebasing#The golden rule]]).

### A merged PR broke main

Don't reset `main`. **Revert** the PR on a new branch, and put the revert through a PR like any change:

```sh
git switch -c revert/pr-142 origin/main
git log --oneline --first-parent -10    # find the PR's commit on main
```

- **PR was squash-merged or rebase-merged** (a normal commit or commits): `git revert <sha>` (one per commit for rebase-merge).
- **PR was merged with a merge commit:** `git revert -m 1 <merge-sha>`. `-m 1` says "keep parent 1 (main) as the mainline and undo what the PR brought in."

```sh
git push -u origin revert/pr-142        # open a PR, get it merged quickly
```

GitHub also has a **Revert** button on merged PRs that does all of this for you.

**Re-landing the fixed feature later:** after reverting a *merge commit*, merging the same branch again brings back nothing, because Git considers those commits already merged. Revert the revert (`git revert <revert-sha>`) and add the fix on top, or rebase the feature into new commits.

### I need to test a teammate's PR locally

```sh
gh pr checkout 142                         # GitHub CLI: easiest
# or without gh:
git fetch origin pull/142/head:pr-142
git switch pr-142
# or if the branch is in the same repo:
git fetch && git switch their-branch-name
```

---

## Being stuck

### I'm stuck in the middle of a merge or rebase

`git status` tells you which operation is in progress. You can either finish or get out:

| State | Finish | Get out (back to before) |
|---|---|---|
| Merging | Fix conflicts → `git add` → `git commit` | `git merge --abort` |
| Rebasing | Fix → `git add` → `git rebase --continue` | `git rebase --abort` |
| Cherry-picking | Fix → `git add` → `git cherry-pick --continue` | `git cherry-pick --abort` |
| Reverting | Fix → `git add` → `git revert --continue` | `git revert --abort` |

Commands like switching branches fail with `fatal: cannot switch branch while rebasing` until you finish or abort.

### I'm stuck in Vim

Git opened Vim for a commit or merge message.
- **Save and continue:** press `Esc`, type `:wq`, press Enter.
- **Quit without saving** (aborts a commit, but for a merge message the merge still continues with the default message): `Esc`, then `:q!`, then Enter.
- **Never again:** `git config --global core.editor "code --wait"` (or `nano`).

### Every line in a file shows as changed

Usually **line endings** (Windows CRLF vs Unix LF) or file permission changes:

```sh
git diff --ignore-all-space file.js     # if this diff is empty, it's whitespace/line endings
git diff --summary                      # "mode change 100644 => 100755" means permissions
```

Fixes:
- Add a `.gitattributes` with `* text=auto eol=lf` to the repo so everyone gets consistent line endings.
- On Windows, `git config --global core.autocrlf true`. On macOS/Linux, `input`.
- Ignore permission changes: `git config core.fileMode false`.

---

## Quick self-test
- [ ] You committed three times to local `main` instead of a branch. Fix it, and explain why no work is lost.
- [ ] You force-deleted a branch with unmerged work. Recover it two different ways.
- [ ] You pushed an `.env` with a live Stripe key. What is the first thing you do, and why?
- [ ] A PR merged with a merge commit broke production. Undo it safely, and explain `-m 1`.
- [ ] Your push is rejected with `(fetch first)`. What do you do, and what must you not do?
- [ ] Your uncommitted work vanished. List the four places you'd look, in order.

Next: [[12 Git Cheat Sheet]]
