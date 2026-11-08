---
tags: [git, setup]
aliases: [Git Setup, git init, First Commit, gitignore]
---
# Setup and Your First Commit

> [!summary] What you'll learn
> How to configure Git once per machine (including a handful of settings that make Git **safer and less confusing**), then the exact sequence of commands to start a brand-new project: `init` → `.gitignore` → first commit → create the GitHub repo → `push -u`. Also: joining an existing project with `clone`, and how `.gitignore` really works (including how to stop tracking a file you committed by mistake).

Back to [[00 Git Index]] · Prev: [[01 How Git Thinks]] · Next: [[03 The Everyday Loop]]

---

## One-time machine setup

Git stamps every commit with a name and email. Set them once per computer:

```sh
git config --global user.name "Brendan Ng"
git config --global user.email "you@example.com"   # use the email on your GitHub account
```

`--global` writes to `~/.gitconfig`, which applies to every repo on this machine. Leave off `--global` (run inside a repo) to override a setting for just that repo, for example a work email on a work project.

### Recommended settings (and why)

These defaults make day-to-day Git noticeably safer and clearer. Each line is explained below.

```sh
git config --global init.defaultBranch main      # new repos start on "main", matching GitHub
git config --global core.editor "code --wait"    # open VS Code for commit messages and rebase lists
git config --global pull.rebase true             # "git pull" rebases your local commits instead of making merge commits
git config --global rebase.autoStash true        # auto-stash uncommitted changes during pull --rebase / rebase
git config --global push.autoSetupRemote true    # first "git push" on a new branch sets up tracking automatically
git config --global fetch.prune true             # remove origin/<branch> refs for branches deleted on GitHub
git config --global merge.conflictStyle zdiff3   # conflict markers also show the ORIGINAL code (huge help)
git config --global rerere.enabled true          # remember how you resolved a conflict and reuse it
git config --global rebase.autosquash true       # makes "fixup!" commits squash automatically in interactive rebase
git config --global diff.colorMoved zebra        # moved lines show in a different color from added/removed lines
```

