# App versioning recipes (git-describe)

Per-ecosystem recipes for the optional git-describe versioning step (SKILL.md step 7). Read only the section that matches the stack — per-subtree if a repo mixes stacks (e.g. a .NET backend and a Vite frontend each get their own treatment).

**The contract every recipe implements.** The version is `git describe --tags --always --dirty` (e.g. `v0.1.0-32-g98af7ae`), resolved **at build time** and baked into the artifact, because a deployed container/binary/bundle has no `.git`. Wherever the app reports its version, resolve it in this order: (1) the baked-in value, (2) `git describe` as a dev-checkout fallback, (3) `"unknown"`. Pick the *idiomatic* baking mechanism below rather than bolting a runtime `git describe` onto a stack that already has one.

**The pitfall that bites every stack: shallow CI clones.** CI checkouts default to depth-1 with no tags, so `git describe --tags` silently degrades to a bare hash and builds lose their real version. Fix it once at checkout — GitHub Actions `actions/checkout` with `fetch-depth: 0`, GitLab `GIT_DEPTH: 0`, Azure `fetchDepth: 0`, or `git fetch --tags --unshallow` before the build. Use **annotated** tags (`git tag -a v0.1.0 -m ...`); `git describe` without `--tags` only sees annotated ones. Package/plugin versions below are representative of late-2025/2026 — check for a newer one when you wire it.

## Contents

