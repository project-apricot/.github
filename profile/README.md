# Project Apricot

A framework for building production-ready, secure .NET applications quickly — a growing
collection of small, focused, well-tested libraries, plus the CI/CD that ships them.

<https://projectapricot.dev>

## Principles

- **Zero or near-zero dependencies.** Taking one of our packages should not drag a
  dependency tree into your application.
- **Stable output is a contract.** Anything a consumer might persist — a hash, an
  encoding, a generated identifier — is pinned by tests and changes only in a major version.
- **Secure by default.** No long-lived publishing credentials, and every released version
  is traceable to a tag, a GitHub Release and the CI run that built it.
- **Small surface.** New public API needs a reason, not just a use case.

## Repositories

| Repo | What it is |
| --- | --- |
| [core-utility](https://github.com/project-apricot/core-utility) | Zero-dependency .NET utilities: URL-safe hashing, random strings, URL helpers |
| [shared-workflows](https://github.com/project-apricot/shared-workflows) | Reusable GitHub Actions workflows used across the org |

More libraries are on the way, and more ecosystems after that.

Everything is Apache-2.0. Contributions welcome —
see [CONTRIBUTING](https://github.com/project-apricot/.github/blob/main/CONTRIBUTING.md).
