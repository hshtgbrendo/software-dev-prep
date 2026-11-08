---
tags: [git, practice]
aliases: [Git Lab, Git Exercises]
---
# Practice Lab

> [!summary] What you'll do
> Reading about conflicts and rebases only gets you so far. Confidence comes from **causing the problems on purpose in a throwaway repo** and fixing them. This lab sets up a fake "GitHub" and a fake teammate on your own machine, then walks through eight exercises: merge conflict, rebase conflict, squashing commits, recovering from `reset --hard`, moving commits off `main`, a rejected push, reverting, and stash plus cherry-pick.
>
> Everything happens inside `~/git-lab`, which you delete at the end. **Nothing here can touch your real projects.** Every command has been run end to end to confirm it behaves as described.

Back to [[00 Git Index]] · Prev: [[12 Git Cheat Sheet]]

> [!tip] How to use this lab
> Type the commands rather than pasting them; it builds muscle memory. After each command, run `git status` and `git log --oneline --graph --all` and predict the output before you look. Each exercise links to the note that explains it.

---

## Setup: a fake GitHub and a fake teammate

```sh
mkdir -p ~/git-lab && cd ~/git-lab
git init --bare github.git          # plays the role of GitHub: a repo with no working files
git clone github.git me             # your machine
git clone github.git teammate       # your teammate's machine
cd me
printf 'function greet(name) {\n  return "Hello, " + name;\n}\nmodule.exports = { greet };\n' > greet.js
echo "node_modules/" > .gitignore
git add .
git commit -m "Initial commit"
git push -u origin main
cd ../teammate && git pull && cd ../me
```

