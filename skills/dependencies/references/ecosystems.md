# Package manager commands

Flags change between major versions. Before relying on a command you haven't used with this exact version, check `<tool> <command> -h` (or `--help`). Run everything from the project or workspace root, through the project's version of the tool (Corepack, `mise`, `asdf`, the wrapper script).

## JavaScript and TypeScript

| Task | npm | pnpm | Yarn Berry (2+) | Bun |
|---|---|---|---|---|
| Add | `npm install <pkg>@<ver>` | `pnpm add <pkg>@<ver>` | `yarn add <pkg>@<ver>` | `bun add <pkg>@<ver>` |
| Add as dev | `-D` | `-D` | `-D` | `-d` |
| Exact version | `-E` | `-E` | `-E` | `-E` |
| In a workspace | `-w <workspace>` | `--filter <name>` | `yarn workspace <name> add ...` | run in the workspace directory |
| Remove | `npm uninstall <pkg>` | `pnpm remove <pkg>` | `yarn remove <pkg>` | `bun remove <pkg>` |
| Upgrade one | `npm install <pkg>@<ver>` | `pnpm update <pkg>@<ver>` | `yarn up <pkg>@<ver>` | `bun update <pkg>` or `bun add <pkg>@<ver>` |
| Install from lockfile only | `npm ci` | `pnpm install --frozen-lockfile` | `yarn install --immutable` | `bun install --frozen-lockfile` |
| Outdated | `npm outdated` | `pnpm outdated` | no built-in command; `yarn npm info <pkg> --fields versions` lists published versions | `bun outdated` |
| Why is it here | `npm explain <pkg>`, `npm ls <pkg>` | `pnpm why <pkg>` | `yarn why <pkg>` | `bun pm why <pkg>` |
| Audit | `npm audit --omit=dev` | `pnpm audit` | `yarn npm audit` | `bun audit` |
| Install scripts pending review | `npm approve-scripts --allow-scripts-pending` | `pnpm ignored-builds` | n/a | `bun pm untrusted` |
| Approve one package's scripts | `npm approve-scripts <pkg>` | `pnpm approve-builds <pkg>` | `dependenciesMeta` in `package.json` | `bun pm trust <pkg>` |

Yarn Classic (1.x): `yarn add`, `yarn remove`, `yarn upgrade <pkg>@<ver>`, `yarn install --frozen-lockfile`, `yarn outdated`, `yarn why`, `yarn audit`.

