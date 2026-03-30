# aqua-checksums

External supply chain integrity monitoring for [Aqua Security](https://github.com/aquasecurity) open source projects.

## What this repository does

This repository continuously monitors the integrity of Aqua Security's published artifacts — container images, GitHub releases, Git tags, and release assets — by recording and verifying cryptographic digests over time. For signed artifacts, it also verifies [Sigstore](https://www.sigstore.dev/) cosign signatures against the Rekor transparency log.

A scheduled GitHub Actions workflow runs every 10 minutes and compares current artifact digests against previously recorded state. If any digest has changed unexpectedly, the workflow fails and creates a GitHub Issue with details.

## Why external monitoring matters

Software supply chain attacks often target distribution channels:

- **Tag hijacking** — force-pushing existing Git tags to malicious commits
- **Release asset replacement** — re-uploading compromised binaries to existing releases
- **Image tag mutation** — pushing malicious content to existing container image tags

These attacks are difficult to detect from inside the target repository because an attacker with sufficient access can also disable in-repo monitoring. External monitoring from a separate repository is immune to this.

## Monitored targets

| Project | Releases | Tags | Container Images |
|---------|----------|------|------------------|
| [trivy](https://github.com/aquasecurity/trivy) | Yes | Yes | GHCR, ECR Public, Docker Hub |
| [trivy-action](https://github.com/aquasecurity/trivy-action) | Yes | Yes | — |
| [setup-trivy](https://github.com/aquasecurity/setup-trivy) | Yes | Yes | — |
| [trivy-operator](https://github.com/aquasecurity/trivy-operator) | Yes | Yes | GHCR, Docker Hub |
| [kube-bench](https://github.com/aquasecurity/kube-bench) | Yes | Yes | Docker Hub |
| [tfsec](https://github.com/aquasecurity/tfsec) | Yes | Yes | GHCR, Docker Hub |
| [tracee](https://github.com/aquasecurity/tracee) | Yes | Yes | Docker Hub |
| [kube-hunter](https://github.com/aquasecurity/kube-hunter) | Yes | Yes | Docker Hub |

## How it works

1. **Discovery** — list current tags and releases from GitHub API and container registries
2. **Integrity check** — compare current digests (via HEAD requests) against recorded state
3. **Signature verification** — for artifacts signed with [Sigstore](https://www.sigstore.dev/), verify [cosign](https://github.com/sigstore/cosign) keyless signatures and check inclusion in the [Rekor](https://docs.sigstore.dev/logging/overview/) transparency log. Supports both legacy tag-based signatures (`.sig` tags) and the newer Sigstore bundle format (OCI Referrers API and `.sigstore.json` release assets).
4. **State update** — commit and push any new entries to this repository

### Check modes

| Mode | Schedule | What it checks |
|------|----------|----------------|
| Quick | Every 10 min | Recent image tags, all releases/tags, no cosign |
| Full | Every 6 hours | All image tags, cosign verification, deep asset check |

## State files

State is stored as TSV files in the `state/` directory (one per target). Each row records an artifact's type, identifier, digest, signature status, and creation date. Changes to state files are committed by the workflow, providing a full audit trail via Git history.

## Alerts

When an integrity violation is detected, the workflow:
1. Exits with a non-zero status (workflow failure)
2. Creates a GitHub Issue with details of the findings