(The clones may warn that you cloned an empty repository. That's expected.) You now have three repos: `github.git` (the shared remote), `me` and `teammate`. Use `cd ~/git-lab/me` and `cd ~/git-lab/teammate` to switch "computers."

---

## Exercise 1 - Merge conflict

Reference: [[07 Resolving Merge Conflicts]]

```sh
cd ~/git-lab/me
git switch -c feature/excited
sed -i 's/"Hello, " + name;/"Hello, " + name + "!";/' greet.js
git commit -am "Add excitement"
git switch main
sed -i 's/"Hello, " + name;/"Hi, " + name;/' greet.js
git commit -am "Friendlier greeting"
git merge feature/excited          # CONFLICT
git status                         # read it: "both modified: greet.js"
cat greet.js                       # read the markers (zdiff3 shows the base too, if configured)
```

**Your task:** make the line `return "Hi, " + name + "!";` (both intents) and delete the markers, in your editor or with:

```sh
printf 'function greet(name) {\n  return "Hi, " + name + "!";\n}\nmodule.exports = { greet };\n' > greet.js
git diff --check                   # no output = no leftover markers
git add greet.js
git commit --no-edit
git log --oneline --graph
```

**Bonus:** before resolving, try `git merge --abort` and confirm everything is back to normal. Then merge again.

---

## Exercise 2 - Rebase with a conflict

Reference: [[08 Rebasing#Resolving conflicts during a rebase]]

```sh
git switch -c feature/bye
printf '\nfunction bye(name) {\n  return "Bye, " + name;\n}\n' >> greet.js
git commit -am "Add bye()"
git switch main
printf '\nfunction shout(name) {\n  return greet(name).toUpperCase();\n}\n' >> greet.js
git commit -am "Add shout()"
git switch feature/bye
git rebase main                    # CONFLICT: both appended at the end of the file
git status                         # "interactive rebase in progress", what's done and what's next
```

**Your task:** keep both functions. Notice that `HEAD` in the markers is **main's** code and the bottom section is **your** commit (the ours/theirs swap).

```sh
printf 'function greet(name) {\n  return "Hi, " + name + "!";\n}\nmodule.exports = { greet };\n\nfunction shout(name) {\n  return greet(name).toUpperCase();\n}\n\nfunction bye(name) {\n  return "Bye, " + name;\n}\n' > greet.js
git add greet.js
git rebase --continue              # save and close the message editor
git log --oneline --graph --all    # feature/bye now sits on top of main in a straight line
```

---

## Exercise 3 - Squash messy commits with interactive rebase

Reference: [[08 Rebasing#Interactive rebase for cleaning up your commits]]

```sh
git switch -c feature/login main
echo "form" > login.js && git add login.js && git commit -m "Add login form"
echo "wip" >> login.js && git commit -am "WIP"
echo "typo fixed" >> login.js && git commit -am "fix typo"
echo "validation" >> login.js && git commit -am "Add validation"
git log --oneline main..HEAD       # 4 commits
git rebase -i main
```

**Your task:** in the editor, change the `WIP` and `fix typo` lines from `pick` to `fixup`, then save and close. Check:

```sh
git log --oneline main..HEAD       # 2 commits: "Add login form", "Add validation"
cat login.js                       # all four lines are still there: no code lost, just fewer commits
```

---

## Exercise 4 - Disaster: reset --hard, then recover

Reference: [[09 Undoing Things Safely#The reflog is your safety net]]

```sh
git reset --hard HEAD~2            # "oops": both login commits are gone from the branch
git log --oneline -3               # they're not in the log...
git reflog -5                      # ...but the reflog remembers. Find "rebase (finish)" or the entry before the reset
git reset --hard HEAD@{1}          # back to where HEAD was one move ago
git log --oneline main..HEAD       # both commits are back
```

**Bonus:** delete the branch with `git switch main && git branch -D feature/login`. Note the `(was abc1234)` in the output, then bring it back with `git branch feature/login abc1234`.

---

## Exercise 5 - Committed to main by mistake

Reference: [[11 Recovery Cookbook#I committed to main instead of a branch]]

```sh
git switch main
git push                           # sync main with "GitHub" first
echo "// new feature" >> greet.js
git commit -am "Start new feature" # oops, this should have been on a branch
git branch feature/new             # 1. label the work
git reset --hard origin/main       # 2. move main back to match the remote
git switch feature/new             # 3. the commit is here
git log --oneline -2
```

---

## Exercise 6 - Rejected push, and pull --rebase

Reference: [[05 Remotes - Fetch, Pull, Push#Why pushes get rejected]]

```sh
cd ~/git-lab/teammate
git pull
echo "# Greeter" > README.md
git add README.md && git commit -m "Add README"
git push                           # teammate pushes first

cd ~/git-lab/me
git switch main
echo "console.log('start');" > index.js
git add index.js && git commit -m "Add entry point"
git push                           # REJECTED (fetch first)
git pull --rebase                  # replay your commit on top of the teammate's
git log --oneline -3               # your commit now comes after "Add README"
git push                           # succeeds
```

**Bonus:** before `pull --rebase`, try plain `git pull --no-rebase` and look at the merge commit it creates. Undo it with `git reset --hard ORIG_HEAD`, then do it the rebase way.

---

## Exercise 7 - Revert a pushed commit

Reference: [[09 Undoing Things Safely#git revert - undo a commit by adding a new one]]

```sh
git log --oneline -3
git revert --no-edit HEAD          # undo "Add entry point" with a new commit
git log --oneline -3               # original and revert both in history
ls                                 # index.js is gone from the working tree
git push                           # safe: no history was rewritten
```

---

## Exercise 8 - Stash and cherry-pick

Reference: [[10 Stash, Cherry-pick and Other Tools]]

```sh
git switch feature/new
echo "// half-done helpers" > helpers.js
git stash push -u -m "half-done"   # -u because helpers.js is new (untracked); working directory is now clean
git switch main                    # switch freely
git switch feature/new
git stash pop                      # work is back
git add helpers.js
git commit -m "Add helpers"

git switch main
git cherry-pick feature/new        # copy just that one commit onto main
git log --oneline -2               # same message, different ID from the one on feature/new
```

---

## Cleanup

```sh
cd ~ && rm -rf ~/git-lab
```

Repeat any exercise until you can do it without looking. Then try making up your own variations: a conflict across two files, a delete-vs-modify conflict, a three-commit rebase where two commits conflict.

---

## Self-check
- [ ] I can resolve a merge conflict, and I know how to abort one.
- [ ] I can rebase onto main, resolve a conflict mid-rebase, and continue.
- [ ] I can squash and reword commits with `git rebase -i`.
- [ ] I recovered "lost" commits with the reflog without looking at the notes.
- [ ] I can move accidental commits off `main`.
- [ ] I know why a push gets rejected and fixed it with `pull --rebase`.
- [ ] I know when to revert vs reset.
