# Sync external and internal Git repositories

Clone the external repo once, work with your internal copy, and share changes
in either direction.

## Basic

Your local clone will have two connections, called **remotes**:

- **`upstream` = external repo**
- **`origin` = internal repo**

These commands use **PowerShell** and the `main` branch. You'll need Git and
access to the repos. Replace `main` if your branch has a different name.
Run commands in order and **stop if one fails**.

### 1. Set up once

These steps assume your **internal repo already exists and is empty** (no
commits).

Replace `EXTERNAL_URL` with your external repo's HTTPS or SSH clone URL.
The internal URL below is an example; replace it if using a different repo.
Do not include passwords or tokens in URLs.

```powershell
git clone 'EXTERNAL_URL' project-sync
Set-Location .\project-sync

git remote rename origin upstream
git remote add origin 'https://github.com/rutzsco/git-sync-internal.git'
git config remote.pushDefault origin
git switch main
git remote -v
```

Check that `upstream` points to external and `origin` points to internal.
Then copy `main` and its history to internal:

```powershell
git push --set-upstream origin main
```

Setup is done. Internal is now the default repo for your local `main`.

### 2. Get external updates into internal

Before syncing, run `git status`. Commit or stash unfinished work so your
working tree is clean.

Get the latest changes from both repos and combine them locally:

```powershell
git switch main
git fetch origin
git fetch upstream
git merge origin/main
git merge upstream/main
```

`fetch` downloads changes; `merge` combines them. Including `origin/main` first
keeps work your teammates have already pushed internally.

Review the result and run your project's tests, then update internal:

```powershell
git push origin main
```

That's it: internal now includes the external updates. External is unchanged.
If direct pushes are blocked, use [pull requests](#protected-branches-and-pull-requests).

### 3. Send internal changes back to external

Make changes through your normal internal workflow. When ready to share them,
**run step 2 again first** so local `main` includes the latest from both repos.

> **Check before publishing:** This sends the branch's unpublished history,
> including older file versions, to external. If internal contains private work,
> use [selected commits](#publish-only-selected-commits) instead.

Review what external will receive:

```powershell
git log --oneline upstream/main..main
git diff upstream/main...main
```

Inspect the outgoing commits for private content, not just the final diff.
If everything is approved for external publication:

```powershell
git push upstream main
```

**Your regular routine:** repeat step 2 to import updates; repeat steps 2 and 3
to sync both ways. Syncing happens when you run these commands, not automatically.

## Advanced (only when needed)

### Protected branches and pull requests

If a repo requires pull requests, push your integrated local `main` to a new
branch instead of pushing directly to its `main`. Choose the destination:

| Destination | Command |
| --- | --- |
| Internal | `git push origin main:refs/heads/sync/from-external` |
| External | `git push upstream main:refs/heads/sync/from-internal` |

Open a pull request from that branch into the destination's `main`. Use a fresh
branch name each time. The destination's `main` changes only after the PR merges.
Then fetch and repeat step 2 before the next synchronization.

**Review private content before pushing an external PR branch**, not just before
merging the PR. If you lack external write access, push to your fork and open a
PR from there. Prefer merge commits or fast-forward merges when policy allows
to retain shared history; squash/rebase merges can change commit IDs.

### Push rejected or merge conflicts

- **Non-fast-forward rejection:** repeat step 2 to include new remote commits,
  review and test again, then retry the required push. Do not force-push.
- **Authentication failure:** check your access and credentials.
- **Merge conflict:** run `git status`, edit the conflicted files, remove the
  conflict markers, and finish the merge:

  ```powershell
  git add -- 'path\to\resolved-file'
  git merge --continue
  ```

Stage each resolved file, then continue. To cancel instead, use
`git merge --abort`. For a cherry-pick, use `git cherry-pick --continue` or
`git cherry-pick --abort`. Review and test the result before pushing.

A failed push does not undo a push that already succeeded to the other repo.

### Publish only selected commits

When some internal work must stay private, start a branch from external and
copy only approved commits onto it:

```powershell
git fetch origin
git fetch upstream
git log --oneline upstream/main..origin/main
git switch -c publish/my-change upstream/main
```

Choose a reviewed, non-merge commit from the log and replace `COMMIT_SHA` below.
Repeat cherry-picks in dependency order if needed. If a commit mixes private and
public changes, prepare a sanitized change instead.

```powershell
git cherry-pick COMMIT_SHA
git log --oneline upstream/main..HEAD
git diff upstream/main...HEAD
```

Review every outgoing commit and run tests. **Do not merge internal `main` into
this branch**, because that would include its private history. When approved:

```powershell
git push upstream HEAD:refs/heads/publish/my-change
```

Open an external PR into `main` (use a fork if needed). After it merges, run
step 2. Cherry-picking creates new commits, so the repos will not have identical
histories; private work stays internal.

### Check that both repos match

After a full two-way sync:

```powershell
git fetch origin
git fetch upstream
git rev-parse main origin/main upstream/main
```

With no concurrent changes, all three commit IDs should match. They may differ
after selective publishing, a one-way import, or squash/rebase PR merges.

### Scope and safety

- These steps sync `main`, not every branch or tag. Repeat for other branches
  as needed; push tags explicitly, such as `git push origin tag v1.2.3`.
  Review a tag's history before publishing it externally.
- Avoid `--mirror`, `--force`, and `--force-with-lease` for routine syncing;
  they can delete or overwrite destination history.
- Git does not copy issues, PRs, settings, permissions, or release assets.
  Submodules and Git LFS objects may need separate handling.
