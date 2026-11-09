# Hotfixing a previous major version

## When this applies

Use this procedure when a consumer is pinned to an older major version (for example `~> 3.0`), can't upgrade yet, and needs a bug or security fix in that major.

- Only bug and security fixes. Releases are patch releases only; no features.
- Only majors covered by the support policy. Consumers on unsupported majors must upgrade.

## Support policy

**Proposal (to be agreed):**

- Only the previous major (N-1) is supported.
- Only security fixes and blocking bugs.
- Support ends 6 months after the next major is released.
- Maintenance branches of unsupported majors are locked.

## Branch naming

- `release/v<major>`, for example `release/v3`.
- Created on demand from the latest tag of that major, and kept while the major is supported.
- Protected by the same ruleset as `main`: PR and required checks, no direct pushes.

## Procedure

### 1. Fix on main first

Open a normal PR to `main` with a conventional `fix:` title.

Include a regression test in `tests/*.tftest.hcl` (or a root `*.tftest.hcl`) using `mock_provider "azurerm"` and `command = plan`. The test must fail without the fix. `terraform validate` does not catch value-type bugs.

If the bug doesn't exist on `main`, skip this step, say so in the hotfix PR, and in step 3 branch from the maintenance branch without cherry-picking.

### 2. Create the maintenance branch (once per major)

Only if `release/v<major>` doesn't exist yet. Example for major 3:

```bash
MAJOR=3
git fetch --tags origin
TAG=$(git tag -l "v${MAJOR}.*" | grep -E "^v${MAJOR}\.[0-9]+\.[0-9]+$" | sort -V | tail -n 1)
git switch -c "release/v${MAJOR}" "$TAG"
```

Create `release-please-config.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "go",
  "versioning": "always-bump-patch",
  "include-component-in-tag": false,
  "packages": { ".": {} }
}
```

Create `.release-please-manifest.json` with the version of `$TAG` without the `v`. For example:

```json
{ ".": "3.3.1" }
```

In `.github/workflows/release.yml`, change `branches: [main]` to `branches: [main, release/v3]`.

```bash
git add release-please-config.json .release-please-manifest.json .github/workflows/release.yml
git commit -m "chore: initialise release/v${MAJOR} maintenance branch from ${TAG}"
git push origin "release/v${MAJOR}"
```

These changes exist only on the maintenance branch, never on `main`. The commit is `chore:`, so it doesn't trigger a release.

If the branch already exists but has no config or manifest (func's `release/v3` today), add the same changes through a PR to that branch.

### 3. Backport the fix

```bash
git fetch origin
git switch -c "backport/<short-name>-v${MAJOR}" "origin/release/v${MAJOR}"
git cherry-pick -x <merge-commit-sha-on-main>
git push -u origin HEAD
gh pr create --base "release/v${MAJOR}" --title "fix: <same title as the main PR>" --body "Backport of #<main PR number>."
```

- Resolve conflicts if the code has diverged, and keep the regression test.
- The PR title must be a conventional `fix:` title, because it ends up in the CHANGELOG.

### 4. Release

After the merge, release-please opens `chore(release/v<major>): release <version>` on the maintenance branch. Review and merge it.

release-please then creates:
- the tag, on the maintenance branch commit
- the GitHub release
- the CHANGELOG entry on that branch

The shared workflow sets "Latest" back to the newest release of `main`. The Terraform Registry picks up the tag automatically.

### 5. Verify

```bash
gh release list
```

- "Latest" still points to the newest release of `main`.
- The Terraform Registry shows the new patch version.

## Rules

- Never create tags, GitHub releases or CHANGELOG entries by hand.
- Don't merge a hotfix release before the fix is merged on `main`, unless the bug doesn't exist on `main`.
- No feature work on maintenance branches.
- Don't copy CHANGELOG entries to `main`. Each branch keeps its own version line, and GitHub Releases shows the combined history.

## Version numbers on main

- Test and tooling dependency updates use `chore(deps):`, so they don't bump the module's minor version.
- If a release must get a specific version, add a `Release-As: x.y.z` footer to the commit.
- A genuine pending `feat` on `main` makes the next release a minor. That's correct semver.

## Requirements per module repo

- The shared release workflow in terraform-azure-workflows supports `release/*` branches.
- A ruleset covers `~DEFAULT_BRANCH` and `refs/heads/release/*`.

Nothing else is needed per module.
