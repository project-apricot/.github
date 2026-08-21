# .NET library conventions

## Layout

```
Directory.Build.props        settings for every project in the repo
Directory.Packages.props     central package versions
src/Directory.Build.props    packaging - applies only to shipping libraries
src/<PackageId>/             one directory per package
tests/<PackageId>.Tests/
tests/.editorconfig          test-only analyzer relaxations
examples/                    runnable apps demonstrating the library (optional)
docs/                        MDX, published by the docs site
assets/icon.*
global.json  nuget.config  .editorconfig
<Solution>.slnx
```

The `src/` vs `tests/` split is what lets packaging settings apply to shipping code only,
with no per-project opt-outs to forget.

`examples/` inherits `IsPackable=false`, so nothing there ships. Keep examples in the
solution — CI then builds them, which catches API breakage the tests might miss.

## Where each setting lives

| File | Holds |
| --- | --- |
| `Directory.Build.props` | TFM, `LangVersion`, `Nullable`, `ImplicitUsings`, analyzers, `TreatWarningsAsErrors`, audit, identity, package metadata, `IsPackable=false` |
| `src/Directory.Build.props` | `IsPackable=true` + `GenerateDocumentationFile=true` |
| `<Package>.csproj` | `Description`, `PackageTags` — only what differs per package |
| `tests/.editorconfig` | `CA1707` off, so `Method_Scenario_Expected` test names build |

**Publishing is opt-in.** The root default is `IsPackable=false`; only `src/` turns it on.
Forgetting to opt in means a package is not published — CI catches the total failure via
`if-no-files-found: error` on the pack artifact. Forgetting to opt *out* would publish a test
or example package permanently.

**`GenerateDocumentationFile` must be set beside `IsPackable`, in a `.props` file.** It cannot
be derived (`$(IsPackable)`): MSBuild evaluates in import order, so `IsPackable` is not final
until after the project body, and by `Directory.Build.targets` the SDK has already computed
`DocumentationFile`. Attempting it gives a **green build with a doc-less package**.

## Target frameworks

Set `TargetFramework` (or `TargetFrameworks`) in the root props — one place.

- Adding a target is a **minor** bump; removing one is always a **major**.
- **Keep `dotnet_versions` in `ci.yml` in sync**, or CI cannot run the tests for a target it
  has no runtime for.
- When multi-targeting, note a newer SDK can *build* an older TFM but not *run* its tests, and
  building an older-TFM **executable** needs that version's apphost pack — which hits test
  projects, since an xunit v3 test project *is* an executable.

## Packaging metadata

Root props carries everything shared: authors, copyright, repository URL (composed from
`RepositoryOwner` + `RepositoryName`), license expression, readme, icon, release notes,
`EnablePackageValidation`, SourceLink, and `snupkg` symbols.

Three things that break quietly:

- **`PackageIcon` must match the packed file's flattened name** — `icon.jpg`, not
  `assets/icon.jpg`, because `PackagePath=""` puts it at the package root.
- **Never set `PackageLicenseFile` alongside `PackageLicenseExpression`** — mutually exclusive,
  NuGet errors.
- **SourceLink needs no PackageReference** since the .NET 8 SDK; `PublishRepositoryUrl` is
  enough.

## global.json

Pin the **minimum** SDK you support, with `rollForward: latestFeature`. It rolls forward only,
so pinning your current patch rejects contributors on a lower feature band. Its real value is
being a ceiling: without it a new major SDK is picked up silently, and with
`TreatWarningsAsErrors` a new analyzer warning then breaks the build with no code change.

## Tests

xunit v3 run natively on Microsoft.Testing.Platform; versions centralised in
`Directory.Packages.props`. Do not set `OutputType` — `xunit.v3` sets it.

**MTP is opted into per repo, in `global.json`:**

```json
{ "sdk": { … }, "test": { "runner": "Microsoft.Testing.Platform" } }
```

This is the only supported mechanism on the .NET 10 SDK. `TestingPlatformDotnetTestSupport` is the
.NET 9-and-earlier property and is being removed; the `dotnet.config` runner section never shipped.
The opt-in applies to the whole tree below it and is **all or nothing** — every test project
`dotnet test` discovers must be MTP, or the run errors.

