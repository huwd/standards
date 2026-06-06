# Reviewing Dependabot PRs

A structured process for Claude to assess and action Dependabot pull requests
autonomously, with a clear escalation path for uncertain cases.

## Step 1 — Gather context

Before assessing anything, collect these three things:

1. **Dependabot config** — read `.github/dependabot.yml`. Confirm that a
   cooldown (e.g. `cooldown: 7`) is configured for the relevant ecosystem.
   If it isn't, note this as a gap and flag it to @huwd before proceeding.

2. **Security advisories** — read the PR description. Dependabot includes
   advisory details when a PR is raised in response to a known CVE. Note
   whether this is an advisory-driven PR or a routine version bump.

3. **CI status** — check all CI jobs on the Dependabot branch. Note which
   passed, which failed, and which (if any) are pending.

Also note:

- **Direct vs transitive** — is the updated package listed in the primary
  manifest (`Gemfile`, `pyproject.toml`, `package.json`, etc.) or only in
  the lock file? Transitive-only updates carry lower regression risk.
- **Package category** — does the package touch auth, cryptography, or
  network/HTTP handling? Flag these for extra scrutiny regardless of semver.

---

## Step 2 — Hard gate: CI must pass

If any required CI job has failed, **do not merge**. Leave a comment
explaining which job failed and why merging is blocked. If the PR is
advisory-driven (a known CVE), tag @huwd so they can decide whether to
investigate the failure or merge with a manual override.

---

## Step 3 — Advisory fast path

If the PR was raised because of a known security advisory:

- CI passes → **merge immediately** with a comment citing the advisory and
  confirming CI passed. Do not apply the 7-day cooldown logic — sitting on
  a known vulnerability to wait out a cooldown inverts the risk.
- CI fails → block and tag @huwd (see step 2).

---

## Step 4 — Routine update assessment

For non-advisory PRs, work through these checks in order.

### 4a. Cooldown verification

Confirm the 7-day cooldown is configured (step 1). This is the primary
supply-chain protection — it gives the ecosystem time to detect and pull
malicious releases before they land in production. If the cooldown is
absent, do not merge; raise a PR to add it and tag @huwd.

### 4b. Package risk tier

If the updated package touches **auth, cryptography, or network/HTTP
handling**, note this explicitly in your assessment and apply the same
scrutiny as a suspected semver violation (step 4d), regardless of the
stated semver bump.

### 4c. Semver accuracy

Read the changelog or commit titles for the version range being bumped.
Assess whether the stated semver level (patch / minor / major) accurately
reflects the changes:

- **Patch** — bug fixes, no interface change. Straightforward.
- **Minor** — new functionality, backwards-compatible. Verify no existing
  interface was altered or deprecated in a way that could affect current
  usage.
- **Major** — breaking changes expected. Always escalate (see step 4d).

Ask: does the changelog contain anything that looks like a breaking change
regardless of what the version number says? Removals, renames, changed
return types, dropped runtime support, and behaviour changes in existing
APIs are all red flags.

### 4d. Semver violation or suspected breaking change

If you assess the change as actually breaking — whether the PR is labelled
major or you believe a minor/patch bump understates it — **escalate
immediately**. Do not attempt a full compatibility analysis; the risk of a
plausible-but-wrong "it's fine" assessment is worse than no assessment.

Escalation comment (see template below).

### 4e. Confident it's safe

If CI passes, the cooldown is in place, the package is not high-risk, and
the semver assessment is consistent with the changelog:

- Post a comment summarising your findings (see template below).
- Merge via **merge commit** (not squash — preserve the Dependabot commit
  for audit trail).

---

## Comment templates

### Merge comment

```
**Dependabot review — merging**

- CI: all required jobs passed
- Advisory: [none / CVE-YYYY-XXXXX — merged on fast path]
- Dependency type: [direct / transitive]
- Package category: [standard / ⚠️ auth/crypto/network — reviewed closely]
- Semver assessment: [patch/minor/major] bump appears accurate —
  [one sentence summary of what changed]
- Cooldown: [configured (Nd) / n/a — advisory fast path]

Merging.
```

### Escalation comment (tag @huwd)

```
**Dependabot review — escalating to @huwd**

**Concern:** [one sentence — e.g. "Changelog describes removal of X, but
the bump is labelled minor" or "This is a cryptographic library and the
patch notes mention a behaviour change in key derivation."]

**Context:**
- Package: [name] [old version] → [new version]
- CI: [passed / failed — see job Y]
- Dependency type: [direct / transitive]

**Suggested next steps:**
- [ ] [Specific thing to check — e.g. "Search the codebase for calls to
      the removed method X and assess impact"]
- [ ] [e.g. "Read the full migration guide at [URL] and check against our
      usage in [file]"]
- [ ] Merge or close this PR once assessed.
```

---

## What this process does not cover

- **Major version bumps** — always escalate; do not attempt to assess
  compatibility.
- **PRs with pending (not yet complete) CI** — wait for CI to finish before
  acting; do not merge on a partial green.
- **Grouped Dependabot PRs** — assess each package in the group
  individually; if any single package warrants escalation, escalate the
  whole PR.
