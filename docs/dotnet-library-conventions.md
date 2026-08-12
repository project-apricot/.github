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
  projects, since xunit v3 builds as an exe.

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

xunit v3 on Microsoft.Testing.Platform; versions centralised in `Directory.Packages.props`.
Do not set `OutputType` — `xunit.v3` sets it. `dotnet test` runs every TFM in sequence.

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