- [Node.js / TypeScript](#nodejs--typescript) — backend, frontend bundlers, `package.json` reconciliation
- [.NET / C#](#net--c) — MinVer, plain MSBuild
- [Python](#python) — setuptools_scm / hatch-vcs, standalone apps
- [Go](#go)
- [Rust](#rust)
- [JVM (Java / Kotlin)](#jvm-java--kotlin) — Gradle, Maven
- [Containers / Docker](#containers--docker)

---

## Node.js / TypeScript

### Backend / Node service

**Idiom:** env-first, dev git fallback, resolved once at module load (never per request).

```ts
// version.ts
import { execSync } from "node:child_process";

function fromGit(): string | undefined {
  // Dev-checkout convenience only — a container has no .git/git, so this
  // throws and we fall through to "unknown". Never the prod source of truth.
  try {
    return execSync("git describe --tags --always --dirty", {
      stdio: ["ignore", "pipe", "ignore"],
    }).toString().trim();
  } catch {
    return undefined;
  }
}

export const APP_VERSION = process.env.APP_VERSION ?? fromGit() ?? "unknown";
```

In production, `APP_VERSION` is set at build/deploy time (Docker build-arg → `ENV`, or the build script below) and wins. Surface it on whatever metadata endpoint already exists (`/healthz` → `{ version: APP_VERSION }`).

### `package.json` version reconciliation

There are two versions: npm's `version` field (strict semver, read by the registry/tooling) and the precise `git describe` string (not valid semver — has the `v` prefix and `-g<sha>` suffix). Keep them separate:

- Leave `package.json` `"version"` as the **coarse release marker** (`0.1.0`), bumped to match the tag at release time. It stays valid semver.
- Inject the **precise** value via `APP_VERSION` / a bundler define at build — that's what the running app reports.
- Don't write the git-describe string back into `package.json` `version`; it breaks `npm publish`/`npm version`. (`process.env.npm_package_version` only exists when launched via an npm script — fine as a dev fallback, not a prod substitute.)

### Build-time injection (cross-platform)

Inline shell substitution (`APP_VERSION=$(git describe …) vite build`) works on macOS/Linux but **breaks on Windows `cmd`** (no `$(...)`), and `cross-env` doesn't do command substitution. The portable pattern is a tiny prebuild script that writes a committed-out artifact:

```js
// scripts/gen-version.mjs
import { execSync } from "node:child_process";
import { writeFileSync } from "node:fs";

const version =
  process.env.APP_VERSION ??
  (() => { try {
    return execSync("git describe --tags --always --dirty").toString().trim();
  } catch { return "unknown"; } })();

writeFileSync("src/version.ts", `export const APP_VERSION = ${JSON.stringify(version)};\n`);
```

```json
{ "scripts": { "prebuild": "node scripts/gen-version.mjs", "build": "vite build" } }
```

`prebuild` runs automatically before `build` (npm lifecycle), no shell-portability issue, and freezes the value into the bundle/image. In CI, set `APP_VERSION` once and the script passes it through. Add the generated file to `.gitignore`.

### Frontend bundlers

Browsers have **no `process.env` and no git** — the value must be statically substituted at build. Wrap it in `JSON.stringify` so it lands as a quoted literal.

```ts
// Vite — vite.config.ts
define: { __APP_VERSION__: JSON.stringify(process.env.APP_VERSION) }
// or set VITE_APP_VERSION in the build env and read import.meta.env.VITE_APP_VERSION

// esbuild
define: { __APP_VERSION__: JSON.stringify(process.env.APP_VERSION) }

// webpack / Rspack
new webpack.DefinePlugin({ __APP_VERSION__: JSON.stringify(process.env.APP_VERSION) })

// Next.js — next.config.js (inlines NEXT_PUBLIC_* at build)
module.exports = { env: { NEXT_PUBLIC_APP_VERSION: process.env.APP_VERSION } };
// read: process.env.NEXT_PUBLIC_APP_VERSION
```

Declare the `define` token for TypeScript (`declare const __APP_VERSION__: string;` in a `.d.ts`). Never read bare `process.env.APP_VERSION` in client code — it only works when the bundler inlines it; touching un-inlined `process.env` throws at runtime.

### Pitfalls

- Shallow CI clones (see top). `--always` masks a missing tag as a bare hash — assert the CI output contains your tag.
- Monorepo: one repo-wide tag describes the whole tree. For per-package versions use path-scoped tags + `git describe --tags --match 'pkgA-v*'`, or pass each package its own `APP_VERSION`.

---

## .NET / C#

**Don't shell out to `git describe` at runtime.** Published .NET apps don't ship `.git`, and the framework has first-class assembly versioning. Compute from git tags at build time, stamp the assembly, read it back by reflection.

### Idiom: MinVer

A build-only MSBuild package (no CLI, no config files). At build it finds the latest version tag, counts commits since ("height"), and stamps the standard version properties — `Version`, `AssemblyVersion`, `FileVersion`, `AssemblyInformationalVersion`, `PackageVersion`. One annotated `v0.1.0` tag *is* the version; new releases are new tags.

```xml
<!-- in the .csproj, or Directory.Build.props for the whole solution -->
<ItemGroup>
  <PackageReference Include="MinVer" Version="7.0.0" PrivateAssets="All" />
</ItemGroup>
<PropertyGroup>
  <MinVerTagPrefix>v</MinVerTagPrefix>
</PropertyGroup>
```

- `PrivateAssets="All"` keeps MinVer build-only (out of runtime/downstream deps).
- `MinVerTagPrefix=v` matches `v0.1.0` tags. **Without it, MinVer expects bare `0.1.0` tags and silently ignores `v`-prefixed ones** — the most common footgun.
- On tag `v1.2.3` → `1.2.3`; 4 commits later → `1.2.4-alpha.0.4`; no tag → `0.0.0-alpha.0.<height>`. `AssemblyInformationalVersion` carries the full string — read that one.

### Runtime read (C#)

```csharp
using System.Reflection;

string version =
    Assembly.GetEntryAssembly()!
        .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?
        .InformationalVersion
    ?? "unknown";
```

**The .NET 8+ `+<sha>` caveat:** the SDK auto-appends the commit SHA to `AssemblyInformationalVersion` (Source Link), e.g. `1.2.4-alpha.0.4+1a2b3c4d`. For a clean display version, `version.Split('+')[0]`. To suppress it entirely, set `<IncludeSourceRevisionInInformationalVersion>false</IncludeSourceRevisionInInformationalVersion>` (the `+sha` is handy for diagnostics, though — consider keeping it and trimming only for UI).

### Plain MSBuild fallback (no package)

For teams who won't take a build dependency — CI computes the version and passes it in:

```bash
VERSION=$(git describe --tags --always --dirty | sed 's/^v//')
dotnet publish -c Release -p:Version="$VERSION" -p:InformationalVersion="$VERSION"
```

`GenerateAssemblyInfo` is on by default for SDK-style projects, so these become assembly attributes; the runtime-read snippet above is unchanged. Keep a `<Version>0.0.0-dev</Version>` default for local builds. (The `git describe` string isn't strict semver — fine for an app's `InformationalVersion`, but reshape it if you also `pack` a NuGet package.)

**When to pick Nerdbank.GitVersioning instead:** if you want versioning *without* tags (it reads a committed `version.json` + git height, so shallow/tag-less clones still work) or deep cloud-CI integration (`nbgv` CLI exports the version to pipeline variables for Docker tags etc.). For the simple "one tag is the version" scheme, MinVer is the lighter fit.

### Pitfalls

- Shallow CI clones (see top) → wrong `0.0.0-alpha.0.N` version. GitHub Actions: `fetch-depth: 0` (add `filter: tree:0` for a cheap full history).
- Legacy non-SDK csproj with hand-written `AssemblyInfo.cs` collides with auto-generated attributes — remove the manual `[assembly: AssemblyInformationalVersion]` or set `<GenerateAssemblyInfo>false</GenerateAssemblyInfo>`.
- Put MinVer config in one root `Directory.Build.props` so every project versions identically.

---

## Python

For **installed packages**, don't shell out to git at runtime — a build-backend plugin reads the tag at build time and bakes it into package metadata. For **standalone apps** that are never `pip install`ed, inject an env var at build.

### Idiom: setuptools_scm (setuptools backend)

```toml
[build-system]
requires = ["setuptools>=80", "setuptools-scm[simple]>=9"]
build-backend = "setuptools.build_meta"

[project]
name = "your-package"
dynamic = ["version"]                 # NOT a static version = "..."

[tool.setuptools_scm]
version_file = "src/your_package/_version.py"   # optional generated module
```

The presence of `setuptools-scm[simple]` in `requires` + `version` in `dynamic` activates inference; the `[tool.setuptools_scm]` table is only needed for options like `version_file` (the current name — `write_to` is deprecated). Add the generated `_version.py` to `.gitignore`.

### Idiom: hatch-vcs (Hatchling backend)

```toml
[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[project]
name = "your-package"
dynamic = ["version"]

[tool.hatch.version]
source = "vcs"
```

Use hatch-vcs if the project already uses Hatchling; setuptools_scm if it uses setuptools. Other backends: **PDM** → `[tool.pdm.version] source = "scm"`; **Poetry** → `poetry-dynamic-versioning` plugin; **uv** → use the Hatchling backend with hatch-vcs (or `uv-dynamic-versioning`). Don't switch backends just for versioning.

### Runtime read

```python
from importlib.metadata import version, PackageNotFoundError
try:
    __version__ = version("your-package")   # the *distribution* name, may differ from import name
except PackageNotFoundError:
    __version__ = "0.0.0+unknown"           # running from a non-installed tree
```

`importlib.metadata.version()` is the official way for anything `pip install`ed (including editable installs). If you set `version_file`, you can instead `from your_package._version import __version__` (useful for frozen/zipped apps).

### Standalone apps (never installed)

- **Simplest:** compute `git describe --tags --always --dirty` in CI/Dockerfile, inject as `APP_VERSION`, read `os.environ.get("APP_VERSION", "dev")` at startup. No build backend needed.
- Or generate `_version.py` at build (not committed — avoids stale values) and import it.

### Pitfalls

- **The runtime version is PEP 440-normalized, not the raw describe string.** On an exact tag: `0.1.0` (the `v` is stripped). 32 commits past, dirty: `0.1.1.dev32+g<sha>.d<date>` (patch bumped, `.dev` counter, sha as `+local`). A `+local` version can't be uploaded to PyPI — expected for dev builds.
- Shallow CI clones (see top). setuptools_scm has `fail_on_shallow`/`fetch_on_shallow` options.
- **Container builds need `.git` at build time.** Either build the wheel in a stage that has `.git` (don't `.dockerignore` it in the build stage) and copy only the artifact into the runtime image, or pass the version explicitly via `SETUPTOOLS_SCM_PRETEND_VERSION` (PDM: `PDM_BUILD_SCM_VERSION`) as a build-arg → env. This also makes builds reproducible/offline.

---

## Go

**Idiom:** `-ldflags -X` overwrites a package-level string var at build time. (Go 1.18+ `-buildvcs` stamping via `runtime/debug.ReadBuildInfo` is a zero-config fallback but gives only commit hash + dirty, not a tag-based semver — use ldflags when you want the `git describe` string.)

```go
// main.go — overwritten at build via -ldflags
var version = "unknown"
```

```sh
go build -ldflags "-X 'main.version=$(git describe --tags --always --dirty)'" -o app .
```

Quote the whole `-X` token (`-X 'main.version=...'`). The target is `importpath.VarName` — for `package main` that's `main.version`; the var must be a non-const `string`. Verify it took with `go version -m app` (a wrong import path silently no-ops).

**Pitfalls:** inside Docker the build stage has no `.git`, so `git describe` can't run in-image and `-buildvcs` fails — resolve on the host and pass the value as a build-arg into the ldflags (or `go build -buildvcs=false`). See [Containers](#containers--docker).

---

## Rust

**Idiom:** a `build.rs` emits `cargo:rustc-env`, read with `env!`. Baseline without git is `env!("CARGO_PKG_VERSION")` (from `Cargo.toml`).

```rust
// build.rs — minimal, no deps
use std::process::Command;
fn main() {
    let v = Command::new("git")
        .args(["describe", "--tags", "--always", "--dirty"])
        .output().ok().filter(|o| o.status.success())
        .map(|o| String::from_utf8_lossy(&o.stdout).trim().to_string())
        .unwrap_or_else(|| "unknown".into());
    println!("cargo:rustc-env=APP_VERSION={v}");
    println!("cargo:rerun-if-changed=.git/HEAD");   // re-stamp when HEAD moves
}
```

```rust
const VERSION: &str = env!("APP_VERSION");   // compile-time; use option_env! if it may be absent
```

Batteries-included alternative: the `vergen-gitcl` crate (v10) emits `VERGEN_GIT_DESCRIBE` and handles `rerun-if-changed` for you.

**Pitfalls:** without `cargo:rerun-if-changed=.git/HEAD` the stamp goes stale (cargo caches `build.rs`); `env!` fails the build if the var is missing (use `option_env!`); inside Docker the build stage has no `.git` — inject `APP_VERSION` as a build-arg and read `env!`/`option_env!`.

---

## JVM (Java / Kotlin)

### Gradle

```kotlin
// build.gradle.kts — pure version string
plugins { id("com.palantir.git-version") version "5.0.0" }
version = gitVersion()                    // ≈ git describe --tags, + ".dirty" when dirty
// versionDetails() exposes .lastTag, .commitDistance, .gitHash, .isCleanTag

tasks.jar { manifest { attributes("Implementation-Version" to project.version) } }
```

For release/tag automation use `axion-release` (`pl.allegro.tech.build.axion-release`, v1.21.2; `version = scmVersion.version`).

```java
// runtime — reads Implementation-Version from the JAR manifest
String v = MyApp.class.getPackage().getImplementationVersion();
```

`getImplementationVersion()` returns `null` from an IDE/raw-classpath run (works only from the built JAR). **Spring Boot fat JARs** relocate classes, so read `BuildProperties` (Actuator) instead — or use the Maven `git.properties` approach below.

### Maven

```xml
<!-- writes git.properties into the build output -->
<plugin>
  <groupId>io.github.git-commit-id</groupId>
  <artifactId>git-commit-id-maven-plugin</artifactId>
  <version>9.2.0</version>
  <executions><execution><goals><goal>revision</goal></goals><phase>initialize</phase></execution></executions>
  <configuration><generateGitPropertiesFile>true</generateGitPropertiesFile></configuration>
</plugin>
```

```java
var p = new java.util.Properties();
try (var in = MyApp.class.getResourceAsStream("/git.properties")) { p.load(in); }
String version = p.getProperty("git.commit.id.describe");   // the git describe string
```

Spring Boot Actuator auto-detects `git.properties` and exposes it at `/actuator/info`. CI-friendly alternative: `<version>${revision}</version>` + `mvn -Drevision="$(git describe …)"` — but then you **must** add `flatten-maven-plugin` or you'll ship an unresolved `${revision}` to consumers.

**Pitfalls:** old git-commit-id groupId `pl.project13.maven` is deprecated (now `io.github.git-commit-id`); set `<failOnNoGitDirectory>false</failOnNoGitDirectory>` for builds without `.git`.

---

## Containers / Docker

A container has no `.git` — resolve `git describe` on the host and pass it in as a build-arg, then re-expose it as an env var so the running app can read it.

```dockerfile
# .dockerignore  →  add a line:  .git

FROM alpine AS runtime
ARG VERSION=unknown          # MUST be re-declared in every stage that uses it
ENV APP_VERSION=$VERSION     # promote build-arg → runtime env
LABEL org.opencontainers.image.version="$VERSION"
```

```sh
docker build --build-arg VERSION="$(git describe --tags --always --dirty)" -t myapp .
```

Document that build line in a header comment so nobody builds without it.

**Nuances:**
- **Multi-stage scope:** each `FROM` opens a fresh scope — re-declare `ARG VERSION` in every stage that needs it. An `ARG` before the first `FROM` reaches only the `FROM` lines.
- **`ARG` is build-time only**; re-expose as `ENV` to make it visible to the running container.
- **OCI labels are the standard stamping spot:** `org.opencontainers.image.version` (release / `git describe`) and `.revision` (commit SHA). Tooling reads these uniformly.
- **Build-inside-Docker exception:** if you compile a Go/Rust binary or build a Python wheel *inside* the Docker build, the build stage also needs the version — but the context excludes `.git`. Pass it as the build-arg and feed it to the compiler (Go `-ldflags "-X 'main.version=$VERSION'"`, Rust `APP_VERSION=$VERSION cargo build`, Python `SETUPTOOLS_SCM_PRETEND_VERSION=$VERSION`). Don't rely on Go `-buildvcs` / vergen / setuptools_scm reading git in-image.
