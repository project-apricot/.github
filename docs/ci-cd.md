# CI/CD model

Shared logic lives in
[`shared-workflows`](https://github.com/project-apricot/shared-workflows), pinned `@v1`.
A library repo holds four thin workflows.

| Workflow | Trigger | Does |
| --- | --- | --- |
| `ci.yml` | PR, push to `main` | restore, build, test, pack. Publishes nothing |
| `pr-title.yml` | PR | enforces conventional-commit titles |
| `preview.yml` | dispatch | prerelease → GitHub Packages |
| `release.yml` | dispatch, `release: published` | version + Release + publish to nuget.org |

**The trigger is the signal** — there is no flag declaring "this is a real release".

## Releasing

One dispatch of `release.yml` does everything:

```
prepare  →  infers the version from conventional commits, creates the Release + tag
publish  →  checks out the TAG, builds, tests, packs, pushes to nuget.org
```

| Inputs | prepare | publish |
| --- | --- | --- |
| *(none)* | runs | runs |
| `dry_run: true` | runs | skipped — version preview only, no build |
| `tag: v0.1.0` | skipped | publishes that existing tag |
| *(a person created a Release)* | skipped | publishes its tag |

The `tag` path is the retry: `--skip-duplicate` makes re-publishing a no-op, so a half-failed
release is re-run, never re-cut.

`prepare` guards before creating anything: on `main`, valid semver, releasable changes exist, a
`prerelease` flag has a suffixed version, and the tag is free.

## Two constraints that shape all of this

### GitHub does not start workflows from events raised with `GITHUB_TOKEN`

A Release created by a workflow **cannot** trigger `release: published` — it is the recursion
guard. Symptom: the Release exists, the publish workflow has zero runs, nothing errors.

So publishing runs as a job in the same graph. `release: published` is kept only for Releases
created **by a person**. A PAT would also make the event fire, at the cost of a long-lived
credential.

### nuget.org validates the OIDC claim `job_workflow_ref`

It must start with `<owner>/<repo>/.github/workflows/`, so **the publishing job must be defined
in the library repo itself**. A reusable workflow sets that claim to its own path and the
exchange fails:

```
Token exchange failed (HTTP 401)
Claim 'job_workflow_ref' has value '<other-repo>/.github/workflows/<workflow>.yml@refs/tags/v1'
which does not start with <owner>/<repo>/.github/workflows/
```

No policy configuration fixes it. Publishing is therefore a **composite action**
(`shared-workflows/actions/dotnet-publish`): composite steps run inside the caller's job, so the
claim stays the caller's workflow. The calling job grants `id-token: write` and checks out the
tag itself.

Rule of thumb: **anything consuming OIDC must be a composite action; everything else can be a
reusable workflow.**

## Trusted publishing

`NuGet/login` exchanges the Actions OIDC token for an API key valid for **1 hour**, so the login
step sits immediately before the push. Policy fields and the activation window are in
[new-repository.md](new-repository.md#8-nugetorg).

`NUGET_USER` is the nuget.org **profile name** of the policy *creator*, not the package owner.
With an org-owned policy it is still your individual profile name.

## Feeds

| Feed | Carries | Auth |
| --- | --- | --- |
| nuget.org | stable `x.y.z` only | OIDC, no stored secret |
| GitHub Packages | previews `x.y.z-preview.<date>.<sha>` | built-in `GITHUB_TOKEN` |

nuget.org versions can be unlisted but **never deleted** — hence prereleases are withheld
unless `allow_prerelease: true`, and hence the human dispatch gate. GitHub Packages versions
*are* deletable, so previews belong there.

Its NuGet registry requires authentication **even for public packages**, so a public repo must
never depend on a package hosted only there.

## shared-workflows versioning

Callers pin `@v1`, a moving tag. A release cuts `vX.Y.Z` *and* moves the major tag — both halves
matter, since a version nobody is pinned to changes nothing. `self-release.yml` does both.
Breaking input changes ship as `v2`.

Consequence: `v*` tags cannot be protected, because force-moving `v1` is inherent to the pattern.

## Gotchas

- **Seed `v0.0.0`** in a new repo; version inference is relative to the previous tag.
- **`ci:` for workflow-only changes** — no bump, so you do not ship a version identical to the
  last one.
- **`.DS_Store` is not in the VisualStudio `.gitignore` template.**
- **`dry_run` only previews the version** — it does not build or test.
