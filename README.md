# Cramraika/.github — fleet CI/CD reusable-workflow library

Central home for the fleet's reusable GitHub Actions workflows (`on: workflow_call`)
and composite actions. Every repo migrates to a **thin caller** that `uses:` a
reusable workflow here, so CI/CD behavior is defined ONCE and inherited fleet-wide
(root fix for the ~150 hand-rolled per-repo workflows that drifted).

> **Account note:** `Cramraika` is a GitHub **User** account (not an Org). There are
> no org-level secrets, variables, or runners — everything is per-repo. Reusable
> workflows still work from a `.github` repo on a User account; each caller repo
> must hold its own secrets and (if it needs the Mac) its own registered runner.

## Reusable workflows

| Workflow | Purpose | Key inputs / secrets |
|---|---|---|
| `ci-python.yml` | flake8 (E9/F63/F7) + pip-audit + optional docker smoke | `runner`, `python-version`, `docker-build` |
| `ci-node.yml` | tolerant npm install + build + test + npm audit | `runner`, `node-version`, `run-tests`, `docker-build` |
| `ci-docs.yml` | lychee link-check + optional docs/design tier check | `runner`, `design-check` |
| `ci-ansible.yml` | yamllint + ansible-lint + `--syntax-check` gate | `runner`, `playbook` |
| `security-scan.yml` | Trivy fs CVE (SARIF→Code Scanning) + gitleaks | `runner`, `blocking` |
| `cosign-sign.yml` | image build + Cosign sign + CycloneDX SBOM attest | `runner`; secrets `COSIGN_PRIVATE_KEY`, `COSIGN_PASSWORD` |
| `renovate.yml` | self-hosted Renovate via vagary-renovate App token | `log-level`; secrets `RENOVATE_APP_ID`, `RENOVATE_APP_PRIVATE_KEY` |

> Deploy bridges that embed host-specific substrate (e.g. the Coolify SSH bridge) are
> intentionally NOT hosted in this public repo — they have a single private consumer, so
> they live in that consumer repo (or a private reusable home if a second consumer
> appears, with infra literals parameterized as secrets).

## The two load-bearing conventions

### 1. Runner selection — resolved in the CALLER, fail-safe `ubuntu-latest`

A reusable workflow can't reliably read the caller's `vars.RUNNER_TYPE` from inside
`runs-on`. So the caller resolves it and passes it as an input:

```yaml
jobs:
  ci:
    uses: Cramraika/.github/.github/workflows/ci-node.yml@main
    with:
      runner: ${{ vars.RUNNER_TYPE || 'ubuntu-latest' }}
```

The fallback is **`ubuntu-latest`**, NOT `self-hosted`: self-hosted runners are
repo-scoped on a User account and only 5 repos have one registered (vps-ansible,
host_page, vagary-voice, vagary-platform, unified-journey-guide). `runs-on:
self-hosted` on a runner-less repo hangs forever.

### 2. Deploy bridges are HARD-PINNED — a repo var must never re-break them

A deploy job that needs the operator-Mac substrate pins `runs-on: self-hosted` with
**no runner input**. Setting `RUNNER_TYPE=ubuntu-latest` once silently broke every
deploy of such a repo (the GitHub-hosted runner has no SSH key / no reachability). CI
may follow `RUNNER_TYPE`; a substrate-bound deploy job may not. These bridges live in
their consumer repo, not here.

## Caller examples

See `examples/` (plain caller examples; the GitHub New-workflow UI does not source from a User-account .github repo) and the per-repo migration in
`platform-docs/docs/specs/2026-06-26-uniform-ci-cd.md`.

## Secrets do NOT inherit automatically

A reusable workflow sees the caller's secrets only if the caller adds
`secrets: inherit` (or passes each explicitly). The secret must EXIST in the caller
repo — there is no org fallback on a User account.

## Versioning

Callers reference `@main` (operator convention). Reusable-workflow changes land
immediately on every caller — centralization is a single point of change. To land
changes deliberately, a caller may pin a moving `@v1` tag instead.

## Renovate base-preset

`renovate.json` at the repo root is the **fleet canonical Renovate base-preset**
(ADR-033 Alternative B). Every repo carries a thin `renovate.json` that does
`"extends": ["github>Cramraika"]` and inherits the full tiered automerge policy
(patch/pin/digest + dev-minor auto-merge after CI; prod-minor + major -> PR +
`needs-review`), the security feeds (OSV + GHSA, schedule-bypass), and the
GitHub-Actions digest-pinning supply-chain rule. Per-repo configs add only their
own overrides (monorepo branch-prefixes, ansible-galaxy specialization, OSS
looser-minor). Single source kills the per-repo copy-paste drift.

`renovate.yml` (the reusable workflow) is the **executor shape only**; this
`renovate.json` is the **config**. Both coordinate via the App-token convention
(`RENOVATE_APP_ID` + `RENOVATE_APP_PRIVATE_KEY`) -- the `vagary-renovate` App token
(not the ephemeral `GITHUB_TOKEN`) is what makes automerge-after-CI and the
vulnerability-alert path actually fire.

Design: `platform-docs/docs/specs/2026-06-25-fleet-dependency-management-architecture.md`.
