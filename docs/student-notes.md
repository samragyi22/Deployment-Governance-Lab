# Student Audit Notes

Use this file to document your governance audit findings.

## Task 1: Audit Findings

### Weakness 1: Unrestricted Deployment Trigger (`branches: '*'`)

**Location (file + line number or GitHub setting path):**
```
.github/workflows/deploy.yml (Lines 3-6)
```

**Evidence:**
```yaml
on:
  push:
    branches:
      - '*'
```

**Risk:**
- Any push to any branch (feature branch, experimental fix, buggy draft) triggers a production deployment immediately.
- Unreviewed and unstable code can overwrite production workloads.
- Business impact: Service outages, corrupted transactional state, loss of revenue in Checkout service.

**Root Cause:**
- Developer convenience during initial prototyping to test deployments without branch constraints.

**Recommended Fix:**
- Restrict workflow triggers strictly to the production release branch (`main`) and manual dispatch:
```yaml
on:
  push:
    branches:
      - 'main'
  workflow_dispatch:
```

---

### Weakness 2: Missing Deployment Environment and Protection Gates

**Location:**
```
.github/workflows/deploy.yml (under jobs.deploy) & GitHub Settings → Environments
```

**Evidence:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    # Missing environment: production
```
In GitHub Repository Settings, no environments exist ("There are no environments for this repository").

**Risk:**
- Without `environment: production`, GitHub Actions treats the job as a plain CI task with zero gate enforcement.
- Deployments bypass any approval rules, review timers, and environment-scoped security checks.

**Root Cause:**
- Environment abstraction in GitHub Actions was omitted during initial CI/CD setup.

**Recommended Fix:**
- Add `environment: production` under `jobs.deploy`.
- Create the `production` environment in GitHub repository settings with explicit deployment protection rules.

---

### Weakness 3: Missing Required Reviewers Gate

**Location:**
```
GitHub Settings → Environments → production → Deployment protection rules
```

**Evidence:**
```
No environment defined; no Required Reviewers configured prior to audit.
```

**Risk:**
- Single-point-of-failure and zero peer-review before production code deployment.
- Unauthorized or malicious code changes can be deployed without team oversight or operational signoff.

**Root Cause:**
- Lack of deployment governance policies requiring separation of duties and manual approval.

**Recommended Fix:**
- Enable "Required reviewers" in GitHub Environment settings and configure authorized reviewers.

---

### Weakness 4: Absence of Deployment Wait Timer

**Location:**
```
GitHub Settings → Environments → production → Deployment protection rules
```

**Evidence:**
```
No wait timer configured in repository settings.
```

**Risk:**
- No cooling-off or grace window between approval and physical execution.
- If an erroneous approval is granted or an upstream regression is identified immediately after sign-off, operators have zero automatic window to abort.

**Root Cause:**
- Lack of release management protocol for staged cooldowns.

**Recommended Fix:**
- Configure a Wait Timer (1 minute for verification / 15-60 minutes in enterprise production) to enforce a mandatory delay before deployment steps start.

---

### Weakness 5: Production Secrets Exposed at Global Repository Scope

**Location:**
```
.github/workflows/deploy.yml (Lines 26-30) & GitHub Settings → Secrets and variables
```

**Evidence:**
```yaml
env:
  DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_TOKEN: ${{ secrets.API_TOKEN }}
```

**Risk:**
- Production credentials (`DEPLOY_KEY`, `DATABASE_URL`, `API_TOKEN`) are exposed repo-wide.
- Any workflow run on any pull request or development branch could access or exfiltrate sensitive production credentials.

**Root Cause:**
- Convenience of setting repo-level secrets instead of environment-bound secrets.

**Recommended Fix:**
- Scope production secrets strictly to the `production` environment in GitHub Settings → Environments → production → Environment secrets.

---

### Weakness 6: Overly Broad Workflow Permissions

**Location:**
```
.github/workflows/deploy.yml (Lines 12-14)
```

**Evidence:**
```yaml
permissions:
  contents: read
  id-token: write
```

**Risk:**
- Granting `id-token: write` when OIDC cloud token federation is not used violates the principle of least privilege.
- Increases attack surface if compromised.

**Root Cause:**
- Copy-pasted workflow boilerplate.

**Recommended Fix:**
- Restrict permissions to minimum necessary (`contents: read`).

---

## Task 2: Configuration Actions

Record what you configured:

- [x] Created production environment in GitHub
- [x] Added branch protection: only `main` can deploy
- [x] Added required reviewers: 1+ (`samragyi22`)
- [x] Added wait timer: 60 seconds (1 minute)
- [x] Created environment secrets:
  - [x] DEPLOY_KEY
  - [x] DATABASE_URL
  - [x] API_TOKEN
- [x] Updated `.github/workflows/deploy.yml` with `environment: production`

### Configuration Checklist

**GitHub Settings → Environments → production:**

- [x] Environment name is "production"
- [x] Branch protection is enabled
- [x] Only "main" branch is allowed
- [x] Required reviewers: configured
- [x] Wait timer: 60 seconds
- [x] Environment secrets are configured

**Workflow file `.github/workflows/deploy.yml`:**

- [x] Job includes `environment: production`
- [x] Deployment only triggers on `push: branches: [main]`
- [x] Secrets are referenced correctly

---

## Task 3: Validation Evidence

### Deployment Trigger

**How I triggered the deployment:**
```
Committed hardened deploy.yml targeting environment: production and pushed to origin/main.
```

### Approval Step Captured

**Screenshot or description of workflow pause:**
```
GitHub Actions paused execution at deploy job with status: 'Waiting for review'. Prompt displays 'Deployment review required' with 1 minute wait timer.
```

**Reviewers shown:**
- Reviewer: samragyi22

### Deployment Completion

**Wait timer duration:** 60 seconds

**Workflow completed successfully:**  ☑ Yes  ☐ No

**Final deployment status:**
```
Completed successfully after manual approval and 1-minute wait timer elapsed.
```

---

## Summary

### Total Weaknesses Found

- [x] 6 weaknesses documented and remediated

### Configuration Completion

- [x] 75–100% complete

### Validation Status

- [x] Workflow pauses at approval ✓
- [x] Reviewers notified ✓
- [x] Wait timer enforced ✓
- [x] Deployment proceeds after approval ✓

---

## Reflection

**What was the most critical weakness you found?**
Deploying to production on every branch push without an `environment` target or required reviewers.

**Which fix was most important to implement?**
Enforcing GitHub Environment approval gates (`environment: production` + required reviewers + branch restriction to `main`).

**What surprised you about deployment governance?**
How easy it is to leave critical credentials exposed repository-wide if environment scoping isn't deliberately applied.

**How would you apply this to a real production system?**
Implement strict multi-stage environments (`dev` -> `staging` -> `production`), enforce dual-person approval (peer review), automated smoke tests during wait timers, and OIDC-based short-lived credentials instead of long-lived secrets.
