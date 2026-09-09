# Workflow Charter

**Version:** 1.0.0  
**Effective Date:** 2026-09-09  
**Status:** Active

This document is the authoritative source of truth for how this team branches, reviews, releases, and recovers from failures. It is not a suggestion. Every engineer on the team is expected to follow it from the date it is adopted.

Companion documents:
- [`BRANCHING.md`](BRANCHING.md) — Naming conventions, branch lifecycle, sync policy
- [`AUDIT.md`](AUDIT.md) — Evidence of problems this charter addresses
- [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md) — Required PR structure
- [`.github/CODEOWNERS`](.github/CODEOWNERS) — Review ownership assignments

---

## Validation Gates

Every pull request targeting `main` must pass all of the following automated checks before it is eligible to merge. A single failing gate blocks the merge. There are no exceptions.

---

### Gate 1 — Lint

| Field | Detail |
|---|---|
| **Gate name** | Lint |
| **What it checks** | Code style, formatting rules, and static analysis violations across all modified files |
| **Failure means** | The PR cannot merge. The author must fix all reported lint errors and push a new commit. Reviewers do not need to re-review formatting; the gate handles it. |
| **Configured in** | `.github/workflows/ci.yml` — job `lint` |

---

### Gate 2 — Unit Tests

| Field | Detail |
|---|---|
| **Gate name** | Unit Tests |
| **What it checks** | All unit tests in the repository. Tests must exit with code 0 and coverage must not drop below the configured threshold. |
| **Failure means** | The PR cannot merge. The author must either fix the failing tests or, if the tests themselves are wrong, update them with a justification in the PR description. |
| **Configured in** | `.github/workflows/ci.yml` — job `test` |

---

### Gate 3 — Security Scan (Dependency Audit)

| Field | Detail |
|---|---|
| **Gate name** | Security Scan |
| **What it checks** | All declared dependencies in `package.json` and `package-lock.json` against known CVE databases. Flags high and critical severity vulnerabilities. |
| **Failure means** | The PR cannot merge. The author must upgrade the affected package, apply a patch, or open a documented exception issue that is approved by the security-team before the merge proceeds. |
| **Configured in** | `.github/workflows/ci.yml` — job `audit` |

---

### Gate 4 — PR Title Format (Conventional Commits)

| Field | Detail |
|---|---|
| **Gate name** | PR Title / Commit Message Format |
| **What it checks** | The PR title and all commit messages must follow the Conventional Commits specification: `type(scope): description`. Valid types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`. |
| **Failure means** | The PR cannot merge. Commit messages like `fix`, `update`, `..`, or `misc` — as seen throughout this repository's history — are rejected. The author must amend commit messages (`git commit --amend` or interactive rebase) before the gate passes. |
| **Configured in** | `.github/workflows/ci.yml` — job `commit-lint` |

---

### Gate 5 — Required CODEOWNERS Review

| Field | Detail |
|---|---|
| **Gate name** | CODEOWNERS Review |
| **What it checks** | GitHub enforces that every owner listed in `.github/CODEOWNERS` for touched paths has approved the PR. This is a GitHub branch protection setting, not a workflow file. |
| **Failure means** | The PR cannot merge. The required owner must review and approve. No engineer may approve on behalf of a team they are not a member of. |
| **Configured in** | GitHub repository Settings → Branches → Branch protection rules for `main` → "Require review from Code Owners" |

---

### Non-Negotiable Rule on Bypassing Gates

**Gates are never bypassed under deadline pressure.**

If a gate is failing and a release is urgent, the correct action is to fix the gate failure — not to disable the gate, not to use admin override, not to push directly to `main`. Bypassing a gate under pressure is precisely when the risk of introducing a defect is highest. The gates exist to protect production, not to slow down the team on easy days.

The only circumstance in which a gate may be temporarily bypassed is a declared production incident (P0) that requires an immediate rollback commit. In that case, the bypass must be logged within one hour, the gate failure must be remediated in a follow-up PR within 24 hours, and the incident must be documented in the post-mortem.

---

## Release Practices

### Versioning Format

This project uses **Semantic Versioning (SemVer)**: `MAJOR.MINOR.PATCH`

| Segment | Increment when |
|---|---|
| `MAJOR` | A breaking change is introduced that requires consumer action |
| `MINOR` | A new backward-compatible feature is added |
| `PATCH` | A backward-compatible bug fix is shipped |

Examples: `1.0.0`, `1.4.0`, `1.4.3`, `2.0.0`

Hotfix releases increment the `PATCH` segment: `1.4.0` → `1.4.1`.

---

### Git Tagging Command

Tags are created on `main` after the release commit is merged and all validation gates have passed.

```bash
# Annotated tag with a descriptive message
git tag -a v1.4.0 -m "release: v1.4.0 — add OAuth login, fix cart rounding (#42, #87)"

