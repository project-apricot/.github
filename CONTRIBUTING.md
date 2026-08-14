# Contributing

Applies to every Project Apricot repository. A repo adds its own `CONTRIBUTING.md` only if
it has something specific to say.

Bug reports with a reproduction are the most useful contribution; a failing test is better
still. For anything that adds public API, **open an issue first** — the bar for new surface
area is high and it is cheaper to agree on the shape before the code exists.

## Pull requests

We squash-merge, so **the PR title becomes the commit message** — which drives the version
bump and becomes a line in the release notes. CI enforces
[Conventional Commits](https://www.conventionalcommits.org/):

| Prefix | Next release |
| --- | --- |
| `fix:` | patch |
| `feat:` | minor |
| `feat!:` | major |
| `docs:` `test:` `chore:` `ci:` `refactor:` | none |

No trailing period, and write the subject for someone reading the changelog: `fix: return empty
string for whitespace input`, not `fix: Fixed the bug.`

Also: warnings are errors in every repo, so the build must be clean. If your PR changes
behaviour, update that repo's `docs/` **in the same PR** — that is why docs live next to the
code.

## Flag in the description

- a change to any value a consumer might have persisted (hash, encoding, generated id)
- removing or renaming public API, or dropping a supported runtime version

## Questions

Open an issue. For anything security-related do **not** open an issue — see
[SECURITY.md](SECURITY.md).

By participating you agree to the [Code of Conduct](CODE_OF_CONDUCT.md).
