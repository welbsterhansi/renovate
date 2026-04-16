# Renovate Bot — Container Image Governance

> **Scope:** This document defines the governance model for container image updates managed by Renovate Bot across parent (`base-images`) and child (developer) repositories, with a focus on preventing image incompatibilities in OpenShift/Kubernetes environments.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Definitions](#2-definitions)
3. [Image Hierarchy](#3-image-hierarchy)
4. [Update Classification](#4-update-classification)
5. [Core Governance Rules](#5-core-governance-rules)
6. [Renovate Configuration](#6-renovate-configuration)
7. [Approval Workflow](#7-approval-workflow)
8. [Compatibility Contract](#8-compatibility-contract)
9. [Validation Gates](#9-validation-gates)
10. [Incident Response](#10-incident-response)
11. [Roles and Responsibilities](#11-roles-and-responsibilities)
12. [FAQ](#12-faq)

---

## 1. Overview

The company operates a two-tier container image model:

- **Parent images** are maintained in the `base-images` repository. They are built directly from Red Hat's official registry (`registry.redhat.io`) and pushed to the internal Azure Container Registry (ACR).
- **Child images** are maintained in individual developer repositories. Their `Dockerfile` `FROM` directive points to a parent image in ACR.

Renovate Bot automates dependency updates at both tiers. Without governance, an uncontrolled update at the parent tier can silently break dozens of child images simultaneously — or introduce a major OS generation change (e.g. UBI 8 → UBI 9) without any human decision.

This document establishes the rules, configurations, workflows, and responsibilities that prevent that from happening.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Parent image** | A base image built in the `base-images` repo, sourced from `registry.redhat.io`, published to ACR. |
| **Child image** | An application image built in a developer repo whose `FROM` references a parent image in ACR. |
| **Digest pin** | Locking an image to its exact `sha256` hash: `image:tag@sha256:abc…`. Guarantees byte-for-byte reproducibility regardless of tag mutations. |
| **Tag-only reference** | Referencing an image by tag only: `image:tag`. The underlying content can change without the tag changing. |
| **Major update** | Change in the first version segment: `ubi8 → ubi9`, `nginx-118 → nginx-126`. High risk — new OS or runtime generation. |
| **Minor update** | Change in the second version segment: `8.9 → 8.10`. Medium risk — new packages, potential behaviour changes. |
| **Patch update** | Change in the third segment or a security rebuild with the same tag: `8.9-110 → 8.9-115`. Low risk — CVE fixes, bug fixes. |
| **Dependency Dashboard** | A GitHub Issue created and maintained by Renovate listing all pending and blocked updates. Major updates require a checkbox to be ticked here before the PR is unblocked. |
| **Image drift** | The state where different environments or replicas run different image content despite referencing the same tag, due to tag mutation. |

---

## 3. Image Hierarchy

```
registry.redhat.io
        │
        │  Renovate watches here
        │  Opens PR with: image:tag@sha256:…
        │
        ▼
  ┌─────────────────────────┐
  │   base-images (GitHub)  │   ← parent images
  │   Dockerfile FROM pinned│
  │   to sha256 digest      │
  └────────────┬────────────┘
               │
               │  CI pipeline: build + docker push
               ▼
  ┌─────────────────────────┐
  │  Azure Container        │
  │  Registry (ACR)         │   ← central image store
  └────────────┬────────────┘
               │
               │  Renovate watches here
               │  Opens PR with: image:tag (no sha)
               │
               ▼
  ┌─────────────────────────┐
  │  Developer repos        │   ← child images
  │  (GitHub)               │
  │  Dockerfile FROM tag    │
  │  only, no digest        │
  └────────────┬────────────┘
               │
               │  CI pipeline: build + docker push
               ▼
  ┌─────────────────────────┐
  │  ACR                    │   ← child images published
  └────────────┬────────────┘
               │
               ▼
  OpenShift / Kubernetes workloads
```

### Why parent images use sha256 and child images use tag only

**Parent images** consume external content from Red Hat. A tag like `ubi8:8.9` can be silently rebuilt by Red Hat (e.g. a security patch is applied and the tag is moved). Without a digest pin, two builds of your parent image on different days could produce different results using the exact same `FROM` line — this is image drift. Pinning to `sha256` means Renovate will only open a PR when Red Hat publishes a new digest, making every upstream change explicit and auditable.

**Child images** reference parent images stored in ACR, which is under internal control. The tag is only moved when our own CI pipeline explicitly pushes a new build after a reviewed and merged PR. There is no silent mutation risk. Digest-pinning child images to ACR would actually break the propagation chain: after a parent image is rebuilt and pushed to ACR, child repos would never receive the update because Renovate would see the digest has changed but the tag hasn't, and skip creating a PR.

---

## 4. Update Classification

### 4.1 Parent images (source: `registry.redhat.io`)

| Update type | Example | Risk level | Default behaviour |
|---|---|---|---|
| Patch | `ubi8:8.9-110 → 8.9-115` | 🟢 Low | PR opened, no extra gate |
| Minor | `ubi8:8.9 → 8.10` | 🟡 Medium | PR opened, no extra gate |
| Major | `ubi8 → ubi9` | 🔴 High | PR opened, **blocked until Dependency Dashboard approval** |
| Major | `nginx-118 → nginx-126` | 🔴 High | PR opened, **blocked until Dependency Dashboard approval** |

### 4.2 Child images (source: ACR)

| Update type | Example | Risk level | Default behaviour |
|---|---|---|---|
| Patch / Minor | `base-img:v1.2 → v1.3` | 🟢 Low–Medium | PR opened, no extra gate |
| Major | `ubi8-base:1.x → ubi9-base:2.x` | 🔴 High | PR opened, **blocked until Dependency Dashboard approval** |

### 4.3 The two-gate rule for major updates

A major update at the parent tier creates a cascade of blocking PRs. The sequence is always:

```
1. Renovate opens a BLOCKED PR in base-images  (gate 1)
2. Platform team approves on Dependency Dashboard
3. PR is reviewed, merged, CI builds new parent image, pushes to ACR
4. Renovate detects new tag in ACR, opens BLOCKED PRs in all child repos  (gate 2)
5. Each dev team approves their own child repo PR on Dependency Dashboard
6. Each child PR is reviewed, merged, CI builds new child image
```

This means a single `ubi8 → ubi9` change requires explicit sign-off at the parent level **and** at every individual child repo. No child image runs a new OS generation silently.

---

## 5. Core Governance Rules

### Rule 1 — No automerge at any tier

No image update, regardless of type or tier, shall be automatically merged. Every PR requires a human to review and merge it. This is non-negotiable for base images.

```json
"automerge": false
```

### Rule 2 — Parent images must always be digest-pinned

Every `FROM` directive in the `base-images` repository must include the `sha256` digest. A CI validation gate must reject any PR that introduces or modifies a `FROM` line without a digest.

```dockerfile
# ✅ Correct — digest-pinned
FROM registry.redhat.io/ubi8/ubi:8.10-1132@sha256:3d6b4e8c...

# ❌ Rejected by CI gate — tag only
FROM registry.redhat.io/ubi8/ubi:8.10
```

### Rule 3 — Child images must NOT use digest pins

Child image `FROM` directives must reference ACR images by tag only. Digest-pinning at the child level would break the automated propagation chain from parent to child.

```dockerfile
# ✅ Correct — tag only
FROM myacr.azurecr.io/base/ubi8:2.1

# ❌ Incorrect — breaks propagation
FROM myacr.azurecr.io/base/ubi8:2.1@sha256:abc123...
```

### Rule 4 — Major updates require explicit approval

Any update classified as `major` must have `dependencyDashboardApproval: true`. The PR will be created but remain in a blocked state until a member of the Platform or Architecture team manually ticks the checkbox in the Renovate Dependency Dashboard issue.

### Rule 5 — PRs must always be recreated

Renovate must be configured to recreate PRs even when they are closed without merging. This ensures updates do not disappear silently from the queue.

```json
"recreateClosed": true
```

### Rule 6 — All image update PRs must carry labels

Labels are mandatory on all image update PRs to enable filtering, reporting, and alerting in GitHub. At minimum:

- `docker` — all image updates
- `redhat` — updates from `registry.redhat.io`
- `security` — patch updates (likely CVE-driven)
- `major-update` + `breaking-change` — major version bumps

### Rule 7 — Parent image tags follow a defined naming convention

Tags pushed to ACR from the `base-images` pipeline must follow semantic versioning: `<major>.<minor>.<patch>-<build>`. This enables Renovate to correctly classify updates in child repos.

```
# Examples
myacr.azurecr.io/base/ubi8:8.10.0-1
myacr.azurecr.io/base/nginx126:1.26.2-3
```

---

## 6. Renovate Configuration

### 6.1 `base-images` repo — `renovate.json`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "dependencyDashboard": true,
  "dependencyDashboardTitle": "Renovate — Base Images Dependency Dashboard",
  "docker": {
    "enabled": true,
    "pinDigests": true
  },
  "recreateClosed": true,
  "packageRules": [
    {
      "description": "Red Hat — patch and minor: open PR, no extra gate",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["registry.redhat.io/.*"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": false,
      "addLabels": ["docker", "redhat", "security"]
    },
    {
      "description": "Red Hat — major: blocked until manual approval",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["registry.redhat.io/.*"],
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "dependencyDashboardApproval": true,
      "addLabels": ["docker", "redhat", "major-update", "breaking-change"]
    }
  ]
}
```

### 6.2 Developer repos — `renovate.json`

This config should be standardised across all developer repos, ideally enforced via a shared Renovate preset hosted in a central repo.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "dependencyDashboard": true,
  "dependencyDashboardTitle": "Renovate — App Image Dependency Dashboard",
  "docker": {
    "enabled": true,
    "pinDigests": false
  },
  "recreateClosed": true,
  "packageRules": [
    {
      "description": "ACR parent images — patch and minor: open PR normally",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["myacr.azurecr.io/.*"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": false,
      "addLabels": ["docker", "base-image-update"]
    },
    {
      "description": "ACR parent images — major: blocked until manual approval",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["myacr.azurecr.io/.*"],
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "dependencyDashboardApproval": true,
      "addLabels": ["docker", "major-update", "breaking-change"]
    }
  ]
}
```

### 6.3 Shared preset (recommended)

To avoid config drift across developer repos, publish a shared preset in an internal repo (e.g. `platform/renovate-config`) and have each dev repo extend it:

```json
{
  "extends": [
    "config:base",
    "github>your-org/renovate-config//presets/acr-child-images"
  ]
}
```

This allows the Platform team to update ACR rules centrally without touching every developer repo.

---

## 7. Approval Workflow

### 7.1 Patch / Minor update

```
Renovate detects new version
          │
          ▼
  PR opened automatically
  (labelled: docker, redhat/base-image-update, security)
          │
          ▼
  Assigned team reviews PR
  - Check changelog / Red Hat errata
  - Verify CI passes
          │
          ▼
       Merge PR
          │
          ▼
  CI builds image, pushes to ACR
  (parent tier only — child PRs open next)
```

### 7.2 Major update

```
Renovate detects major version change
          │
          ▼
  PR opened — BLOCKED
  (labelled: docker, major-update, breaking-change)
  PR description includes:
  - Current version
  - Target version
  - Link to Red Hat release notes
          │
          ▼
  Platform/Arch team is notified
  (via GitHub label subscription or Slack integration)
          │
          ▼
  Compatibility assessment:
  - Is the new image supported by our OpenShift version?
  - Are all dependent RPMs/runtimes available in the new generation?
  - Which child repos consume this parent?
  - What is the rollout plan?
          │
    ┌─────┴─────┐
  Approved     Rejected
    │               │
    ▼               ▼
  Tick checkbox   Close PR
  on Dependency   Add comment explaining
  Dashboard       reason for rejection
    │
    ▼
  PR unblocked
  Team reviews and merges
    │
    ▼
  CI builds parent, pushes to ACR
    │
    ▼
  Renovate opens BLOCKED PRs in all child repos
    │
    ▼
  Each dev team repeats assessment for their app
  and approves/rejects independently
```

---

## 8. Compatibility Contract

The following contract must be respected between parent and child image maintainers.

### 8.1 What the Platform team guarantees

When publishing a new parent image to ACR, the Platform team guarantees that:

- The image has passed all internal security scans.
- The image is supported by the current production OpenShift version.
- The image tag follows the agreed naming convention (`major.minor.patch-build`).
- Release notes or a changelog entry is linked in the PR description.
- For major updates: at least 30 days notice is given to developer teams before the parent PR is merged.

### 8.2 What developer teams are responsible for

When a Renovate PR updates a child image's `FROM`:

- The team must not merge without verifying that application-level tests pass.
- The team must not close or dismiss a major-update PR without documenting the reason.
- The team is responsible for testing their own application behaviour against the new parent image.
- The team must not manually edit the `FROM` line to bypass the Renovate-managed tag.

### 8.3 Prohibited practices

| Practice | Risk | Alternative |
|---|---|---|
| Manually pinning a child `FROM` to a specific sha256 | Breaks update propagation | Use tag only; let Renovate manage updates |
| Using `latest` tag in any `FROM` | Completely unpredictable | Always use a versioned tag |
| Merging a Renovate PR without CI passing | Untested image in production | Fix CI before merging |
| Closing a major-update PR silently | Creates invisible technical debt | Document the rejection with a comment |
| Running different parent image versions across envs | Image drift, non-reproducible builds | All envs must use the same tag at the same time |

---

## 9. Validation Gates

The following automated checks must be in place to enforce the governance rules above.

### 9.1 CI gate — Parent images (base-images pipeline)

```yaml
# Example: GitHub Actions step
- name: Validate digest pin in Dockerfile
  run: |
    if grep -E "^FROM" Dockerfile | grep -qv "@sha256:"; then
      echo "ERROR: FROM line is missing sha256 digest pin."
      echo "All parent image FROM directives must include @sha256:..."
      exit 1
    fi
```

### 9.2 CI gate — Child images (developer repo pipeline)

```yaml
# Example: GitHub Actions step
- name: Validate no digest pin in Dockerfile
  run: |
    if grep -E "^FROM.*myacr\.azurecr\.io" Dockerfile | grep -q "@sha256:"; then
      echo "ERROR: Child image FROM must not include sha256 digest."
      echo "Use tag only: FROM myacr.azurecr.io/base/ubi8:2.1"
      exit 1
    fi
```

### 9.3 CI gate — Tag naming convention (base-images pipeline)

The pipeline must validate that the tag being pushed to ACR matches the required semver format before pushing.

```bash
TAG="${IMAGE_VERSION}"
if ! echo "$TAG" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+-[0-9]+$'; then
  echo "ERROR: Tag '$TAG' does not match required format: major.minor.patch-build"
  exit 1
fi
```

### 9.4 Required status checks on GitHub

The following status checks must be set as **required** on the default branch of both `base-images` and all developer repos:

- `validate-dockerfile` — runs the FROM line checks above
- `image-build` — the image must build successfully
- `image-scan` — security scan must pass with no critical CVEs
- `renovate-config-validator` — validates `renovate.json` schema on every PR

---

## 10. Incident Response

### 10.1 Incompatible image deployed to production

**Symptoms:** Application crashes, startup errors, missing libraries or RPMs, unexpected behaviour after a `FROM` update.

**Immediate response:**

1. Identify the image tag currently running: `oc get pod <pod> -o jsonpath='{.spec.containers[*].image}'`
2. Roll back the deployment to the previous image tag: `oc rollout undo deployment/<name>`
3. Open a GitHub Issue in the affected repo tagged `incident` and `image-compatibility`
4. Notify the Platform team to investigate the parent image

**Root cause investigation:**

1. Compare the two parent image digests using `skopeo inspect`
2. Identify which RPMs or libraries changed between builds
3. Determine if the child image's application dependencies are incompatible with the change
4. Document findings in the GitHub Issue

**Resolution:**

- If the parent image is at fault: revert the parent PR, rebuild and push previous version to ACR with a new patch tag.
- If the child image is at fault: update the application to be compatible, then re-merge the child PR.

### 10.2 Renovate opens hundreds of PRs simultaneously

This can happen after a long Renovate pause or a major parent image update propagating to many child repos.

**Prevention:** Set `prConcurrentLimit` and `prHourlyLimit` in the shared preset:

```json
{
  "prConcurrentLimit": 5,
  "prHourlyLimit": 2
}
```

**If it happens:** Use the Dependency Dashboard to batch-approve or batch-close PRs. Do not merge them all at once — stagger merges to allow CI capacity to absorb the load.

### 10.3 Renovate stops creating PRs

**Likely causes:**
- `renovate.json` has a syntax error — validate with `renovate-config-validator`
- GitHub App token has expired or lost permissions
- The Renovate bot has been rate-limited by the GitHub API

**Check:** Open the Dependency Dashboard issue and look for error messages. Check the Renovate bot logs in your CI runner or Renovate Cloud dashboard.

---

## 11. Roles and Responsibilities

| Role | Responsibilities |
|---|---|
| **Platform / Image team** | Maintains `base-images` repo. Reviews and merges all parent image PRs. Owns the CI gates and tag naming convention. Gives 30-day notice before merging major parent updates. |
| **Developer teams** | Reviews and merges child image PRs in their own repos. Responsible for application-level compatibility testing. Must not bypass Renovate-managed `FROM` lines. |
| **Architect / Tech Lead** | Approves major updates on the Dependency Dashboard at both parent and child tiers. Owns the compatibility assessment process. Reviews this governance document annually. |
| **Security team** | Subscribes to `security` label notifications. Verifies that patch updates for CVE fixes are merged within the agreed SLA (see below). |
| **Renovate admin** | Manages the Renovate GitHub App installation, shared presets, and bot configuration. Responds to Renovate outages or misconfigurations. |

### 11.1 SLA for merging security patch updates

| Severity | Target merge time |
|---|---|
| Critical CVE (CVSS ≥ 9.0) | Within 24 hours of PR creation |
| High CVE (CVSS 7.0–8.9) | Within 72 hours |
| Medium / Low | Within next scheduled sprint |

---

## 12. FAQ

**Q: Why does Renovate open a PR even for a patch update? Can't we automerge patches?**

A: For container base images, even a patch update can introduce breaking changes — a new glibc version, a changed default config file, a removed binary. Given that parent images underpin all application containers in the company, the cost of a bad patch reaching production outweighs the convenience of automerge. Manual review is mandatory at all levels.

---

**Q: A developer wants to pin their child `FROM` to a specific sha256 for extra stability. Why is that not allowed?**

A: Digest-pinning at the child level severs the automated update chain. Renovate would no longer be able to open PRs for that image because the digest would never change from Renovate's perspective (it doesn't know the tag moved). The stability guarantee should come from the parent image governance, not from freezing the child.

---

**Q: What happens if Red Hat pulls or deprecates an image?**

A: Renovate will not open a PR because there is no new version. However, the image will continue to build using the last known digest until it is explicitly removed from Red Hat's registry. The Platform team should monitor Red Hat's lifecycle announcements and proactively open a migration PR when an image reaches end-of-life.

---

**Q: Can a developer pin a specific parent image version and opt out of Renovate updates for their repo?**

A: No. All developer repos must have Renovate enabled and must not disable or exclude ACR image packages from the scan. Opting out creates invisible dependency debt and means security patches never reach that application. If a developer has a legitimate compatibility reason to stay on a specific parent version, they must open a GitHub Issue documenting the reason and a target date for migration.

---

**Q: How do we handle images that don't follow semver? For example, `ubi8/openjdk-21`.**

A: Renovate's `versioning: "loose"` handles non-semver tags by doing a best-effort comparison. For images with opaque tags (e.g. date-based or hash-based), the Platform team must define a custom `versioning` rule in the shared preset and document the expected tag pattern. If versioning cannot be reliably parsed, the image should be excluded from automated updates and managed manually with a documented review cycle.

---

**Q: We have a staging environment. Can we automerge there and only require manual approval in production?**

A: Environment-specific automerge is not supported natively by Renovate (it acts on the repo, not on the deployment target). The recommended pattern is to have a separate `staging` branch where patch and minor updates can be merged with lighter review, and only promote to `main` after validation. This is outside Renovate's scope and should be handled by your GitOps / promotion pipeline.

---

*Last updated: April 2026 — Platform Engineering team*
*Review cycle: Annual or after any major Renovate version upgrade*