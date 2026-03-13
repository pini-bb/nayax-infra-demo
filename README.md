# nayax-infra-demo

Real working demo for Bluebricks infrastructure automation.
Answers Valery's 7 questions through actual code, real PR workflows, and live plan output.

---

## Questions → Coverage Map

| # | Question | Where to look |
|---|----------|---------------|
| Q1 | Full working demo with actual file examples and PR outputs | This entire repo |
| Q2 | Upgrade PR content: affected packages table, new/changed/removed inputs | `impact-summary.yml` posts a cascade table on any PR touching product-packages |
| Q3 | Plan posted as GitHub PR comment (not just pipeline log) | `bricks-deploy.yml` — bricks-action posts plan as collapsible PR comment automatically |
| Q4 | Multi-stack PR: VPC + RDS in one PR where RDS depends on VPC outputs | See [Multi-Stack PR Story](#multi-stack-pr-story) below |
| Q5 | Pull directly from public git tag, detect new upstream versions | Each product package uses `source: git::https://github.com/terraform-aws-modules/...?ref=vX.Y.Z`; `renovate.json` watches upstream tags |
| Q6 | Full inputs/outputs declared in bricks.yaml | All three product packages have complete `inputs:` and `outputs:` sections |
| Q7 | Terraform state migration | See [State Migration](#state-migration) below — out of scope for this demo |

---

## Repo Structure

```
nayax-infra-demo/
├── README.md
├── .github/
│   └── workflows/
│       ├── impact-summary.yml         # Cascade table on PRs touching product-packages/**
│       └── bricks-deploy.yml          # Plan on PR, Apply on merge for environments/**
├── product-packages/
│   ├── networking-stack/
│   │   └── bricks.yaml                # VPC + subnets (terraform-aws-modules/vpc v5.21.0)
│   ├── eks-platform/
│   │   └── bricks.yaml                # EKS + ArgoCD (terraform-aws-modules/eks v20.31.6)
│   └── rds-cluster/
│       └── bricks.yaml                # RDS Aurora (terraform-aws-modules/rds v6.10.0)
├── environments/
│   └── nayax-dev/
│       ├── 01-networking.yaml         # slug: nayax-dev-networking
│       ├── 02-eks-platform.yaml       # slug: nayax-dev-eks; refs $outs.nayax-dev-networking.*
│       └── 03-rds-cluster.yaml        # slug: nayax-dev-rds; refs $outs.nayax-dev-networking.*
└── renovate.json                      # Watches terraform-aws-modules tags, opens bump PRs
```

---

## How It Works

### Product Packages (Q5 + Q6)

Product packages in `product-packages/` are the reusable building blocks. Each one:

- References upstream Terraform modules via inline `source: git::https://...?ref=vX.Y.Z`
- Declares complete `inputs:` and `outputs:` so callers know exactly what to provide and what they get back
- Is published once to the Bluebricks platform via `bricks bp publish` (in production: CI on merge to main)

The `source:` field directly answers Q5: no intermediate wrapper, no copying module code. Bluebricks pulls the Terraform module straight from the upstream Git tag at plan/apply time.

### Environment Manifests (Q4)

Deployment manifests in `environments/nayax-dev/` are what you change in day-to-day operations. Each manifest:

- References a product package by name + version
- Supplies inputs (including cross-stack references via `$outs.*`)
- Gets deployed in order: networking first, then eks + rds (which depend on networking outputs)

### Cross-Stack References

The `$outs` syntax connects stacks:

```yaml
# 03-rds-cluster.yaml
props:
  subnet_ids: $outs.nayax-dev-networking.aws_vpc.private_subnets
  vpc_id: $outs.nayax-dev-networking.aws_vpc.vpc_id
```

Syntax: `$outs.{deployment-slug}.{package-id}.{output-key}`

**Important caveat:** `$outs` resolves from the last *deployed* state of the referenced environment. It does not resolve from a hypothetical plan output. On first deploy, you must deploy `nayax-dev-networking` before deploying `nayax-dev-eks` or `nayax-dev-rds`. The numbered prefixes (01-, 02-, 03-) encode this ordering, and the workflow steps enforce it.

---

## Multi-Stack PR Story (Q4)

Concrete scenario:

1. `nayax-dev-networking` is already deployed (VPC exists, outputs stored in Bluebricks)
2. Open a PR that changes both `01-networking.yaml` (add a subnet CIDR) and `03-rds-cluster.yaml` (bump rds-cluster version)
3. `bricks-deploy.yml` runs plan for each manifest in order:
   - **networking plan**: shows the subnet change as a Terraform diff
   - **rds plan**: resolves `$outs.nayax-dev-networking.aws_vpc.private_subnets` from the last deployed networking state, shows the RDS version bump diff
4. Both plans are posted as collapsible PR comments by bricks-action
5. PR merge triggers apply in the same ordered steps

---

## Upgrade Impact Table (Q2)

When a PR modifies any file under `product-packages/**`, the `impact-summary.yml` workflow:

1. Uses `git diff` to detect which product package directories changed
2. Scans all `environments/**/*.yaml` manifests for `blueprint:` references to the changed package
3. Posts a markdown table as a PR comment listing each affected deployment, its current pinned version, and the required action

**Honest note on input diffs:** Detailed input-level diffs (which specific inputs changed between package versions) are not yet native in Bluebricks. This table shows affected deployments and pinned versions — it tells you *where* to look, not which specific inputs changed. That gap has been filed as product feedback.

---

## Renovate (Q2 + Q5)

`renovate.json` watches `terraform-aws-modules` upstream tags. When a new tag is published (e.g. `terraform-aws-modules/terraform-aws-vpc` publishes `v6.0.0`), Renovate opens a PR updating the `?ref=` pin in the affected `bricks.yaml`. This is the auto-notify mechanism for upstream version changes.

---

## State Migration (Q7)

State migration (moving existing Terraform state into Bluebricks management) is out of scope for this demo repo. See the official guide: https://bluebricks.co/docs/help/guides/migrate-terraform-state

---

## Running Locally with `act`

```bash
# Plan job (simulates a PR)
act pull_request --secret-file .secrets

# Apply job (simulates a merge to main)
act push --secret-file .secrets
```

`.secrets` file (gitignored):
```
BRICKS_API_KEY=<your-api-key>
```

---

## Pre-Published Packages

The product packages in this repo are published once manually before running the workflows:

```bash
cd product-packages/networking-stack && bricks bp publish
cd ../eks-platform && bricks bp publish
cd ../rds-cluster && bricks bp publish
```

In production, publishing runs automatically on merge to main via a publish CI step.
