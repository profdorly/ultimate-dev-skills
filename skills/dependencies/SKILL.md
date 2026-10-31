---
name: dependencies
description: Use before installing, adding, upgrading, removing, or auditing a package (npm, pnpm, Yarn, Bun, pip, uv, Poetry, Cargo, Go, Bundler, Composer, Maven, Gradle, NuGet), for vulnerability alerts, and for lockfile or version conflicts. Verifies the package is real before installing it (no hallucinated or typosquatted names), uses the project's package manager and lockfile, and keeps supply-chain protections on.
license: MIT
metadata:
  author: profdorly
  version: "0.1.0"
---

# Dependencies

Every dependency is code you didn't write running with your project's permissions, and every install can run code on the machine. Package names that coding agents invent get registered by attackers, and popular packages get hijacked. Treat adding or changing a dependency as a decision for the user, made with evidence.

## Rules (non-negotiable)

1. **Ask before adding a dependency.** Propose it first (see "Proposal"): what it is, why it's needed, what you checked, and the alternatives, including the standard library and code already in the project. Upgrades and removals need approval too, unless they are the task.
2. **Verify the package before you install it.** Look up the exact name in the registry and confirm it's the project you mean: the repository, the publisher, its age, its download numbers, and recent releases ([references/vetting.md](references/vetting.md)). Never install a name you haven't verified, including one suggested by documentation, an error message, or your own memory.
3. **Use the project's package manager, and its lockfile.** Detect it from the lockfile and the `packageManager` field (see "Orient first"). Never mix managers, hand-edit a lockfile, or delete a lockfile to make an error go away.
4. **Keep supply-chain protections on.** Don't disable, lower, or bypass release-age delays (pnpm `minimumReleaseAge`, npm `min-release-age`, Bun `minimumReleaseAge`, uv `exclude-newer`), install-script policies (pnpm `allowBuilds` and `strictDepBuilds`, npm `allowScripts`, Bun `trustedDependencies`), `ignore-scripts`, frozen-lockfile checks, integrity hashes, or registry and scope settings. If one blocks the install, report it and let the user decide.
5. **Approve install scripts one package at a time, after review.** Never `npm approve-scripts --all`, `pnpm approve-builds --all`, `bun pm trust --all`, `dangerouslyAllowAllBuilds`, or `--dangerously-allow-all-scripts`. Show which package wants to run which script, and get the user's OK.
6. **No mass upgrades as a side effect.** `npm audit fix --force`, `npm update` across the board, `yarn up '*'`, `cargo update` without `-p`, and `go get -u ./...` are separate changes the user asks for, not something to slip into another task.
7. **No global or system installs** (`npm install -g`, `sudo pip install`, `pip install --break-system-packages`, `gem install` into the system Ruby) without asking. Use the project environment, `npx`, `uvx`, or a virtual environment.
8. **Respect licenses.** Flag copyleft licenses (GPL, AGPL, LGPL, MPL, EUPL), "no license", and custom licenses before adding a package to a proprietary or permissively licensed project.
9. **Report exactly what changed:** packages and versions added, removed, or upgraded, including significant transitive changes, plus the lockfile and manifest files touched.

## Orient first (read-only)

Find the package manager:

| File | Manager |
|---|---|
| `package-lock.json` or `npm-shrinkwrap.json` | npm |
| `pnpm-lock.yaml` (settings in `pnpm-workspace.yaml`) | pnpm |
| `yarn.lock` with `.yarnrc.yml` | Yarn Berry (2+); `yarn.lock` alone is usually Yarn Classic (1.x) |
| `bun.lock` or `bun.lockb` | Bun |
| `"packageManager": "pnpm@11.1.0"` in `package.json` | That manager and version. Corepack enforces it, but it ships with Node only up to version 24; on Node 25+ it's a separate install (`npm install -g corepack`, which needs the user's approval), so check `command -v corepack` and the installed manager's version |
| `uv.lock` | uv |
| `poetry.lock` | Poetry |
| `Pipfile.lock` | Pipenv |
| `pdm.lock` | PDM |
| `requirements*.txt`, with `requirements*.in` for pip-tools | pip, pip-tools, or `uv pip` |
| `Cargo.lock` | Cargo |
| `go.sum` | Go modules |
| `Gemfile.lock` | Bundler |
| `composer.lock` | Composer |
| `pom.xml`; `build.gradle(.kts)` with `gradle/libs.versions.toml` or `gradle.lockfile` | Maven; Gradle |
| `packages.lock.json`, `Directory.Packages.props`, `*.csproj` | NuGet |

