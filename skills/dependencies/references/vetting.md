# Vetting a package before installing it

The goal is to answer two questions with evidence: is this the package I mean, and is it reasonable to trust it? All the lookups below are read-only and run nothing from the package. Use the non-interactive network settings your environment needs, and never print registry tokens.

## 1. Look up the exact name

**npm**

```bash
npm view <pkg> name version description repository.url homepage license maintainers \
  time.created time.modified deprecated dist.unpackedSize dependencies scripts --json
npm view <pkg> time --json                                          # every version's publish date
curl -sS https://api.npmjs.org/downloads/point/last-week/<pkg>      # works for scoped names too
```

**PyPI**

```bash
curl -sS https://pypi.org/pypi/<pkg>/json | jq '{name: .info.name, version: .info.version,
  summary: .info.summary, urls: .info.project_urls,
  license: (.info.license_expression // .info.license), requires: .info.requires_dist}'
curl -sS https://pypi.org/pypi/<pkg>/json | jq -r '.releases | to_entries
  | map(select(.value | length > 0)) | sort_by(.value[0].upload_time)
  | .[-10:][] | "\(.key) \(.value[0].upload_time)"'                # last 10 releases with dates
curl -sS https://pypistats.org/api/packages/<pkg>/recent             # downloads
```

**crates.io** (the API requires a User-Agent that identifies you)

```bash
cargo info <crate>
curl -sS -A '<project-name> (<contact>)' https://crates.io/api/v1/crates/<crate> \
  | jq '.crate | {name, max_stable_version, downloads, recent_downloads, repository, created_at, updated_at}'
```

**Go** (in proxy URLs, each uppercase letter in a module path is written as `!` plus the lowercase letter)

```bash
go list -m -versions <module>
curl -sS https://proxy.golang.org/<module>/@latest
```

**RubyGems, Packagist, NuGet, Maven Central**

```bash
curl -sS https://rubygems.org/api/v1/gems/<gem>.json | jq '{name, version, downloads, source_code_uri, homepage_uri, licenses}'
curl -sS https://packagist.org/packages/<vendor>/<pkg>.json | jq '.package | {name, repository, downloads: .downloads.monthly, time}'
dotnet package search <Name> --exact-match
```

For Maven Central, search `central.sonatype.com` for the group and artifact, and check that the group ID matches the coordinates the project documents.

Then confirm identity:

- The repository URL points to the project you meant, and that repository's README or docs name this exact package (including the scope, such as `@org/name`).
- The publisher or maintainers match the project's owners. For npm, `npm audit signatures` checks registry signatures and provenance attestations of installed packages.
- The name is spelled exactly as the project documents it. Compare letter by letter: `requests` vs `reqeusts`, `lodash` vs `lodahs`, `python-dateutil` vs `dateutil`, hyphen vs underscore, a missing scope.

## 2. Red flags

Stop and ask the user if any of these apply:

| Red flag | Why it matters |
|---|---|
| The package or the version you'd install was published in the last few days | Hijacked and malicious releases are usually caught within days; release-age settings exist for this reason |
| Few downloads for something that should be popular | The popular one probably has a different name |
| No repository, or a repository that doesn't mention this package | You can't review the source, or someone copied a README |
| A name one typo, separator, or scope away from a popular package | Typosquatting |
| A name you can't find in any documentation, only in generated code or your memory | Possibly a hallucinated name that someone registered (slopsquatting) |
| `preinstall`, `install`, or `postinstall` scripts, especially ones that download or run other code | The usual way malicious packages execute |
| Obfuscated or minified source in a package that isn't a build artifact | Hides behavior from review |
| A new maintainer and a new major release shortly after a long quiet period | A pattern in account takeovers |
| Many versions published in a short burst | Common in automated attacks |
| A deprecation notice, or "this package has been moved" | Use the replacement it names, after verifying that one too |

## 3. Health and fit

- **Maintenance:** date of the last release, frequency of releases, open issues and pull requests, number of maintainers.
- **Size and dependency count:** `dist.unpackedSize` and `dependencies` from `npm view`; `requires_dist` from PyPI; `cargo tree` after a trial add in a scratch project. Each transitive dependency is another package to trust.
- **Platform needs:** native builds, binary downloads at install time, required system libraries, supported runtimes and operating systems.
- **Types and docs:** bundled TypeScript types or a maintained `@types/` package; documentation for the version you'll install.
- **License:** compatible with the project. Note that `license` fields can be missing or wrong; check the repository's LICENSE file for anything unusual.

## 4. Known vulnerabilities

Query OSV for the exact version before installing:

```bash
curl -sS https://api.osv.dev/v1/query \
  -d '{"package": {"name": "<pkg>", "ecosystem": "npm"}, "version": "<version>"}' | jq -r '.vulns[]?.id'
```

Ecosystem names: `npm`, `PyPI`, `crates.io`, `Go`, `RubyGems`, `Packagist`, `Maven` (name as `group:artifact`), `NuGet`. Empty output means no known advisories for that version. After installing, run the manager's audit command (see [ecosystems.md](ecosystems.md)).

## 5. Inspect the code without running it

When a package is unfamiliar or has install scripts, look before installing:

- **npm:** `npm view <pkg>@<ver> scripts --json` shows its lifecycle scripts. `npm pack <pkg>@<ver>` downloads the tarball into the current directory without installing it; list it with `tar -tzf <file>.tgz` and read `package/package.json` and any script it runs. Do this in a temporary directory and delete the tarball afterwards.
- **PyPI:** wheels contain no install-time code. Download one for reading with `python -m pip download <pkg>==<ver> --no-deps --only-binary :all: -d <tmp-dir>` and inspect it with `unzip -l`. Source distributions run build code, so read them as archives without building.
- **Cargo:** `cargo download` isn't built in; read the source on docs.rs ("Source" view) or the repository at the tag, paying attention to `build.rs` and procedural macros.
- **Go:** module code doesn't run at install time; read it at the tagged version on the repository or pkg.go.dev.

For a first trial install of an unfamiliar npm package, `--ignore-scripts` keeps lifecycle scripts from running; check afterwards whether the package needs them to work.

## 6. Write it into the proposal

Record what you verified, with the numbers you actually saw: the registry entry, the repository, publisher, age, downloads, license, dependency count, install scripts, and the OSV result. See "Proposal" in SKILL.md.
