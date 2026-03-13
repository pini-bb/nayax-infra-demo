# nayax-infra-demo

Working demo for Bluebricks infrastructure automation.
Answers Valery's 7 questions through actual code, real PR workflows, and live plan output.

---

## Questions -> Coverage Map

| # | Question | Where to look |
|---|----------|---------------|
| Q1 | Full working demo with actual file examples and PR outputs | This entire repo |
| Q2 | Upgrade PR content: affected packages, version bumps, cascade to environments | `impact-summary.yml` posts affected-environments table on product-package PRs; Renovate cascades version bumps to environments |
| Q3 | Plan posted as GitHub PR comment (not just pipeline log) | `bricks-deploy.yml` -- bricks-action posts plan as collapsible PR comment automatically |
| Q4 | Multi-stack PR: VPC + RDS in one PR where RDS depends on VPC outputs | See [Multi-Stack PR Story](#multi-stack-pr-story) below |
| Q5 | Pull directly from public git tag, detect new upstream versions | Each product package uses `source: git::https://github.com/terraform-aws-modules/...?ref=vX.Y.Z`; Renovate watches upstream tags |
| Q6 | Full inputs/outputs declared in bricks.yaml | All three product packages have complete `inputs:` and `outputs:` sections |
| Q7 | Terraform state migration | See [State Migration](#state-migration) below -- out of scope for this demo |

---

## Repo Structure

```
nayax-infra-demo/
├── README.md
├── .github/
│   └── workflows/
│       ├── publish.yml                # Publish changed blueprints on merge to main
│       ├── impact-summary.yml         # Post affected-environments table on product-package PRs
│       └── bricks-deploy.yml          # Plan on PR, Apply on merge for environments/**
├── product-packages/
│   ├── networking-stack/
│   │   └── bricks.yaml                # VPC + subnets (terraform-aws-modules/vpc v5.21.0)
│   ├── eks-platform/
│   │   └── bricks.yaml                # EKS + ArgoCD (terraform-aws-modules/eks v20.31.6)
│   └── rds-cluster/
│       └── bricks.yaml                # Aurora PostgreSQL (terraform-aws-modules/rds-aurora v9.3.1)
├── environments/
│   └── nayax-dev/
│       ├── 01-networking.yaml         # slug: nayax-dev-networking
│       ├── 02-eks-platform.yaml       # slug: nayax-dev-eks; refs $outs.nayax-dev-networking.*
│       └── 03-rds-cluster.yaml        # slug: nayax-dev-rds; refs $outs.nayax-dev-networking.*
└── renovate.json                      # Watches upstream tags + Bluebricks blueprint versions
```

---

## How It Works

### Product Packages (Q5 + Q6)

Product packages in `product-packages/` are the reusable building blocks. Each one:

- References upstream Terraform modules via inline `source: git::https://...?ref=vX.Y.Z`
- Declares complete `inputs:` and `outputs:` so callers know exactly what to provide and what they get back
- Is published automatically to the Bluebricks platform on merge to main via `publish.yml`

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

`$outs` resolves from the last *deployed* state of the referenced environment. It does not resolve from a hypothetical plan output. On first deploy, you must deploy `nayax-dev-networking` before deploying `nayax-dev-eks` or `nayax-dev-rds`. The numbered prefixes (01-, 02-, 03-) encode this ordering, and the workflow steps enforce it.

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

## Upgrade Cascade (Q2 + Q5)

The full upgrade loop is automated via Renovate:

**Note:** The impact table shows affected deployments and version numbers. Detailed input-level diffs (which specific inputs were added, changed, or removed) are not yet native in Bluebricks. However, Renovate PRs include the full source code diff of the changed `bricks.yaml`, so reviewers can see exactly what changed at the code level.

**Leg 1 -- Upstream module bump -> Product package PR**
- Renovate watches `terraform-aws-modules` upstream tags
- When a new tag is published (e.g. `terraform-aws-vpc` releases `v6.0.0`), Renovate opens a PR updating the `?ref=` pin in the affected product-package `bricks.yaml`
- On merge, `publish.yml` auto-publishes the new blueprint version to Bluebricks

**Leg 2 -- Blueprint version bump -> Environment PR**
- Renovate polls the Bluebricks API for new blueprint versions (custom datasource)
- When a new version is detected (e.g. `networking_stack` `1.0.5`), Renovate opens a PR bumping `version:` in the affected environment manifests
- On PR open, `bricks-deploy.yml` runs Terraform plan
- On merge, `bricks-deploy.yml` applies

---

## State Migration (Q7)

State migration (moving existing Terraform state into Bluebricks management) is out of scope for this demo repo. Refer to the [official Bluebricks state migration guide](https://bluebricks.co/docs/guides/state-migration) for the documented process.

This is a one-time operational step per environment, not a GitOps workflow artifact.
