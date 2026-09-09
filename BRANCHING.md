# Branching Strategy

This document defines the branching model this team follows. Every developer is expected to understand and apply these rules from their first commit.

---

## 1. Branch Naming Convention

All branches must use one of the following prefixes, followed by a short, lowercase, hyphen-separated description. Ticket or issue numbers should be included where they exist.

| Prefix | Purpose | Example |
|---|---|---|
| `feature/` | New functionality or enhancement | `feature/42-user-login-oauth` |
| `bugfix/` | Fix for a non-critical bug found outside production | `bugfix/87-cart-total-rounding` |
| `hotfix/` | Emergency fix for a production incident | `hotfix/101-payment-timeout-crash` |
| `release/` | Release stabilization branch | `release/1.4.0` |
| `chore/` | Non-functional work: tooling, CI, dependencies, docs | `chore/update-eslint-config` |

**Rules:**
- Branch names must be lowercase and use hyphens, never underscores or spaces.
- Names like `johns-stuff`, `wip-final`, `new-ui`, or `testing` are not permitted and will be rejected by the team lead.
- If a ticket number exists, it must appear immediately after the prefix: `feature/42-description`.

---

## 2. Protected Branches

| Branch | Protected | Required Settings |
|---|---|---|
| `main` | ✅ Yes | Require PR with ≥ 1 approval; require all status checks to pass; block direct pushes; block force pushes; restrict deletion |
| `release/*` | ✅ Yes | Require PR with ≥ 1 approval; require all status checks to pass; block direct pushes |

**Why these settings:**

- **Require PR with ≥ 1 approval:** No code reaches `main` without a second human reviewing it. This is the minimum gate against untested or broken code.
- **Require all status checks to pass:** Lint, tests, and security scans must be green. The branch cannot be merged if any automated gate is failing, regardless of deadline pressure.
- **Block direct pushes:** The commit pattern seen in this repository's history — `fix`, `update`, `testing` pushed directly to main — is structurally prevented. Not discouraged; prevented.
- **Block force pushes:** Shared history on `main` cannot be rewritten. This preserves `git bisect` integrity and audit trail.
- **Restrict deletion:** `main` cannot be accidentally deleted by any individual contributor.

---

## 3. Branch Lifecycle

```
1. OPEN   →  Create branch from latest main
              git checkout main && git pull
              git checkout -b feature/42-description

2. BUILD  →  Develop in small, logical commits with descriptive messages
              Follow Conventional Commits: feat:, fix:, chore:, docs:

3. SYNC   →  Rebase or merge from main at least once every two days (see Section 4)

4. READY  →  Open a PR against main using the PR template
              Self-review the diff before requesting reviewers
              Assign CODEOWNERS automatically (GitHub enforces this)

5. REVIEW →  Address reviewer comments with follow-up commits
              Do not force-push after a review has started

6. MERGE  →  Squash-merge or rebase-merge into main (no bare merge commits)
              PR author merges after approvals and all checks pass

7. CLOSE  →  Delete the branch immediately after merge
              git push origin --delete feature/42-description
```

**Maximum branch age:** A feature branch must not remain open for more than **14 calendar days** from creation. If work cannot be completed in 14 days, it must be broken into smaller PRs or moved behind a feature flag. Branches older than 14 days will be flagged in the weekly team sync and the author must either close, merge, or justify extension.

---

## 4. Sync Policy

Developers must sync their branch with `main` **at minimum every two days** while the branch is active.

**Command (preferred — rebase keeps history linear):**
```bash
git fetch origin
git rebase origin/main
```

**If rebase conflicts are complex, merge is acceptable:**
```bash
git fetch origin
git merge origin/main
```

Syncing prevents large, painful merge conflicts at PR time and ensures the branch is tested against current `main` code. A branch that is 50+ commits behind `main` at PR time will be sent back to the author for syncing before review begins.