Supply-chain settings to respect (don't lower or remove them):

- **npm 11:** `min-release-age` (days) and `before` in `.npmrc`; `allowScripts` in `package.json` (maintained with `npm approve-scripts` and `npm deny-scripts`; currently advisory unless `strict-allow-scripts=true`); `ignore-scripts`; `allow-git`.
- **pnpm 11+:** settings live in `pnpm-workspace.yaml` (pnpm 11 no longer reads the `pnpm` field of `package.json`, and reads only auth and registry settings from `.npmrc`). `minimumReleaseAge` defaults to 1440 minutes (one day); `allowBuilds` maps package names to `true` or `false`; `strictDepBuilds` (on by default) fails the install when an unapproved package wants to build; `blockExoticSubdeps` (on by default) blocks git and tarball sources in transitive dependencies; `trustPolicy: no-downgrade` rejects versions published with weaker provenance than earlier ones.
- **Yarn Berry:** `enableScripts`, `dependenciesMeta.<pkg>.built`, and, in recent Yarn 4 releases, `npmMinimalAgeGate` in `.yarnrc.yml`.
- **Bun:** dependency lifecycle scripts run only for packages in `trustedDependencies` in `package.json`; without that field, Bun uses its built-in default list (`bun pm default-trusted`), and defining the field replaces that list. Release-age delay: `[install] minimumReleaseAge` (seconds) in `bunfig.toml`, or `--minimum-release-age`.

`pnpm approve-builds` with no package names, and `yarn upgrade-interactive`, prompt interactively; pass explicit names or use the non-interactive commands above.

## Python

| Task | uv (project) | Poetry 2 | pip with requirements files |
|---|---|---|---|
| Add | `uv add '<pkg>>=<ver>'` or `'<pkg>==<ver>'` | `poetry add '<pkg>@^<ver>'` | Add the line to `requirements.in` (pip-tools) or `requirements.txt`, then compile/install |
| Add as dev | `--dev` or `--group <name>` | `--group dev` | separate `requirements-dev.in` |
| Remove | `uv remove <pkg>` | `poetry remove <pkg>` | Delete the line; recompile; `python -m pip uninstall <pkg>` |
| Upgrade one | `uv lock --upgrade-package <pkg>`, then `uv sync` | `poetry update <pkg>` | `uv pip compile --upgrade-package <pkg> requirements.in -o requirements.txt` or `pip-compile --upgrade-package <pkg>` |
| Install from lockfile only | `uv sync --locked` | `poetry sync` | `python -m pip install -r requirements.txt` (add `--require-hashes` if the file has hashes) |
| Outdated | `uv tree --outdated` | `poetry show --outdated` | `python -m pip list --outdated` |
| Why is it here | `uv tree --invert --package <pkg>` | `poetry show --tree` | `pipdeptree -r -p <pkg>` (if installed) |
| Audit | `uv audit` | `osv-scanner` on `poetry.lock` (pip-audit can't read it; export a requirements file first) | `pip-audit -r requirements.txt` |

- Install into the project's virtual environment, never the system or Homebrew Python. Check with `python -c 'import sys; print(sys.prefix)'`. An "externally-managed-environment" error (PEP 668) means you're about to install into the system Python: create or activate a venv; don't pass `--break-system-packages`.
- Installing from a source distribution runs its build code. `--only-binary :all:` restricts pip to wheels when you're evaluating an unfamiliar package.
- uv settings to respect: `exclude-newer` and `exclude-newer-package` in `[tool.uv]`, index and source pinning (`[[tool.uv.index]]`, `tool.uv.sources`).
- Pipenv: `pipenv install <pkg>`, `pipenv install --dev`, `pipenv uninstall`, `pipenv update <pkg>`, `pipenv sync`. PDM: `pdm add`, `pdm remove`, `pdm update <pkg>`, `pdm sync`.

## Rust (Cargo)

- Add: `cargo add <crate>@<ver>`; `--dev`, `--build`, `--features a,b`. Remove: `cargo remove <crate>`.
- Upgrade one: `cargo update -p <crate>` (within the version range), `cargo update -p <crate> --precise <ver>`; change the range in `Cargo.toml` for a new major.
- Lockfile only: `cargo build --locked` (or `--frozen` offline).
- Inspect: `cargo info <crate>` (Cargo 1.82+), `cargo tree -i <crate>`, `cargo update --dry-run`.
- Audit: `cargo audit` (cargo-audit) or `cargo deny check` (cargo-deny, also checks licenses and sources).
- `build.rs` scripts and procedural macros run at build time. Vet them like install scripts.

## Go modules

- Add or upgrade: `go get <module>@<version>`, then `go mod tidy`. A new major version is a different import path (`example.com/mod/v2`).
- Remove: delete the imports, then `go mod tidy`.
- Tools (Go 1.24+): `go get -tool <module>/cmd/<tool>@<version>`.
- Inspect: `go list -m -versions <module>`, `go list -m -u all` (available updates), `go mod why -m <module>`, `go mod graph`.
- Verify and audit: `go mod verify`, `govulncheck ./...` (reports vulnerabilities in code paths you actually call).
- Respect `GOPROXY`, `GOSUMDB`, `GOPRIVATE`, `GONOSUMDB`, and `GOFLAGS`. Never set `GOSUMDB=off`, `GOINSECURE`, or `GONOSUMDB` for public modules to get past a checksum error.

## Ruby (Bundler)

- Add: `bundle add <gem> --version '~> <ver>'` (`--group development,test`). Remove: `bundle remove <gem>`.
- Upgrade one: `bundle update <gem> --conservative` (doesn't upgrade shared dependencies).
- Lockfile only: `BUNDLE_FROZEN=true bundle install`.
- Inspect: `bundle outdated`, `bundle info <gem>`.
- Audit: `bundle-audit check --update` (bundler-audit).
- Gems with native extensions compile and run code at install time.

## PHP (Composer)

- Add: `composer require <vendor>/<pkg>:^<ver>` (`--dev`). Remove: `composer remove <vendor>/<pkg>`.
- Upgrade one: `composer update <vendor>/<pkg> --with-dependencies`.
- Lockfile only: `composer install`.
- Inspect: `composer outdated --direct`, `composer why <vendor>/<pkg>`, `composer show -a <vendor>/<pkg>`.
- Audit: `composer audit`.
- Composer plugins run code; `allow-plugins` in `composer.json` lists the permitted ones. Add specific entries after review; never set it to `true` wholesale.

## Java and Kotlin

- **Maven:** edit `pom.xml` (or the BOM in `dependencyManagement`). Inspect with `mvn -q dependency:tree -Dincludes=<groupId>:<artifactId>` and `mvn versions:display-dependency-updates` (versions-maven-plugin). Audit with the OWASP `dependency-check-maven` plugin or `osv-scanner` on `pom.xml`.
- **Gradle:** edit the build script or the version catalog (`gradle/libs.versions.toml`). Inspect with `./gradlew dependencies --configuration runtimeClasspath` and `./gradlew dependencyInsight --dependency <name> --configuration runtimeClasspath`. If the project uses dependency locking, update with `./gradlew dependencies --write-locks`; if it has `gradle/verification-metadata.xml`, regenerate checksums with `./gradlew --write-verification-metadata sha256 help` and review the diff.
- Maven and Gradle plugins execute code during the build. Vet new plugins like dependencies.

## .NET (NuGet)

- Add: `dotnet add <project> package <Name> --version <ver>`; with central package management, set the version in `Directory.Packages.props` and add a versionless `PackageReference`.
- Remove: `dotnet remove <project> package <Name>`.
- Lockfile only: `dotnet restore --locked-mode` (projects with `packages.lock.json`).
- Inspect: `dotnet list package --outdated`, `dotnet nuget why <project> <Name>` (recent SDKs).
- Audit: `dotnet list package --vulnerable --include-transitive`; NuGet also audits during restore on .NET 8+. Don't turn off `NuGetAudit`.

## Multi-ecosystem scanning

`osv-scanner scan source -r .` reads most lockfiles (npm, pnpm, Yarn, Bun, uv, Poetry, Pipenv, requirements files, Cargo, Go, Bundler, Composer, Maven, Gradle, NuGet) and checks them against the OSV database. It's a separate install; ask before adding it.
