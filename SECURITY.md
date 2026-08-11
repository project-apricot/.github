# Security Policy

**Please do not report vulnerabilities in a public issue.**

Use GitHub's private vulnerability reporting on the affected repository — **Security** tab →
**Report a vulnerability** — or email **info [at] projectapricot.dev**.

Include the affected repository and version, what the issue is, and a reproduction if you
have one. You will get an acknowledgement within 3 business days and an assessment within
10. We will credit you in the advisory unless you would rather stay anonymous.

## Supported versions

Only the latest released minor version of each package receives security fixes. There are no
maintenance branches — these libraries are small enough that upgrading is cheap.

## What we do on our side

- Zero or near-zero runtime dependencies, so our packages add almost no transitive
  supply-chain surface.
- Dependency auditing on every build, with Dependabot opening the bumps.
- No long-lived publishing credentials: releases authenticate with short-lived tokens
  exchanged via GitHub OIDC.
- Nothing is published from a local machine — every release is cut by CI from a tagged
  commit, with deterministic, source-linked builds.