# Push the tag to origin
git push origin v1.4.0
```

Every release tag must be:
- Annotated (use `-a`), not lightweight
- Prefixed with `v`
- Include a message naming the key changes and issue references

---

### Release Checklist

The following must be true before a release tag is created:

- [ ] All PRs intended for this release are merged into `main`
- [ ] All five validation gates are passing on `main` at the release commit
- [ ] The version number in `package.json` has been updated to match the new tag
- [ ] A CHANGELOG entry has been written describing what changed, fixed, and deprecated
- [ ] The release has been smoke-tested in the staging environment
- [ ] At least one engineer other than the release author has verified the staging deployment
- [ ] The team lead has approved the release

---

### Hotfix Release Process

A hotfix is used when a critical bug in production must be patched faster than the next regular release cycle.

```
1. Create a hotfix branch from the PRODUCTION TAG (not from main's current HEAD)
   git checkout v1.4.0
   git checkout -b hotfix/101-payment-timeout-crash

2. Apply the minimal fix. Do not bundle unrelated changes.

3. Open a PR from hotfix/101-payment-timeout-crash to main.
   - This PR must pass all five validation gates.
   - It must be reviewed by the CODEOWNERS of the affected paths.

4. After the PR merges to main, tag the hotfix release immediately:
   git checkout main && git pull
   git tag -a v1.4.1 -m "hotfix: v1.4.1 — fix payment timeout crash under high load (#101)"
   git push origin v1.4.1

5. Deploy v1.4.1 to production from the tag, not from a branch.

6. Delete the hotfix branch after the tag is pushed.
```

Hotfix commits must never be pushed directly to `main`. The branch protection rules prevent this regardless.

---

## Rollback Readiness

Two rollback methods are available, chosen based on the scope of the failure.

---

### Method 1 — `git revert` (Shared Branch Problem)

**When it applies:**  
A specific commit or small set of recent commits introduced a known regression. The commit hash(es) are identifiable. Other work has been merged to `main` since the bad commit, making a tag-based redeployment disproportionate.

**What it does:**  
Creates a new commit that reverses the changes of the target commit. History is preserved. No force push. Safe on a shared branch.

**First command the engineer runs:**
```bash
# Identify the bad commit
git log --oneline -20

# Revert it (creates a new revert commit)
git revert <commit-hash>

# Example:
git revert 6e9a3f2
```

After reverting, open a PR with the revert commit. The PR must still pass all validation gates before merging to `main`.

---

### Method 2 — Redeploy from Previous Tag (Widespread Failure)

**When it applies:**  
Multiple systems are broken, the failure is widespread, and the team cannot quickly identify a single bad commit. The safest action is to restore the entire application to a known-good tagged version.

**What it does:**  
Redeploys the production environment from a previously tagged and verified release. Does not touch the Git history of `main`. A follow-up PR is created after the incident to fix the root cause.

**First command the engineer runs:**
```bash
# List available tags to identify the last known-good release
git tag --sort=-version:refname | head -10

# Check out the tag to inspect it
git checkout v1.3.2

# Trigger deployment pipeline against this tag
# (exact command depends on deployment tooling — e.g., CI/CD release trigger)
git push origin v1.3.2
```

After redeployment stabilizes production, a post-mortem is required within 48 hours. The root cause must be resolved in a standard PR before the next release.
