# jj ↔ Git cheatsheet (colocated workspace)

Read this when you need the exact command for a specific Git habit or a fuller recipe. Commands verified against jj 0.43.0.

## Core facts

- Colocated repo: `.jj/` and `.git/` at the same root, shared working copy. `jj git colocation status` checks this.
- jj auto-imports/auto-exports Git on every jj command. `jj git import` / `jj git export` are **not** needed here.
- `@` = working-copy commit; `@-` = its parent; `@--` = grandparent. `main@origin` = remote bookmark snapshot.
- Bookmarks follow change IDs across rewrites; they do not auto-advance.

## Git → jj mapping

| Git operation | jj equivalent |
|---|---|
| `git status` | `jj status` / `jj log` |
| `git log --oneline` | `jj log --no-graph` (or `jj log`) |
| `git diff` | `jj diff` / `jj diff -r <rev>` |
| `git add` + `git commit -m` | edit files, then `jj commit -m "..."` (or `jj describe -m "..."` to keep `@`) |
| `git commit --amend` | `jj describe -m "..."` (amends `@` in place) |
| `git checkout -b name` | `jj new <base>` then `jj bookmark create name -r @-` |
| `git checkout <branch>` | `jj new <bookmark>` (creates a new change on it) |
| `git switch <branch>` (only before a Git mutation) | give Git a HEAD; jj itself has no current branch |
| `git branch -f name <rev>` | `jj bookmark move name -t <rev>` (`-B` to allow backwards) |
| `git reset --soft HEAD~1` | `jj new @-` (keep changes in working copy) |
| `git reset --hard` | `jj abandon <rev>` or `jj restore <paths>` |
| `git revert <rev>` | `jj new <rev>` + `jj restore --from <rev>@-` (or cherry-pick equivalent) |
| `git cherry-pick <rev>` | `jj rebase -r <rev> -o <dest>` (rebase a single revision) |
| `git rebase -i` | `jj rebase` + `jj squash` / `jj split` / `jj abandon` interactively |
| `git stash` | not needed; every change is a commit — use `jj new` and bookmarks |
| `git pull` | `jj git fetch` then `jj rebase -b <bookmark> -o main@origin` |
| `git push` | `jj git push` (`-b`, `--all`, `-c <rev>`) |
| `git fetch` | `jj git fetch` |
| `git reflog` / `git reset --hard ORIG_HEAD` | `jj op log` / `jj undo` / `jj op restore <id>` |
| `git blame` | `jj file annotate` |
| `git bisect` | no direct equivalent; use Git if needed (warn first) |

## Recipes

### Inspect your unpushed work
```
jj log -r 'bookmarks() & ~(main | remote_bookmarks())'
jj log -r 'remote_bookmarks()..@'
```

### Start a change and push a PR without naming a bookmark
```
jj new main@origin
# edit files
jj commit -m "feat: do the thing"
jj git push -c @-
```
`-c` generates a bookmark from the change ID and pushes it; jj tracks the new remote bookmark.

### Named bookmark workflow
```
jj new main@origin
# edit files
jj commit -m "feat: do the thing"
jj bookmark create my-feature -r @-
jj bookmark track my-feature
jj git push -b my-feature
```

### Fold follow-up fixes into an earlier commit (clean-history / stacked PRs)
```
jj new my-feature-          # revset suffix '-' = parent of my-feature
# edit files
jj squash                   # working copy into its parent
jj git push -b my-feature   # jj performs a safe force-push
```

### Address review comments as new commits
```
jj new my-feature
# edit files
jj commit -m "address review comments"
jj bookmark move my-feature -t @-
jj git push -b my-feature
```

### Update after collaborators push
```
jj git fetch
jj rebase -b my-feature -o main@origin    # repeat -b for each bookmark
jj git push
```
If the remote moved unexpectedly, fetch again and resolve the bookmark conflict; do not override the safety check.

### Recover from a bad rewrite or a Git command gone wrong
```
jj op log                                  # find the operation before the mistake
jj --at-op=<id> log                        # preview that state
jj op restore <id>                         # return to it
jj undo / jj redo                          # step back / forward
```
Git mutations show up as "import git refs" operations in `jj op log`.

### Newly ignored file got snapshotted
```
jj file untrack <paths>                    # paths must already be ignored
```

## Revset quick reference

- `@` working-copy commit, `@-` parent, `@--` grandparent.
- `<bookmark>-` parent of a bookmark; `<bookmark>+` child.
- `<name>@<remote>` remote bookmark; `main@origin`.
- `remote_bookmarks()..@` commits in your working copy not on any remote.
- `bookmarks() & ~remote_bookmarks()` local bookmarks not pushed.

## When Git is genuinely required

Use Git only for capabilities jj lacks (for example some exotic plumbing or tool-specific operations), warn the user first, give Git a branch with `git switch <branch>`, then re-sync with `jj status`. Never use `git rebase` where `jj rebase` exists — Git rebase also drops jj's change-ID header.
