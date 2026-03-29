# Standard: NuGet Library Publishing

> **Audience:** AI agents writing, reviewing, or packaging .NET libraries for NuGet.
> **Prerequisite:** Read `skills/csharp-dev/standards/idiomatic-csharp.md` for general C# rules.

## Core Principle

A NuGet library is a **public API contract**. Every public type is a promise. Design for consumers, version deliberately, and automate the publish pipeline so humans only decide *what* to release.

---

## 1. Package Metadata in Directory.Build.props

**Rule:** Centralise package metadata in `Directory.Build.props` at the solution root. Individual `.csproj` files set only `<IsPackable>true</IsPackable>` and project-specific overrides.

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <Authors>John Azariah</Authors>
    <Company>johnazariah</Company>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <PackageProjectUrl>https://github.com/johnazariah/$(PackageId)</PackageProjectUrl>
    <RepositoryUrl>https://github.com/johnazariah/$(PackageId)</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PackageIcon>icon.png</PackageIcon>
    <Copyright>Copyright © $(Authors) $([System.DateTime]::UtcNow.Year)</Copyright>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

### Guidelines

- **`PackageId`** defaults to the assembly name. Only override if the NuGet name differs from the project name.
- **`PackageReadmeFile`** — include a README.md in the package. Add `<None Include="README.md" Pack="true" PackagePath="" />` to the csproj.
- **Source Link** — add `<PublishRepositoryUrl>true</PublishRepositoryUrl>` and the `Microsoft.SourceLink.GitHub` package for debugger source stepping.

---

## 2. Versioning via Git Tags

**Rule:** Use the git tag as the single source of truth for package version. `MinVer` reads the nearest semver tag and sets `<Version>` automatically.

```xml
<!-- Directory.Build.props -->
<ItemGroup>
  <PackageReference Include="MinVer" Version="6.*" PrivateAssets="all" />
</ItemGroup>

<PropertyGroup>
  <MinVerTagPrefix>v</MinVerTagPrefix>
  <MinVerDefaultPreReleaseIdentifiers>preview.0</MinVerDefaultPreReleaseIdentifiers>
</PropertyGroup>
```

### Flow

1. Tag `v1.0.0` → builds produce `1.0.0`
2. Commit after tag → builds produce `1.0.1-preview.0.1` (auto pre-release)
3. Tag `v1.1.0` → builds produce `1.1.0`

### Guidelines

- **Never set `<Version>` manually** in csproj — MinVer owns it.
- **Tag prefix `v`** — matches the release process convention (`v1.2.3`).
- **Pre-release on untagged commits** — CI packages are always pre-release, preventing accidental stable publishes.

---

## 3. Public API Surface

**Rule:** Every public type and member must have XML documentation. Use `<GenerateDocumentationFile>true</GenerateDocumentationFile>` and treat missing docs as warnings.

### Guidelines

- **`[EditorBrowsable(EditorBrowsableState.Never)]`** on types that must be public for technical reasons but aren't part of the logical API.
- **`internal`** by default — only make types public when consumers need them.
- **No public mutable state** — all public properties are `init`-only or `required`.
- **Seal everything** — `sealed record`, `sealed class`. Inheritance is an API contract that's hard to maintain.
- **F# libraries** — use `[<AutoOpen>]` sparingly. Prefer `[<RequireQualifiedAccess>]` modules to avoid polluting consumer namespaces.

---

## 4. Pack and Publish

**Rule:** `dotnet pack` produces the `.nupkg`. CI publishes on tag. Local builds never publish.

```bash
# Local — just verify it packs
dotnet pack -c Release --no-build

# CI — tag-triggered publish
dotnet pack -c Release
dotnet nuget push **/*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
```

### Guidelines

- **`--skip-duplicate`** — idempotent pushes. Safe to re-run CI without version conflicts.
- **Symbol packages** — use `<IncludeSymbols>true</IncludeSymbols>` with `<SymbolPackageFormat>snupkg</SymbolPackageFormat>` for NuGet.org symbol server.
- **Deterministic builds** — add `<Deterministic>true</Deterministic>` and `<ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>` in CI.

---

## 5. CI Workflow — publish-nuget.yml

The `publish-nuget.yml` workflow fires on semver tags:

```yaml
name: Publish to NuGet
on:
  push:
    tags: ['v[0-9]+.[0-9]+.[0-9]+']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # MinVer needs full history
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.x'
      - run: dotnet restore
      - run: dotnet build -c Release --no-restore
      - run: dotnet test -c Release --no-build
      - run: dotnet pack -c Release --no-build -o ./artifacts
      - run: dotnet nuget push ./artifacts/*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
```

---

## Self-Check Table

| # | Smell | Fix |
|---|-------|-----|
| 1 | `<Version>` hardcoded in csproj | Use MinVer — version from git tags |
| 2 | Missing XML docs on public API | `<GenerateDocumentationFile>` + write docs |
| 3 | Public mutable properties in library types | `init`-only or `required` properties |
| 4 | Unsealed public types | `sealed` unless inheritance is the design intent |
| 5 | Package metadata scattered across csproj files | Centralise in `Directory.Build.props` |
| 6 | Manual version bumps before publish | Tag-triggered CI with MinVer |
| 7 | Missing Source Link | Add `Microsoft.SourceLink.GitHub` |
