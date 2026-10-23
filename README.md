# memex-git

A [memex] adapter for the `code` role in git. The role is specified in
memex's [code.md].

## Reaching the server

REPO is `git rev-parse --show-toplevel` of the current directory. ADDR
is the URL of one of REPO's remotes, compared as `remote.NAME.url` has
it; that remote is REMOTE. `versions`, `base` and `fetch` fail without
one. Upstream is REMOTE's remote-tracking branches; for `base`, its
`HEAD`.

Only `fetch` reaches the server, with prompts for passwords off and ssh
in batch mode. Nothing is pushed.

## Commands

    workspaces      git worktree list --porcelain, without bare and
                    prunable entries
    commits RANGE   git log RANGE
    versions VALUE  git log --fixed-strings --grep=VALUE over every
                    worktree's HEAD, --branches and --remotes=REMOTE,
                    kept when the Change-Id trailer is VALUE
    diff COMMIT     git diff-tree -r -M --name-status against the first
                    parent, or the empty tree; then diff-tree -p per file
    compare OLD NEW git range-diff OLD NEW, paired as below
    base RANGE      git merge-base --octopus of the ends of RANGE; of
                    COMMIT and REMOTE/HEAD for a single COMMIT
    fetch WHAT      git fetch --no-tags REMOTE WHAT, then FETCH_HEAD

`versions` searches only what REPO has fetched: git can't ask a Gerrit
server for the commits of a Change-Id. `fetch` refuses a WHAT with `:`,
so it writes no local ref but FETCH_HEAD. Arguments starting with `-`
are refused.

`capabilities` prints `{"local": true, "remote": true, "exact": false}`:
`remote` means that `fetch` works.

## Workspace

- `id` is the path.
- `upstream`: the branch's `@{upstream}`, e.g. `origin/master`.
- `changed`: `git status --porcelain --untracked-files=no` prints
  something; untracked files are no change.
- `basis`: head, branch, upstream and changed.
- `links`: `commit`, the head.

## Commit

- `id` is the commit id.
- IDENTITY is `["commit", CHANGE_ID]` from the first Change-Id trailer.
- `trailers` come from `%(trailers:only,unfold)`.
- `author.me` is true when its email is REPO's `user.email`.
- `authored` and `committed` are `%aI` and `%cI`.
- `where` is empty for `commits`, ADDR for `fetch`. For `versions` and
  `base`: the worktree paths whose HEAD has the commit, then the local
  branches and REMOTE's branches that have it, by short name.
- `basis`: the commit id, then `where`.

## File

- `id` is `COMMIT:PATH`; `basis` the commit id.
- `status`: A added, D deleted, R renamed, any other modified.
- `old_path` is set for renamed only.

## Pair

Commits of OLD and NEW with the same Change-Id pair first. range-diff
pairs the rest by their diffs, so `exact` is false. A pair's `status`
and `patch` come from `git range-diff` of its two commits alone, the
indent of its lines removed. Pairs come in NEW's order, base to tip,
then the commits NEW dropped. Merges are skipped, as range-diff does.

- `id` and `basis` are `OLD..NEW`, `-` for a side that is null.
- `links`: `commit` for each side and the `change-id`.

## Open questions

- Report `exact` per pair: Change-Id pairs are exact, range-diff pairs
  are not.
- `versions` on the server: query Gerrit for the Change-Id's patch sets
  and fetch them, or leave that to memex-gerrit and `fetch`?
- Should upstream be each branch's `@{upstream}` rather than REMOTE's
  branches?
- Match ADDR after `url.*.insteadOf` rewrites?

[memex]: https://github.com/roolebo/memex
[code.md]: https://github.com/roolebo/memex/blob/main/code.md
