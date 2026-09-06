# Package Manager Reference

How to find, edit, and update dependency versions for each ecosystem Dependabot supports.

## npm

| Field | Value |
|-------|-------|
| **Manifest** | `package.json` |
| **Lockfile** | `package-lock.json` |
| **Version format** | semver (`1.2.3`) |
| **Direct update** | `npm install {pkg}@{ver}` |
| **Transitive update** | `npm update {pkg}` |
| **Lock-only regen** | `npm install` |

**Finding the version in the manifest:**
Look in `dependencies`, `devDependencies`, `peerDependencies`, or `optionalDependencies` objects:
```json
"dependencies": {
  "lodash": "^4.17.19"
}
```
Replace the version value (including range prefix like `^` or `~`) with `^{patched_version}`. Preserve the existing range prefix if present; if no prefix, use `^`.

---

## pip

| Field | Value |
|-------|-------|
| **Manifest** | `requirements.txt`, `setup.py`, `setup.cfg`, or `pyproject.toml` |
| **Lockfile** | None (or `requirements.txt` if used as a lockfile) |
| **Version format** | PEP 440 (`1.2.3`) |
| **Direct update** | `pip install {pkg}=={ver}` |
| **Transitive update** | `pip install --upgrade {pkg}` |
| **Lock-only regen** | N/A |

**Finding the version in the manifest:**

`requirements.txt`:
```
package==1.2.3
package>=1.2.0,<2.0.0
```
Replace the version specifier with `>={patched_version}`.

`pyproject.toml` (under `[project]` or `[tool.poetry.dependencies]`):
```toml
dependencies = ["package>=1.2.0"]
```

---

## maven

| Field | Value |
|-------|-------|
| **Manifest** | `pom.xml` |
| **Lockfile** | None |
| **Version format** | Maven versioning (`1.2.3`, `1.2.3-RELEASE`) |
| **Direct update** | Edit `<version>` tag in `pom.xml` |
| **Transitive update** | Add/update `<dependencyManagement>` entry to force version |
| **Lock-only regen** | N/A |

**Finding the version in the manifest:**
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>{package_name}</artifactId>
  <version>1.2.3</version>
</dependency>
```
Replace the content of the `<version>` tag. The package name in the alert maps to `artifactId` (or `groupId:artifactId`).

Note: versions may be defined as properties (`${some.version}`). If so, find and update the property in the `<properties>` block.

---

## gomod

| Field | Value |
|-------|-------|
| **Manifest** | `go.mod` |
| **Lockfile** | `go.sum` |
| **Version format** | semver with `v` prefix (`v1.2.3`) |
| **Direct update** | `go get {pkg}@v{ver}` |
| **Transitive update** | `go get {pkg}@v{ver}` |
| **Lock-only regen** | `go mod tidy` |

**Finding the version in the manifest:**
```
require (
    github.com/example/pkg v1.2.3
)
```
Replace the version. Always include the `v` prefix.

---

## cargo

| Field | Value |
|-------|-------|
| **Manifest** | `Cargo.toml` |
| **Lockfile** | `Cargo.lock` |
| **Version format** | semver (`1.2.3`) |
| **Direct update** | Edit `Cargo.toml`, then `cargo update -p {pkg}` |
| **Transitive update** | `cargo update -p {pkg}` |
| **Lock-only regen** | `cargo generate-lockfile` |

**Finding the version in the manifest:**
```toml
[dependencies]
serde = "1.0.130"
serde = { version = "1.0.130", features = ["derive"] }
```
Replace the version string.

---

## composer

| Field | Value |
|-------|-------|
| **Manifest** | `composer.json` |
| **Lockfile** | `composer.lock` |
| **Version format** | semver (`1.2.3`) |
| **Direct update** | `composer require {pkg}:{ver}` |
| **Transitive update** | `composer update {pkg}` |
| **Lock-only regen** | `composer install` |

**Finding the version in the manifest:**
```json
"require": {
    "vendor/package": "^1.2.0"
}
```
Replace the version constraint with `^{patched_version}`.

---

## nuget

| Field | Value |
|-------|-------|
| **Manifest** | `*.csproj`, `*.vbproj`, `*.fsproj`, or `packages.config` |
| **Lockfile** | None (NuGet restores from manifest) |
| **Version format** | semver (`1.2.3`) |
| **Direct update** | `dotnet add package {pkg} -v {ver}` |
| **Transitive update** | Add explicit `<PackageReference>` to force version |
| **Lock-only regen** | `dotnet restore` |

**Finding the version in the manifest:**

`.csproj`:
```xml
<PackageReference Include="Newtonsoft.Json" Version="12.0.3" />
```
Replace the `Version` attribute.

`packages.config`:
```xml
<package id="Newtonsoft.Json" version="12.0.3" targetFramework="net48" />
```
Replace the `version` attribute.

---

## rubygems

| Field | Value |
|-------|-------|
| **Manifest** | `Gemfile` |
| **Lockfile** | `Gemfile.lock` |
| **Version format** | RubyGems (`1.2.3`) |
| **Direct update** | `bundle update {pkg}` |
| **Transitive update** | `bundle update {pkg}` |
| **Lock-only regen** | `bundle install` |

**Finding the version in the manifest:**
```ruby
gem 'rails', '~> 6.1.0'
gem 'nokogiri', '>= 1.12.5'
```
Replace the version constraint with `'>= {patched_version}'`.

---

## github-actions

| Field | Value |
|-------|-------|
| **Manifest** | `.github/workflows/*.yml` |
| **Lockfile** | None |
| **Version format** | `@v{major}`, `@v{major}.{minor}.{patch}`, or `@{sha}` |
| **Direct update** | Edit the `uses:` line |
| **Transitive update** | N/A |
| **Lock-only regen** | N/A |

**Finding the version in the manifest:**
```yaml
steps:
  - uses: actions/checkout@v3
  - uses: actions/setup-node@v3.6.0
  - uses: some/action@abc123def456
```
Replace the ref after `@`. If the current ref is a major tag (`@v3`), update to the new major tag. If it's a full version (`@v3.6.0`), update to the patched version. If it's a SHA, look up the SHA for the patched version tag.

The `package_name` from the alert corresponds to the action name (e.g., `actions/checkout`).
