# .github

Org-wide defaults and handbook for `project-apricot`. The health files here are inherited by
every repository in the org that does not define its own copy.

## Handbook

| Doc | Read it when |
| --- | --- |
| [new-repository.md](docs/new-repository.md) | Standing up a new library repo — ordered runbook, `gh` commands and UI steps |
| [dotnet-library-conventions.md](docs/dotnet-library-conventions.md) | Deciding where a build setting goes, or what a new package must declare |
| [ci-cd.md](docs/ci-cd.md) | Touching workflows, releases, or trusted publishing |

## Inherited defaults

| Path | Purpose |
| --- | --- |
| `profile/README.md` | The org landing page at github.com/project-apricot |
| `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md` | Defaults for every repo |
| `.github/ISSUE_TEMPLATE/`, `.github/PULL_REQUEST_TEMPLATE.md` | Default issue and PR forms |

Reusable workflows and composite actions live in
[shared-workflows](https://github.com/project-apricot/shared-workflows), not here — they need
semver tags and review that community health files do not.