| Setting | What problem it solves |
|---|---|
| `init.defaultBranch main` | Older Git creates `master`; GitHub uses `main`. Avoids a mismatch on your first push |
| `core.editor "code --wait"` | Without it, Git may drop you into Vim for commit messages and rebases. `--wait` makes Git wait until you close the tab. Use `nano` if you prefer the terminal |
| `pull.rebase true` | Plain `git pull` on diverged branches either refuses (modern Git) or creates noisy "Merge branch 'main' of github.com..." commits. Rebasing keeps your history a straight line. See [[05 Remotes - Fetch, Pull, Push#fetch vs pull]] |
| `rebase.autoStash true` | Rebase refuses to start when you have uncommitted changes. This stashes them first and reapplies them afterwards |
| `push.autoSetupRemote true` | Saves you from the `fatal: The current branch has no upstream branch` error on every new branch (Git 2.37+) |
| `fetch.prune true` | Keeps `git branch -r` from filling up with branches teammates already merged and deleted |
| `merge.conflictStyle zdiff3` | Conflict markers show three versions (yours, theirs, **and the original**), so you can see what each side actually *changed*. Explained in [[07 Resolving Merge Conflicts]] (Git 2.35+) |
| `rerere.enabled true` | "Reuse recorded resolution." If the same conflict comes up again (common during rebases), Git resolves it the way you did last time |
| `rebase.autosquash true` | Pairs with `git commit --fixup` ([[08 Rebasing#Fixup commits and autosquash]]) |

Check what's set with `git config --global --list`, or see where a value comes from with `git config --show-origin user.email`.

### Authentication with GitHub

GitHub doesn't accept your account password for Git operations. Pick one:
- **GitHub CLI (easiest):** install `gh`, run `gh auth login`, choose HTTPS, and it configures Git's credential helper for you.
- **SSH key:** `ssh-keygen -t ed25519 -C "you@example.com"`, add the contents of `~/.ssh/id_ed25519.pub` to GitHub (Settings → SSH and GPG keys), then use `git@github.com:user/repo.git` URLs. Test with `ssh -T git@github.com`.

---

## Starting a brand-new project

The full flow, with what each step does.

### 1. Create the repository

```sh
mkdir my-app && cd my-app
git init
```

```text
Initialized empty Git repository in /home/brendan/my-app/.git/
```

`git init` creates the hidden `.git` folder, which **is** the repository. Your project folder is now the working directory. Nothing is tracked yet. (Delete `.git` and the folder is just a normal folder again, with all its history gone. Never delete `.git` casually.)

> [!warning] Don't nest repositories
> Run `git init` in the project folder, not your home folder or a parent of other projects. A repo inside another repo causes confusing behavior. If `git status` shows files from all over your computer, you ran `git init` in the wrong place: delete that stray `.git` folder.

### 2. Add a `.gitignore` before the first commit

Create `.gitignore` **first**, so you never accidentally commit `node_modules` or secrets. For a Node project:

```gitignore
# dependencies (reinstallable with npm install)
node_modules/

# environment variables and secrets: NEVER commit these
.env
.env.*
!.env.example

# build output (regenerated by the build)
dist/
build/
coverage/

# logs and OS/editor junk
*.log
.DS_Store
.vscode/*
!.vscode/extensions.json
```

How the patterns work:

| Pattern | Matches |
|---|---|
| `node_modules/` | A folder named `node_modules` anywhere (trailing `/` = directories only) |
| `*.log` | Any file ending in `.log`, at any depth |
| `/config.local.js` | Leading `/` = only at the repo root |
| `!.env.example` | `!` re-includes something an earlier line excluded |
| `docs/**/*.pdf` | `**` matches any number of directories |

GitHub's [gitignore templates](https://github.com/github/gitignore) have good starting points for every language. Commit `.gitignore` itself; it's shared with the team.

> [!info] What should and shouldn't be committed
> **Commit:** source code, config files, `package.json` **and** `package-lock.json` (the lockfile guarantees everyone installs identical versions), docs, tests, `.env.example` with fake values.
> **Don't commit:** dependencies (`node_modules`), build output, secrets (`.env`, API keys, private keys), large binaries, personal editor settings.

### 3. Check what Git sees, stage, and commit

```sh
git status
```

```text
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	.gitignore
	greet.js

nothing added to commit but untracked files present (use "git add" to track)
```

Read this every time. It confirms you're on `main`, nothing is committed yet, and lists what Git will pick up. If `node_modules` or `.env` show up here, fix `.gitignore` **before** adding.

```sh
git add .                       # stage everything in this folder (respecting .gitignore)
git status                      # verify what's staged
```

```text
Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   .gitignore
	new file:   greet.js
```

```sh
git commit -m "Initial commit"
```

```text
[main (root-commit) c9a6f53] Initial commit
 2 files changed, 5 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 greet.js
```

`root-commit` means this commit has no parent: it's the start of history.

### 4. Create the GitHub repository and connect it

On GitHub, create a new repository **without** a README, .gitignore or license. If GitHub creates those, its repo already has a commit that your local repo doesn't, and your first push will be rejected. (If that happens, see [[11 Recovery Cookbook#My first push was rejected because the GitHub repo has a README]].)

Or with the GitHub CLI, in one step: `gh repo create my-app --private --source=. --push`.

Manually:

```sh
git remote add origin git@github.com:brendan/my-app.git   # give the GitHub URL the nickname "origin"
git remote -v                                              # verify
git push -u origin main
```

```text
To github.com:brendan/my-app.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

What `-u` (short for `--set-upstream`) does: it links your local `main` to `origin/main`. After that, plain `git push` and `git pull` on `main` know where to go, and `git status` can tell you if you're ahead or behind. You only need `-u` once per branch (or never, if you set `push.autoSetupRemote`).

### 5. From here on: work on branches

On a team, you'll almost never commit directly to `main` after the initial setup. Each change goes on its own branch and arrives in `main` through a pull request. That's the workflow in [[06 Team Workflow and Pull Requests]].

---

## Joining an existing project: `git clone`

```sh
git clone git@github.com:company/project.git
cd project
git status
```

`clone` downloads the **entire history**, creates the `origin` remote automatically, and checks out the default branch with tracking already set up. Then follow the README's setup (usually `npm install`, copying `.env.example` to `.env`, and so on).

Useful variants:
- `git clone <url> my-folder-name`: clone into a differently named folder
- `git clone --depth 1 <url>`: only the latest snapshot, no history (fast for huge repos or CI, but limits what you can do)

---

## Fixing `.gitignore` mistakes

### "I added a pattern to `.gitignore` but the file still shows up"

`.gitignore` only affects **untracked** files. If the file was already committed, Git keeps tracking it. Tell Git to stop tracking it **without deleting your local copy**:

```sh
git rm --cached .env              # stop tracking, keep the file on disk
git rm -r --cached node_modules   # same for a folder
git commit -m "Stop tracking .env and node_modules"
```

> [!danger] If the file contained secrets
> Untracking doesn't remove the file from **history**: anyone can still check out an old commit and read it. If you ever pushed a real secret, **rotate the secret immediately** (generate a new key, revoke the old one). That's the actual fix. Rewriting history is optional cleanup. See [[11 Recovery Cookbook#I committed a secret]].

### "Why is this file being ignored?"

```sh
git check-ignore -v path/to/file
```

prints which `.gitignore` file and line is matching it.

---

## Quick self-test
- [ ] What does `git init` create, and what happens if you delete it?
- [ ] Why create `.gitignore` before the first commit? What should never be committed?
- [ ] Should `package-lock.json` be committed? Why?
- [ ] What does `-u` do in `git push -u origin main`?
- [ ] Why create the GitHub repo without a README when you already have local commits?
- [ ] You added `.env` to `.gitignore`, but it still appears in `git status` as modified. Why, and what's the fix?
- [ ] Explain what `pull.rebase`, `merge.conflictStyle zdiff3` and `push.autoSetupRemote` each fix.

Next: [[03 The Everyday Loop]]
