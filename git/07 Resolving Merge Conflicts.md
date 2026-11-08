---
tags: [git, merge-conflicts]
aliases: [Merge Conflict, Conflict Markers, ours theirs, git mergetool]
---
# Resolving Merge Conflicts

> [!summary] What you'll learn
> A merge conflict is **not an error and not data loss**. It's Git saying "two people changed the same lines differently, and I won't guess which one is right." Nothing is overwritten: both versions are sitting in the file, and you can **abort at any time** to go back to exactly where you started.
>
> This note covers why conflicts happen, how to **read conflict markers** (including the much clearer `zdiff3` style), a repeatable **resolution procedure**, and worked scenarios: a basic code conflict, a PR that conflicts with `main`, conflicts during a rebase (where "ours" and "theirs" swap), deleted-vs-modified files, both-added files, and `package-lock.json`. All terminal output here is from real runs.

Back to [[00 Git Index]] · Prev: [[06 Team Workflow and Pull Requests]] · Next: [[08 Rebasing]]

---

## Why conflicts happen

Recall the three-way merge ([[04 Branches and Merging#Case 2 - Three-way merge (merge commit)]]): Git compares each side against the **merge base** (the last commit both share).

- Only one side changed a region → Git takes that change automatically.
- Both sides changed **the same lines (or directly adjacent lines)** in different ways → **conflict**.
- Other structural clashes also conflict: one side deleted a file the other edited, or both sides created a file with the same name.

Conflicts can appear during **any operation that combines changes**: `merge`, `pull`, `rebase`, `cherry-pick`, `revert`, `stash pop`. The resolution procedure is the same for all of them; only the "continue" command differs.

> [!tip] Conflicts are about lines, not meaning
> Git only detects **textual** overlap. Two changes can merge cleanly and still break the code (you renamed a function while a teammate added a new call to its old name). That's why you **run the tests after every conflict resolution and every merge**.

---

## Your safety exits

At any point during a conflicted operation, you can back out completely:

| In the middle of... | Abort with | Result |
|---|---|---|
| `git merge` / `git pull` (merge) | `git merge --abort` | Back to before the merge, as if it never happened |
| `git rebase` / `git pull --rebase` | `git rebase --abort` | Back to your branch exactly as it was before the rebase |
| `git cherry-pick` | `git cherry-pick --abort` | Back to before the cherry-pick |
| `git revert` | `git revert --abort` | Back to before the revert |
| `git stash pop` / `apply` | `git reset --merge` (the stash is **kept** when pop conflicts) | Back to before the pop |

Not sure what you're in the middle of? `git status` tells you on the first lines ("You have unmerged paths", "interactive rebase in progress", and so on).

---

## Reading conflict markers

Setup (real run): `main` changed the greeting to "Hi", while `feature/excited` added an exclamation mark to the same line.

```text
$ git merge feature/excited
Auto-merging greet.js
CONFLICT (content): Merge conflict in greet.js
Automatic merge failed; fix conflicts and then commit the result.
```

### Default style

```js
function greet(name) {
<<<<<<< HEAD
  return "Hi, " + name;
=======
  return "Hello, " + name + "!";
>>>>>>> feature/excited
}
module.exports = { greet };
```

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` | Start of **your current branch's version** ("ours": the branch you're merging *into*) |
| `=======` | Divider |
| `>>>>>>> feature/excited` | End of the **incoming version** ("theirs": the branch being merged) |

Everything outside the markers merged cleanly.

### `zdiff3` style (turn this on)

With `git config --global merge.conflictStyle zdiff3`, the same conflict shows **the original line too**:

```js
function greet(name) {
<<<<<<< HEAD
  return "Hi, " + name;
||||||| c9a6f53
  return "Hello, " + name;
=======
  return "Hello, " + name + "!";
>>>>>>> feature/excited
}
```

The middle section (`|||||||`) is the **merge base**: what the line looked like before either change. Now you can see each side's *intent*:
- Ours changed `Hello` → `Hi`.
- Theirs added `!`.

So the correct resolution is obviously **both changes**: `return "Hi, " + name + "!";`. Without the base section you'd have to guess or dig through the history. This is why `zdiff3` makes conflicts dramatically easier.

---

## The resolution procedure

Use this every time:

1. **Don't panic, and read `git status`.** It lists every conflicted file under "Unmerged paths":
   ```text
   You have unmerged paths.
     (fix conflicts and run "git commit")
     (use "git merge --abort" to abort the merge)

   Unmerged paths:
     (use "git add <file>..." to mark resolution)
   	both modified:   greet.js
   ```
2. **For each conflicted file, understand both sides.** Look at the markers. If the intent isn't obvious, check what each side was trying to do:
   ```sh
   git log --oneline --left-right HEAD...MERGE_HEAD -- greet.js   # commits from each side touching this file
   git show <sha>                                                  # read a commit's message and diff
   ```
   (During a rebase, use `REBASE_HEAD` instead of `MERGE_HEAD`. During a cherry-pick, `CHERRY_PICK_HEAD`.)
3. **Edit the file into the correct final state.** This might be ours, theirs, a combination, or something new that satisfies both intents. **Delete all marker lines.** Unsure about a business decision? Ask the teammate who wrote the other side.
4. **Check nothing is left:**
   ```sh
   git diff --check     # flags leftover conflict markers (and whitespace errors)
   grep -rn '<<<<<<<\|>>>>>>>' src/
   ```
5. **Run the code and the tests.** A clean textual merge can still be semantically broken.
6. **Mark the file resolved by staging it:** `git add greet.js`. Staging is how you tell Git "this file is resolved."
7. **Finish the operation:**

| Operation | Continue with |
|---|---|
| merge / pull (merge) | `git commit` (accept or edit the prepared message) |
| rebase / pull --rebase | `git rebase --continue` (may stop again at the next commit) |
| cherry-pick | `git cherry-pick --continue` |
| revert | `git revert --continue` |
| stash pop | Nothing to continue; just `git stash drop` once it's resolved, since a conflicted pop keeps the stash |

Resolved merge from the example:

```text
$ git add greet.js
$ git commit --no-edit
[main bcd1fea] Merge branch 'feature/excited'

$ git log --oneline --graph
*   bcd1fea Merge branch 'feature/excited'
|\
| * 8e35175 Add excitement to greeting
* | 383904f Use friendlier greeting
|/
* c9a6f53 Initial commit
```

> [!tip] Messed up a file mid-resolution?
> `git restore --merge greet.js` (older form: `git checkout -m greet.js`) **recreates the original conflict markers** in that file so you can start over, without aborting the whole merge.

---

## Taking one whole side

Sometimes you know one version is entirely right for a file (a generated file, or you know the other change is obsolete):

```sh
git restore --ours path/to/file      # keep the current branch's version
git restore --theirs path/to/file    # keep the incoming version
git add path/to/file
```

(Older form: `git checkout --ours file`.) Be careful: this discards **all** of the other side's changes to that file, not just the conflicted parts.

To pre-decide for an entire merge, `git merge -X ours feature` / `-X theirs` makes Git automatically pick a side **only for conflicting hunks** (non-conflicting changes from both sides still merge). Use it rarely and deliberately.

> [!warning] "ours" and "theirs" swap during a rebase
> - **Merge** (you're on `main`, merging `feature`): ours = `main` (HEAD), theirs = `feature`.
> - **Rebase** (you're on `feature`, rebasing onto `main`): Git first moves to `main`, then replays your commits one at a time. So **ours = `main`** (the base being built on) and **theirs = your commit being replayed**.
>
> In a rebase, `git restore --theirs file` keeps **your** change. Read the marker labels rather than trusting the words: in a rebase, the bottom label shows your commit's ID and message.

---

## Scenario 1 - A basic code conflict (merge)

Covered above: both branches edited the same line. Steps: read both sides (using the zdiff3 base to understand intent), combine, remove markers, test, `git add`, `git commit`.

## Scenario 2 - Several kinds of conflict at once

Real run: `feat` deleted `utils.js`, changed `config.js`, and created `api.js`. Meanwhile `main` edited `utils.js`, changed the same line of `config.js`, and also created an `api.js`.

```text
$ git merge feat
Auto-merging api.js
CONFLICT (add/add): Merge conflict in api.js
Auto-merging config.js
CONFLICT (content): Merge conflict in config.js
CONFLICT (modify/delete): utils.js deleted in feat and modified in HEAD.  Version HEAD of utils.js left in tree.
Automatic merge failed; fix conflicts and then commit the result.

$ git status
Unmerged paths:
  (use "git add/rm <file>..." as appropriate to mark resolution)
	both added:      api.js
	both modified:   config.js
	deleted by them: utils.js
```

How to resolve each type:

| Status | What happened | How to resolve |
|---|---|---|
| **both modified** | Same lines edited differently | Edit the markers as usual → `git add` |
| **both added** | Each side created a file at the same path | The file contains markers around the differing parts; combine → `git add` |
| **deleted by them** / **deleted by us** | One side deleted the file, the other edited it | Decide: should the file exist? **Keep it** (maybe reapplying the edit somewhere else if it was moved) → `git add file`. **Delete it** → `git rm file`. Find out *why* it was deleted first: `git log --oneline -1 --diff-filter=D -- utils.js` on the deleting branch |
| **both deleted** / renamed in different ways | Rename clashes | `git status` describes it; keep the path you want with `git add`, `git rm` the others |

For the `utils.js` case: if `feat` deleted it because the functions moved to a new file, the right move is to apply `main`'s edit to the new location, then `git rm utils.js`.

## Scenario 3 - My PR has conflicts with main

GitHub says **"This branch has conflicts that must be resolved."** Teammates merged changes into `main` that overlap with yours. You fix this **on your branch** (never on `main`), using whichever update method your team uses ([[06 Team Workflow and Pull Requests#4. Keep your branch up to date with main]]).

**With rebase (your own branch):**

```sh
git switch feature/password-reset
git status                        # make sure it's clean; commit or stash first
git branch backup/pw-reset        # optional safety label (cheap and reassuring)
git fetch
git rebase origin/main            # replays your commits on top of the latest main
# CONFLICT -> fix the files, then:
git add <files>
git rebase --continue             # repeat for each commit that conflicts
npm test                          # make sure the combined code works
git push --force-with-lease --force-if-includes
```

**With merge (shared branch, or team preference):**

```sh
git switch feature/password-reset
git fetch
git merge origin/main
# CONFLICT -> fix, git add, then:
git commit
npm test
git push
```

The PR updates automatically and the conflict warning disappears.

> [!info] Why the rebase version can stop several times
> A rebase replays your commits one at a time, so a conflict can appear at commit 1, and again at commit 3 if it touches the same area. A merge resolves everything once. If a rebase keeps stopping on the same lines, either squash your commits first (`git rebase -i` onto the merge base) or use a merge. `rerere` also helps here ([[#Let Git remember resolutions (rerere)]]).

You *can* resolve simple conflicts in GitHub's web editor, but that always creates a merge commit of `main` into your branch, and you can't run the tests. Doing it locally is better.

## Scenario 4 - Conflict during a rebase

Real run: `main` added a `shout()` function at the bottom of `greet.js`, while the feature branch added `bye()` in the same spot.

```text
$ git rebase main
Auto-merging greet.js
CONFLICT (content): Merge conflict in greet.js
error: could not apply 486d823... Add bye()
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
```

```js
function greet(name) {
  return "Hi, " + name + "!";
}
<<<<<<< HEAD
function shout(name) {
  return greet(name).toUpperCase();
}
module.exports = { greet, shout };
||||||| parent of 486d823 (Add bye())
module.exports = { greet };
=======
function bye(name) {
  return "Bye, " + name;
}
module.exports = { greet, bye };
>>>>>>> 486d823 (Add bye())
```

Notice the labels: `HEAD` is **main's code** (with `shout`), and the bottom section is **your commit** being replayed. The base shows that originally only `greet` was exported. Both sides added a function and an export, so the resolution keeps both:

```js
function greet(name) {
  return "Hi, " + name + "!";
}
function shout(name) {
  return greet(name).toUpperCase();
}
function bye(name) {
  return "Bye, " + name;
}
module.exports = { greet, shout, bye };
```

```sh
git add greet.js
git rebase --continue      # Git opens the editor for the commit message; save and close
```

If the next commit also conflicts, Git stops again. Repeat until you see `Successfully rebased and updated refs/heads/...`. `git status` during a rebase shows how many commits are done and remaining.

**`git rebase --skip`** drops the commit that's currently being applied. Use it only when that commit is no longer needed (for example, the same fix already landed on `main`). Otherwise you lose that commit's changes.

## Scenario 5 - `package-lock.json` (and other generated files)

Lockfiles conflict constantly when two branches both add dependencies, and hand-merging them is pointless and error-prone. **Regenerate instead:**

```sh
# during a merge or rebase with package-lock.json conflicted:
git restore --theirs package-lock.json   # or --ours; just pick a starting point
# make sure package.json itself is resolved correctly first (it's small; resolve by hand)
npm install                              # regenerates the lockfile so it matches package.json
git add package.json package-lock.json
git merge --continue                     # or: git rebase --continue
```

The same goes for other generated files (compiled output, snapshots, schema dumps): resolve the **source**, then regenerate the output.

## Scenario 6 - The conflict was resolved wrong and I already committed

- **Merge commit not pushed yet:** `git reset --hard ORIG_HEAD` (if you have no uncommitted work) goes back to before the merge, then merge again. Or just fix the code and make a normal new commit.
- **Already pushed:** fix it forward with a normal commit. If the whole merge was bad, revert it ([[11 Recovery Cookbook#A merged PR broke main]]).
- **During a rebase, you notice a mistake in an earlier step:** finish or abort the rebase (`--abort` returns everything to the original state), then redo it.

---

## Tools that make conflicts easier

### VS Code

Conflicted files show **"Accept Current Change | Accept Incoming Change | Accept Both Changes | Compare Changes"** buttons above each conflict, and there's a 3-way **Merge Editor** ("Resolve in Merge Editor") that shows incoming, current and result side by side. Current = ours, Incoming = theirs (with the swap during a rebase). After resolving, stage the file in the Source Control panel, then continue the operation from the terminal.

`git config --global merge.tool vscode` and `git config --global mergetool.vscode.cmd 'code --wait --merge $REMOTE $LOCAL $BASE $MERGED'` let `git mergetool` open each conflicted file in turn.

### Let Git remember resolutions (rerere)

With `git config --global rerere.enabled true`, Git records how you resolved each conflict. When the exact same conflict appears again (very common when you rebase a long branch several times, or abort and redo a merge), Git applies your previous resolution automatically and prints `Resolved 'file' using previous resolution.` You still review and `git add` it.

---

## Preventing painful conflicts

- **Integrate often:** rebase or merge `main` into your branch daily. Small conflicts are easy; a week of drift is not.
- **Keep PRs small and short-lived.**
- **Don't mix big reformatting with logic changes.** Running Prettier on a whole file conflicts with everyone. Do formatting in its own PR, coordinated with the team.
- **Communicate:** if you're about to refactor a shared module, tell the team.
- **Use the same formatter and line endings** (Prettier, `.editorconfig`, `.gitattributes` with `* text=auto eol=lf`) so whitespace-only differences don't conflict.

---

## Quick self-test
- [ ] Why is a merge conflict not data loss? How do you get out of one completely?
- [ ] Read a zdiff3 conflict: what are the three sections, and why does the middle one help?
- [ ] List the full resolution procedure, ending with the right "continue" command for merge, rebase and cherry-pick.
- [ ] During a rebase of `feature` onto `main`, which side is "ours"? What does `git restore --theirs` keep?
- [ ] One branch deleted a file, the other modified it. What are your options and commands?
- [ ] How should you resolve a `package-lock.json` conflict?
- [ ] Your PR shows "This branch has conflicts." Give the commands to fix it with a rebase, and with a merge.
- [ ] Two branches merged with no conflicts but the app broke. How is that possible?

Next: [[08 Rebasing]]
