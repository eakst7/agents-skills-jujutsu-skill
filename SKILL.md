---
name: jujutsu-skill
description: Safe day-to-day version control with Jujutsu (`jj`) in a repository already colocated with Git (both `.jj/` and `.git/` in the same working copy). Use this skill whenever the user is working in a jj+git colocated repo — for status/log/diff, creating and describing changes, bookmarks versus Git branches, rebasing/squashing/splitting/abandoning commits, fetching and pushing, or recovering from a messy state — even if they never say "colocated" or "skill". Only for repos that are already colocated: do NOT use for initial setup, enabling or turning on colocation in a non-colocated repo, or general Git questions. Default to `jj` for all repository changes; only fall back to `git` when jj genuinely cannot do the job, and warn the user before running any mutating git command. Also covers "jj and git in the same repo", "detached HEAD under jj", "jj bookmarks vs branches", and "undo a jj operation".
---

# Jujutsu in a colocated Git repository

This skill is for routine work in a workspace where `.jj/` and `.git/` live side by side and share one working copy. The goal is to keep changes safe and reversible while collaborating with Git users and tools.

## Start here: confirm the situation

- Check `jj git colocation status` and that both `.jj/` and `.git/` exist at the repository root.
- If jj is not initialized yet, do not assume — ask the user before running `jj git init` / `jj git clone`, since that changes how the repo is managed.
- In a colocated workspace, **jj imports from and exports to Git automatically on every jj command.** Do not routinely run `jj git import` or `jj git export` here; they are for non-colocated workspaces.

## The mental model that matters

- The working copy is itself a commit, `@`. Files are snapshotted automatically; there is no staging area and no "dirty/untracked" concept in jj.
- There is no "current branch." Git therefore usually sees a **detached HEAD** — this is normal, not a problem to fix.
- **Bookmarks** are jj's equivalent of Git branches, but they do **not** advance automatically when you add commits. Remote bookmarks like `main@origin` are immutable snapshots refreshed by `jj git fetch`.
- A **change ID** stays stable across rewrites; the commit ID changes. Bookmarks follow change IDs, which is why rewriting is safe.
- Almost every operation is recorded in the operation log and can be undone.

## Golden rule: prefer jj; only use git as a last resort — and warn first

- Use `jj` for every repository-mutating operation.
- Read-only Git commands (`git status`, `git log`, `git show`, `git diff`, `git remote -v`) are fine for inspection.
- For **any other Git command**, warn the user *before* running it:
  1. why jj cannot do the task,
  2. the exact Git command you intend to run, and
  3. that it can desync jj's view of branches, and you will re-sync afterwards.
- Before a mutating Git command, give Git a branch with `git switch <branch>` so HEAD is not detached. Afterward run a `jj` command (e.g. `jj status`) to re-import Git state; confirm `jj op log` shows an "import git refs" operation.
- Never reach for `git rebase`, `git commit`, `git reset --hard`, `git cherry-pick`, or `git add` when a jj equivalent exists (see below).

## Safe day-to-day loop

1. **Orient:** `jj status`, `jj log` (or `jj log -r 'remote_bookmarks()..@'` for your unpushed work), `jj diff`, `jj bookmark list`.
2. **Work:** edit files freely. `jj describe -m "..."` names the current change; `jj commit -m "..."` seals it and starts a fresh `@`; `jj new` starts an empty change on top.
3. **Structure** the result with `jj squash`, `jj split`, `jj rebase`, `jj restore`, or `jj abandon` (see below).
4. **Sync:** `jj git fetch`, rebase your bookmarks onto the updated remote, then `jj git push`.
5. Inspect with `jj log` / `jj diff` before and after any rewrite. If something looks wrong, undo rather than forcing it.

## Bookmarks vs branches

In a colocated repo bookmarks map 1:1 to Git branches: creating or moving one via jj is reflected in Git. Treat them as deliberate labels for sharing work, not as a cursor that follows you.

