---
name: do-release
description: Run or prepare a package release. Use when the user asks to cut, prepare, tag, publish, or verify a release for a repo, including RubyGems, npm, Cargo, PyPI, or tag-driven GitHub Actions releases.
---

# Do Release

Use this workflow to prepare and execute a repository release without assuming a specific package ecosystem. Discover the repo release mechanism first, then follow the local process exactly.

## Safety Rules

Release actions are state-changing. Do not push release commits, push tags, create GitHub releases, publish packages, or trigger non-dry-run release workflows without explicit user approval.

Never print, read, or persist package registry tokens. Prefer trusted publishing, OIDC, npm provenance, or existing CI secrets over local credentials.

If the repository release process is ambiguous, stop after preparing a release checklist and ask before choosing a path.

## Discover the Release Mechanism

Read the repo from the root. Gather:

```bash
git status --short --branch
git remote -v
rg -n "release|publish|rubygems|npm publish|cargo publish|twine|trusted publishing|provenance|workflow_dispatch|tags:" . --glob ".github/workflows/**" --glob "README*" --glob "CHANGELOG*" --glob "package.json" --glob "Cargo.toml" --glob "pyproject.toml" --glob "*.gemspec" --glob "Rakefile" --glob "Makefile" --glob "justfile" 2>/dev/null
rg --files | rg "(^CHANGELOG|CHANGELOG.md$|package.json$|Cargo.toml$|pyproject.toml$|gemspec$|version|VERSION|release)"
```

Identify:

- package ecosystem and package name
- current version source of truth
- changelog file and format
- release trigger: tag push, GitHub Release, manual workflow, registry command, or project script
- tag format, such as `v1.2.3` or `1.2.3`
- CI/release checks that must pass
- publishing path: trusted CI publishing, registry token, local publish command, or manual workflow

Prefer the repository workflow over generic ecosystem habits. If `.github/workflows/release.yml` exists, treat it as authoritative.

## Version Sources

Find and update the version in the ecosystem-native source of truth:

| Ecosystem | Common version source | Common checks |
|-----------|------------------------|---------------|
| RubyGems | `lib/**/version.rb`, `*.gemspec` | `bundle exec rake build`, `gem build`, `gem install pkg/*.gem` |
| npm | `package.json`, lockfile | `npm pack --dry-run`, `npm publish --dry-run`, package manager tests |
| Cargo | `Cargo.toml`, `Cargo.lock` for binaries/examples as needed | `cargo test`, `cargo package`, `cargo publish --dry-run` |
| PyPI | `pyproject.toml`, package `__version__` if used | `python -m build`, `twine check dist/*` |
| Go modules | tags are often source of truth | `go test ./...`, module-aware tag format if monorepo |

All visible version data must agree before tagging. Search for the old version to catch docs, examples, install snippets, and lockfiles:

```bash
rg -n "<old-version>|<old_version>|version" .
```

Do not update generated lockfiles or package metadata unless the ecosystem expects it.

## Choose the Next Version

If the user gave a version, use it. Otherwise infer the smallest SemVer bump from merged changes and changelog intent:

- patch: bug fixes, docs-only package metadata fixes, internal compatibility fixes
- minor: new backwards-compatible functionality
- major: breaking API, runtime, packaging, or support-policy change
- prerelease: only when the repo already uses prerelease tags or the user asks

Confirm the proposed version with the user before editing if the bump is not obvious.

## Prepare the Release Commit

1. Start from a clean branch based on the release branch, usually `main`.
2. Update version sources.
3. Update `CHANGELOG.md` or the repo changelog with:
   - release heading for the exact version
   - release date if the changelog style uses dates
   - user-facing changes grouped consistently with existing style
   - migration notes for breaking changes
4. Update package metadata if needed: README install examples, gemspec/package metadata, lockfiles, generated docs, or manifests.
5. Run the repo checks that mirror CI. If a release workflow has preflight checks, run local equivalents.

For tag-driven RubyGems workflows that read a Ruby constant, verify:

```bash
ruby -r ./lib/<package>/version -e "puts <Module>::VERSION"
grep -Eq "^##[[:space:]]+\[?<version>\]?([[:space:]-]|$)" CHANGELOG.md
bundle exec rspec
bundle exec rubocop
bundle exec rake build
gem install pkg/*.gem --no-document
ruby -r <package> -e "puts <Module>::VERSION"
```

Adapt `<package>` and `<Module>` from the repo. If the workflow performs extra checks, such as documentation generation, run those too.

For npm:

```bash
npm test
npm run build --if-present
npm pack --dry-run
```

For Cargo:

```bash
cargo test
cargo package
cargo publish --dry-run
```

For PyPI:

```bash
python -m build
python -m twine check dist/*
```

Commit the release prep using the repo commit standard. A typical subject is:

```text
chore: prepare v<version> release
```

## Tag and Publish

Before any state-changing release action, show the user:

- version
- tag name
- release commit SHA
- changelog excerpt to use as release notes
- local checks run and results
- exact commands you plan to run

Then ask for approval.

For tag-driven GitHub Actions releases:

```bash
git tag -a v<version> -m "Release v<version>"
git push origin v<version>
```

If the workflow expects lightweight tags, use the repo convention instead. After pushing the tag, watch the release workflow until it finishes.

```bash
gh run list --workflow release.yml --limit 5
gh run watch <run-id>
```

If the release workflow supports `workflow_dispatch` with `dry_run`, use dry-run first when available:

```bash
gh workflow run release.yml -f dry_run=true
```

Only trigger the non-dry-run workflow after user approval.

## Release Notes

Use the changelog entry as the source of truth for release notes. If the repo uses GitHub Releases, create or update the release after the tag exists and before calling the task complete.

```bash
gh release create v<version> --title "v<version>" --notes-file /tmp/release-notes-v<version>.md
```

If the tag already has a release, update it instead of creating a duplicate:

```bash
gh release edit v<version> --title "v<version>" --notes-file /tmp/release-notes-v<version>.md
```

Do not invent release notes. If the changelog is incomplete, update the changelog first.

## Verify Publication

Confirm the package ecosystem sees the new version. Use the command appropriate to the detected ecosystem:

```bash
# RubyGems
gem info <gem-name> --remote --version <version>

# npm
npm view <package-name>@<version> version

# Cargo
cargo search <crate-name> --limit 5

# PyPI
python -m pip index versions <package-name>
```

Also verify the GitHub tag and release:

```bash
git ls-remote --tags origin "v<version>"
gh release view v<version>
```

If publishing is delayed by registry indexing, say that explicitly and show the workflow success URL or run id.

## Failure Handling

If any preflight fails, stop before tagging. Fix the release prep and rerun checks.

If tag push succeeds but publish fails:

- do not delete or move the tag unless the user explicitly asks
- inspect the failed workflow logs
- fix forward with a new commit and, if required, a new patch/prerelease version
- if the failure is credential/trusted-publishing configuration, report the exact missing setup without exposing secrets

If the package version already exists in the registry, do not reuse the version. Prepare a new version.

## Final Report

End with:

- release version and tag
- commit SHA
- checks run
- release workflow status
- package registry verification result
- GitHub release URL, if created
