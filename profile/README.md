# Project Apricot

A framework for building production-ready, secure .NET applications quickly — a growing
collection of small, focused, well-tested libraries, plus the CI/CD that ships them.

<https://projectapricot.dev>

## What it covers

The libraries fill the gaps between "the framework gives you a primitive" and "your application
needs a decision made": identity and secret handling, request and response plumbing, data access,
locale and template rendering, identifier generation, and letting a service describe itself and
find its neighbours.

Each is independent. Take one, take several, or take one and replace the piece of it you disagree
with — nothing assumes you adopted the rest.

## How a library is put together

Most repositories publish two or three packages that layer:

| Package | Contains |
| --- | --- |
| `ApricotFramework.<Feature>` | the feature itself, dependency-free unless the job genuinely needs a library — usable from a console app, a worker, or a test |
| `ApricotFramework.<Feature>.AspNetCore` | registration, configuration binding and startup validation, usually taking everything from the shared framework |
| `ApricotFramework.<Feature>.<Integration>` | an optional adapter — problem-details mapping, a storage backend, an alternative source format |

The split is the point. The core never knows how it was configured, the integration never
reimplements the feature, and an adapter you do not reference costs you nothing. Where two
libraries need to agree — on how an error becomes an HTTP response, say — they agree through a
published contract rather than by depending on each other.

## Principles

- **Zero or near-zero dependencies.** Taking one of our packages should not drag a
  dependency tree into your application.
- **Stable output is a contract.** Anything a consumer might persist — a hash, an
  encoding, a generated identifier, a stored format — is pinned by tests and changes only in a
  major version.
- **Secure by default.** No long-lived publishing credentials, and every released version
  is traceable to a tag, a GitHub Release and the CI run that built it.
- **Small surface.** New public API needs a reason, not just a use case.
- **Fail at startup, not in production.** Most libraries validate their configuration as the host
  starts, so a missing or nonsensical setting is a boot failure rather than a first-request surprise.

## How it ships

Every library lives in its own repository with its own docs, and publishes to nuget.org under the
`ApricotFramework.*` prefix. Versions are inferred from conventional-commit pull request titles —
no version is ever committed — and pushed by trusted publishing, so there is no API key to leak or
rotate. The reusable workflows behind that live in
[shared-workflows](https://github.com/project-apricot/shared-workflows); the conventions every
repository follows are written down in
[the handbook](https://github.com/project-apricot/.github/tree/main/docs).

Browse the repositories below for the full set. More are on the way, and more ecosystems after
that.

Everything is Apache-2.0. Contributions welcome —
see [CONTRIBUTING](https://github.com/project-apricot/.github/blob/main/CONTRIBUTING.md).