- **Create:** `jj bookmark create <name> -r <rev>` (defaults to `@`). When pushing a PR, put the bookmark on the working copy's parent `@-`, since `@` is usually empty.
- **Track:** `jj bookmark track <name>@<remote>` so remote updates propagate to the local bookmark on `jj git fetch`. List with `jj bookmark list`.
- **Move:** `jj bookmark move <name> -t <rev>` (use `-B` / `--allow-backwards` if you must move it backwards). Because bookmarks do not auto-advance after `jj new` / `jj commit`, move them intentionally.
- **Push:** `jj git push -b <name>` for a specific bookmark, `-c <rev>` to let jj generate a name, or `--all`. Updates are force-safe (like `--force-with-lease`); if a safety check fails, run `jj git fetch` and resolve the bookmark conflict rather than overriding it.
- **New remote branches:** `jj git fetch` does not create local bookmarks for brand-new remote branches. Use `jj new <name>@<remote>`, or set `remotes.<remote>.auto-track-bookmarks = "*"` if the user wants them all.
- After a push, commits may become immutable and jj creates a new empty `@` automatically — that is expected.
- Avoid `git branch`, `git checkout -b`, and `git reset --hard` as a way to move branches.

## Rewriting history safely

Rewriting is cheap and recoverable in jj, but inspect first (`jj log`, `jj diff -r <rev>`) and keep shared bookmarks in mind.

- **Reword:** `jj describe -m "..." <rev>`.
- **Fold into parent:** `jj squash` (working copy → parent), `jj squash --into <rev>`, or `jj squash -r <rev>`.
- **Split:** `jj split -r <rev>` opens a diff editor; `jj commit -i` splits interactively. `-p` makes the two parts siblings.
- **Move changes:** `jj rebase -b <bookmark> -o <dest>` moves a bookmark's branch; `-s` includes descendants; `-r` moves one revision without descendants; `-A` / `-B` insert between commits.
- **Discard / reset:** `jj abandon <rev>` removes a change and rebases its descendants onto its parent; `jj restore <paths>` resets file contents from the parent (or `--from` / `--into` a chosen revision). `jj new` keeps history instead of deleting it.
- **Conflicts:** after a rebase, resolve the conflicted files in the working copy, review with `jj diff`, then continue. Check `jj log` for divergent change IDs and converge them if needed.
- **Pushed bookmarks:** rewriting history behind a shared bookmark needs a force push; `jj git push` handles it safely. Warn the user if collaborators may be relying on the old commits.
- Never use `git rebase` as a substitute — besides losing jj's safety net, it does not preserve jj change-ID headers.

## Syncing with a remote

There is no `jj pull`. The equivalent is two steps:

- `jj git fetch` — refreshes remote bookmarks (e.g. `main@origin`).
- `jj rebase -b <your-bookmark> -o main@origin` — replay your commits on the updated base (repeat with multiple `-b` values for several bookmarks).
- `jj git push` — push tracked bookmarks; use `-b`, `-c`, or `--all` to be explicit.

## Colocated-repo pitfalls

- Do not "fix" detached HEAD with `git checkout` unless you are about to run a mutating Git command.
- IDEs that run `git fetch` in the background can cause bookmark conflicts or divergent change IDs; minimize interleaving mutating Git and jj commands.
- Git cannot represent jj conflicts; it sees `.jjconflict-*` directories. If you accidentally snapshot them with a Git tool, use `jj abandon` to return to the unresolved-conflict state.
- jj ignores Git's staging area and unfinished Git rebase states; do not rely on either.
- With very many refs, automatic imports slow jj down; occasional `jj util gc` helps.
- If a newly ignored file was already snapshotted, run `jj file untrack <paths>` (paths must already be ignored).
- jj's change-ID header survives `git commit --amend` but not `git rebase` — another reason to stay in jj.

## Recovery

- `jj undo` steps back through operations; `jj redo` moves forward.
- `jj op log` shows history; preview a state with `jj --at-op=<id> log`, then `jj op restore <id>` to return to it.
- Mutating Git commands appear as "import git refs" operations and are undoable the same way.
- If a bookmark ends up in the wrong place after mixing Git and jj, prefer `jj bookmark move` or `jj op restore` over another Git mutation.

## Detailed reference

For a command-by-command Git→jj mapping and fuller recipes, read `references/jj-git-cheatsheet.md`.
