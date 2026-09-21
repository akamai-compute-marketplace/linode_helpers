# Development Guide

`linode_helpers` is the shared library of Ansible roles consumed as a **git submodule**, at path `apps/linode_helpers`, by:

- [`marketplace-apps`](https://github.com/akamai-compute-marketplace/marketplace-apps)
- [`marketplace-clusters`](https://github.com/akamai-compute-marketplace/marketplace-clusters)

Each app/cluster playbook references helper roles via a relative `roles_path` (e.g. `roles_path=./roles:../linode_helpers/roles` in its `ansible.cfg`). Because a submodule pins an exact commit, changes made here are **not** automatically picked up by consumers — you must explicitly update the pinned reference in each consumer repo after merging here.

## Overview of the flow

```
linode_helpers (this repo)
   1. branch -> edit role -> PR -> merge to main
   2. tag a release (vX.Y.Z)
         |
         v
   marketplace-apps / marketplace-clusters
   3. bump apps/linode_helpers
      to vX.Y.Z -> PR -> merge
```

Steps 1–2 must always happen first and be merged before either consumer repo is bumped. The two consumer-repo bump PRs (step 3) are independent of each other and can be done in parallel, in any order.

## Step 1: Make the change in `linode_helpers`

```bash
git checkout main && git pull
git checkout -b fix/some-role-change
# ... edit roles/<role>/tasks/*.yml ...
git add -A
git commit -m "fix: description of the change"
git push -u origin fix/some-role-change
```

Open a PR against `main` in `linode_helpers`, get it reviewed, and merge it.

## Step 2: Tag a release

Once merged, tag the release on `main` so consumers pin to an immutable, human-readable version instead of a floating commit:

```bash
git checkout main && git pull
git tag -a v1.3.0 -m "Short description of what changed"
git push origin v1.3.0
```

Use semantic versioning:
- **MAJOR** – breaking changes (renamed roles/variables, removed behavior)
- **MINOR** – new roles or backward-compatible additions
- **PATCH** – bug fixes with no interface changes

## Step 3: Bump the submodule in each consumer repo

Repeat this for `marketplace-apps` and then `marketplace-clusters` (order between the two doesn't matter):

```bash
cd marketplace-apps # or marketplace-clusters
git checkout develop && git pull
git checkout -b chore/bump-linode-helpers-v1.3.0

cd apps/linode_helpers
git fetch --tags
git checkout v1.3.0
cd ../..

git add apps/linode_helpers
git commit -m "chore: bump linode_helpers to v1.3.0"
git push -u origin chore/bump-linode-helpers-v1.3.0
```

Open a PR against `develop` in the consumer repo, referencing the `linode_helpers` PR/tag in the description, then get it reviewed and merged.

### Finding which apps are affected

Before opening the bump PR, check which apps actually use the role(s) you changed, so you know what to spot-check:

```bash
grep -rl "name: <role-name>" apps/*/site.yml apps/*/roles 2>/dev/null
```

## Validation checklist (before merging a bump PR)

- [ ] Confirm the submodule now points at the intended tag: `git -C apps/linode_helpers describe --tags`
- [ ] For each affected app/cluster found above, review its task files to confirm variable names and expected behavior still match the updated role
- [ ] Run a development workflow in the consumer repo on the bump PR for affected apps/clusters, and confirm the playbook runs successfully and the app/cluster is healthy afterward

## Rollback procedure

If a bumped version causes issues after merge, revert the gitlink commit in the consumer repo — no changes to `linode_helpers` itself are needed:

```bash
git checkout develop && git pull
git checkout -b revert/linode-helpers-v1.3.0
git revert <bump-commit-sha>
git push -u origin revert/linode-helpers-v1.3.0
```

Open a PR against `develop` to merge the revert, then investigate the issue in `linode_helpers` before re-attempting the bump.