The test stack is therefore `xunit.v3` + `coverlet.mtp` +
`Microsoft.Testing.Extensions.TrxReport`. `Microsoft.NET.Test.Sdk` and
`xunit.runner.visualstudio` stay only so an IDE can still run tests through VSTest; they are inert
once `global.json` opts in. **`coverlet.collector` does not work under MTP** — it is a VSTest data
collector. `coverlet.mtp` is the same project's MTP extension and is MIT, where Microsoft's
`Microsoft.Testing.Extensions.CodeCoverage` ships under a proprietary licence file.

MTP takes different reporting flags from VSTest: `--collect:"XPlat Code Coverage"` becomes
`--coverlet --coverlet-output-format cobertura`, and `--logger trx` becomes `--report-trx`.
`--results-directory`, `-c` and `--no-build` are unchanged.

Two behaviours that differ from VSTest and will change what CI does:

- **A run that discovers zero tests exits 8**, where VSTest exited 0. A filter that matches nothing
  now fails the build.
- **Do not pass `--report-trx-filename` for a multi-project run.** Each test assembly writes its own
  report, so a fixed name makes them overwrite each other and the last one wins — a green build
  publishing a fraction of its results. The default `{asm}_{tfm}_{arch}.trx` keeps them separate.

**Anything a consumer might persist is pinned by golden-value tests** — a hash, an encoding, a
generated identifier. If such a test fails that is the test working; do not update the
expected value unless a major bump is the intent.

## Versioning

Versions come from conventional commits, computed in CI. No version is ever committed.

| PR title prefix | Effect |
| --- | --- |
| `fix:` | patch |
| `feat:` | minor |
| `feat!:` or `BREAKING CHANGE:` footer | major |
| `docs:` `test:` `chore:` `ci:` `refactor:` `perf:` `build:` | no release |

We squash-merge, so **the PR title is the unit of release**, not the commit. Four commits
collapse to one and only the title is read — a `feat!:` in a commit body does *not* register.
Across PRs the highest bump wins; it is not additive.

Workflow-only changes are `ci:` — no bump, so you do not ship a version identical to the last.

Needs a major: changing a persistable value, removing or renaming public API, dropping a
supported runtime.

## Multi-package repositories

Supported with no extra machinery:

- One version per repo, from that repo's tags. Packages release together.
- `src/Directory.Build.props` already makes every library under `src/` packable, so a new
  package needs only its directory and a `Description`.
- `dotnet pack <solution>` packs all packable projects and the publish action pushes
  `./artifacts/*.nupkg` — no workflow change.
- `PackageValidationBaselineVersion` is per package, set once each has a published version.

Naming follows the ecosystem, not the source repo: `<Prefix>.<Library>.AspNetCore`, not
`.Aspnet`. A package id can be changed for free until the first publish and never after — a
rename post-publish is a *new* id forever, since nuget.org versions can be unlisted but not
deleted.

## Porting an existing library

This props set is stricter than any code written before it, so a port that built fine in its
old home will not build here. `TreatWarningsAsErrors=true` with
`AnalysisLevel=latest-recommended` turns established idioms into hard failures, not style
nits. Budget for it rather than being surprised by it.

Seen so far, and both recur:

- **CA1051** — `protected readonly` fields on a public type. Convert to a protected property,
  or make the field private and expose only what callers need.
- **CA1707** — underscores in externally visible names, which includes `public`/`protected`
  `SCREAMING_CASE` consts. The `tests/.editorconfig` waiver is scoped to `tests/` and does
  nothing for `src/`.

Both rules are **visibility-scoped**, which is the practical lever: a private member trips
neither. `core-utility`'s `private static readonly char[] ALL_CHARS` survives untouched for
exactly that reason, while the same name on a public const would not.

**Fix these before the first publish.** Reshaping public API is free while nothing depends on
it and a major bump afterwards, so the port is the only cheap window — the same logic that
makes the package id renameable only now.

**Do not relax the root analyzer set to make a port compile.** The root props stays strict so
new code is held to it. Where a finding genuinely must stand, the escape hatch is narrow: a
`#pragma` at the site, or a `src/<Package>/.editorconfig` entry, with a comment naming the
reason — the same shape as `tests/.editorconfig`.

Two more that are policy, not analyzers:

- **No per-file license headers.** The `LICENSE` file is the only notice; strip any headers the
  source carried.
- **`grep -rin <old-org-or-namespace>` over the whole tree must return nothing** before the PR.
  It catches namespaces, package ids, feed URLs and doc links in one pass. Run it last.