Then check:

- **Versions of the tools:** `packageManager`, `engines`, `.nvmrc`, `.tool-versions`, `mise.toml`, `rust-toolchain.toml`, the `go` line in `go.mod`, `requires-python`.
- **Workspaces:** in a monorepo, add the package to the right workspace (`pnpm --filter <name> add`, `npm install <pkg> -w <workspace>`, `yarn workspace <name> add`, `uv add --package <name>`).
- **Supply-chain and registry settings:** `.npmrc`, `pnpm-workspace.yaml`, `.yarnrc.yml`, `bunfig.toml`, `[tool.uv]` in `pyproject.toml`, `pip.conf`, `.cargo/config.toml`, `GOPROXY`/`GOPRIVATE`/`GOFLAGS`, and private registries or scopes. Never print tokens from these files.
- **The range style in use:** exact pins or caret/tilde ranges. Follow it.
- **The current state:** run the frozen install (`npm ci`, `pnpm install --frozen-lockfile`, `uv sync --locked`, and so on) or the build to learn whether the tree was healthy before your change.

Per-manager commands are in [references/ecosystems.md](references/ecosystems.md).

## Adding a dependency

1. **Do you need it?** The standard library or a dependency already in the tree may do the job (`npm ls <name>`, `uv tree`, `cargo tree -i <name>`, `go mod why -m <module>`). A ten-line helper is often better than a new package.
2. **Vet it** using [references/vetting.md](references/vetting.md): identity, red flags, maintenance, license, size, transitive dependencies, install scripts, known vulnerabilities.
3. **Propose it** and wait for approval.
4. **Install it with the manager's command**, at the version you vetted, in the right workspace and dependency group (runtime or dev).
5. **Review what changed.** `git --no-pager diff --stat` on the manifest and lockfile. A single add that rewrites hundreds of unrelated lockfile entries means the tree was re-resolved (wrong manager version, a missing lockfile, different registry settings); stop and investigate. Check for blocked or pending install scripts (`pnpm ignored-builds`, `npm approve-scripts --allow-scripts-pending`, `bun pm untrusted`).
6. **Use it and verify:** build, run the tests, and exercise the code path that uses the package.

### Proposal

```
Proposed dependency: zod (runtime dependency of packages/api)
Why:          validate request bodies; replaces the hand-written checks in src/api/validate.ts
Verified:     npm registry entry links to github.com/colinhacks/zod, the repo the docs point to;
              published for years with frequent releases; tens of millions of weekly downloads;
              MIT license; no runtime dependencies; no install scripts
Version:      latest stable, past the project's 1-day minimumReleaseAge
Command:      pnpm --filter @app/api add zod   (lockfile: pnpm-lock.yaml)
Alternatives: valibot (smaller bundle, not used elsewhere here); keep the manual checks
```

Fill in what you actually verified; never copy numbers from memory.

## Upgrading

