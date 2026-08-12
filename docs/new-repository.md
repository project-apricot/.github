# Setting up a new library repository

Substitute `REPO` and `PKG`. Two ordering constraints, both of which fail confusingly:

- **`shared-workflows` must be tagged `v1` before any caller runs** — workflows pin `@v1`.
- **Add the `main` ruleset *after* the first PR merges.** Requiring the `ci` check before it
  has ever reported can block your own first PR.

## 1. Create

```bash
gh repo create project-apricot/REPO --public \
  --description "..." --license apache-2.0
```

## 2. Settings

```bash
gh api -X PATCH repos/project-apricot/REPO \
  -F allow_merge_commit=false -F allow_rebase_merge=false -F allow_squash_merge=true \
  -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=BLANK \
  -F delete_branch_on_merge=true -F has_wiki=false \
  -f homepage=https://projectapricot.dev
```

`PR_TITLE` matters: the default `COMMIT_OR_PR_TITLE` uses the *commit* message for
single-commit PRs, so the title check could pass while a different message lands on `main` and
the version bump breaks.

UI equivalent: **Settings → General → Pull Requests**.

## 3. Seed tag

```bash
git tag v0.0.0 <first-commit-sha>
git push origin v0.0.0
```

Version inference is relative to the previous tag, so a base tag must exist.

## 4. Scaffold

Copy from an existing library repo, so the scaffold is always one that is known to build.

```bash
EXISTING=../<existing-library>
cp -R $EXISTING/.github .
cp $EXISTING/{Directory.Build.props,Directory.Packages.props,global.json} .
cp $EXISTING/{nuget.config,.editorconfig,.gitignore} .
cp $EXISTING/src/Directory.Build.props src/
cp $EXISTING/tests/.editorconfig tests/
```

Then edit:

- `Directory.Build.props` → `RepositoryName`
- `assets/icon.*` → and `PackageIcon` must match its flattened name
- each `src/*/*.csproj` → `Description`, `PackageTags`

See [dotnet-library-conventions.md](dotnet-library-conventions.md).

## 5. First PR

```bash
git checkout -b feat/initial
git add -A && git commit -m "feat: initial library"    # feat: is what produces 0.1.0
git push -u origin feat/initial
gh pr create --fill && gh pr checks --watch
gh pr merge --squash --delete-branch
```

## 6. Ruleset

UI only — **Settings → Rules → Rulesets → New branch ruleset**, target `main`: require a PR,
require the `ci` status check, block force pushes, require linear history. No bypass actor is
needed; nothing in CI pushes to `main`.

## 7. Secret

```bash
gh secret set NUGET_USER --repo project-apricot/REPO --body '<nuget.org profile name>'
```

## 8. nuget.org

**Trusted Publishing policy** — nuget.org → your username → Trusted Publishing → add:

| Field | Value |
| --- | --- |
| Package Owner | `project-apricot` — required, see below |
| Repository Owner | `project-apricot` |
| Repository | `REPO` |
| Workflow File | `release.yml` — filename only, no path |
| Environment | empty |

A new policy may be only **temporarily active for 7 days** until the first successful publish
supplies GitHub's repo and owner IDs.

**Package Owner must be `project-apricot`.** The `ApricotFramework.*` prefix is reserved to
that owner and is not public, so nuget.org **rejects** a package under the prefix submitted by
any other owner. Publishing as an individual account fails even with a valid policy.

## 9. Release

```bash
gh workflow run release.yml -f dry_run=true    # confirm the computed version
gh workflow run release.yml                    # prepare + publish in one run
gh run watch
```

## 10. Verify

```bash
curl -s https://api.nuget.org/v3-flatcontainer/<lowercase-pkg-id>/index.json
```

Unzip the `.nupkg` from the run artifact: `lib/<tfm>/` present, `.xml` docs beside the dll,
readme/licence/icon embedded, sibling `.snupkg`, and **no test or example package**.