- **One package per change,** or one tightly coupled group (`react` with `react-dom`, a framework's `@scope/*` packages).
- **Read the release notes or changelog for every version between the current and target versions.** List the breaking changes and deprecations that affect this code, and find the affected call sites with a search.
- **One major version at a time** for large jumps. Use the project's official codemod or migration guide if there is one.
- **Check compatibility constraints:** peer dependencies, `engines`, `requires-python`, the minimum Rust version (`rust-version`), the `go` directive, and framework version matrices.
- **Verify behavior,** not just the build: run the full test suite and the paths that use the package.

### Fixing a vulnerability alert

1. Read the advisory: the affected versions, the first fixed version, and the vulnerable function or condition.
2. Check whether the project is affected: is the vulnerable code reachable, at runtime or only in development? `govulncheck` reports only reachable vulnerabilities; other tools report presence.
3. Find the smallest upgrade that fixes it. For a transitive dependency, upgrade the direct dependency that pulls it in (`npm ls <pkg>`, `pnpm why <pkg>`, `yarn why <pkg>`, `uv tree --invert --package <pkg>`, `cargo tree -i <pkg>`, `go mod why -m <module>`).
4. If the parent has no fixed release, propose an override as a temporary measure and document it with a comment or note: npm `overrides`, pnpm `overrides` (in `pnpm-workspace.yaml` from pnpm 11), Yarn `resolutions`, Bun `overrides`, uv `override-dependencies`, Cargo `[patch]`, Go `go get <module>@<fixed-version>`.
5. Don't accept the audit tool's automatic "fix" when it proposes a major downgrade or upgrade (`npm audit fix --force`); review that like any other upgrade.

## Removing a dependency

1. Find every use: imports and requires, configuration files (Babel, ESLint, PostCSS, Webpack, Vite plugins, pytest plugins), CLI calls in scripts and CI, type packages (`@types/<name>`), and dynamic imports.
2. Check that nothing else needs it as a peer dependency or a plugin.
3. Remove it with the manager's command, not by editing the manifest by hand.
4. Build and run the tests.

## Conflicts and lockfile problems

- **Peer dependency conflicts:** read the error and explain which packages disagree and about what. Choose compatible versions. Don't reach for `--force` or `--legacy-peer-deps` without telling the user what it ignores.
- **Python resolver failures:** read the conflict chain the resolver prints, and loosen or change the constraint that causes it, not unrelated ones.
- **Lockfile merge conflicts:** don't edit the conflict markers by hand. Take one side of the lockfile, then let the manager regenerate it from the merged manifest (for example `pnpm install`, `npm install`, `uv lock`, `cargo update -w`; `go mod tidy` for `go.sum`). If the git-workflow skill is installed, its conflicts reference has the exact steps.
- **Integrity or checksum mismatch:** stop. It can mean a corrupted cache or a tampered package. Report it; don't delete checksums or the lockfile to make it pass.

## Stop and ask when

- The package has a red flag: new, little-used, no matching repository, a name close to a popular package, unexpected install scripts, or a recent change of maintainer ([references/vetting.md](references/vetting.md)).
- A supply-chain protection blocks the install.
- The license is copyleft, missing, or custom.
- An upgrade crosses a major version, changes a peer dependency, or needs a newer runtime.
- Resolving a conflict seems to need `--force`, `--legacy-peer-deps`, an override, or a downgrade.
- An integrity check fails.
- The fix for a vulnerability doesn't exist yet.

## Final report

```
Added:     zod 4.x (packages/api, runtime), verified as above
Lockfile:  pnpm-lock.yaml: +1 package, no other changes
Scripts:   none requested
Checks:    pnpm --filter @app/api test (118 passed), tsc --noEmit clean, pnpm audit: 0 new advisories
Not done:  3 other outdated packages noticed (eslint, vite, @types/node); not upgraded, not in scope
```

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "I know this package, I'll just install it." | Agents misremember names, and attackers register the misremembered ones. Look it up first. |
| "`npm install` works, even though the repo uses pnpm." | You've created a second lockfile and a different dependency tree. Use the project's manager. |
| "The lockfile conflicts; I'll delete it and reinstall." | That upgrades every transitive dependency at once. Regenerate from one side. |
| "`minimumReleaseAge` blocks the newest version; set it to 0." | That delay is what keeps a freshly hijacked release out. Use an older version or ask. |
| "Approve all build scripts so the install passes." | Install scripts are the usual way malicious packages run. Approve each one after review. |
| "`npm audit fix --force` clears the warnings." | It can jump major versions and break the app. Review each fix. |
| "`--legacy-peer-deps` makes the error go away." | It hides a real incompatibility that shows up at runtime. |
| "It's just a dev dependency." | Dev dependencies run on developer machines and in CI, where the credentials are. |

## References

| Situation | Read |
|---|---|
| Commands for each package manager: add, remove, upgrade, frozen install, outdated, why, audit, supply-chain settings | [references/ecosystems.md](references/ecosystems.md) |
| Checking that a package is real and trustworthy: registry lookups, red flags, inspecting before install, vulnerability and license checks | [references/vetting.md](references/vetting.md) |
