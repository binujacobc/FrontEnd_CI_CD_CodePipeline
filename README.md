# FrontEnd CI/CD on AWS — CodePipeline + S3/CloudFront + Terraform

[![CI](https://github.com/binujacobc/FrontEnd_CI_CD_CodePipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/binujacobc/FrontEnd_CI_CD_CodePipeline/actions/workflows/ci.yml)

End-to-end delivery of a Vue 3 single-page application on AWS: static hosting behind CloudFront, and a fully automated CodePipeline/CodeBuild pipeline that builds and deploys on every push — all infrastructure provisioned with modular Terraform.

## Architecture

```
                       ┌────────────── CI/CD (terraform-cicd/) ─────────────┐
 GitHub push ─────▶  CodePipeline ──▶ CodeBuild (buildspec.yml)               │
                       │                  • npm ci && npm run build             │
                       │                  • aws s3 sync dist/ ──────┐          │
                       │                  • cloudfront invalidation  │          │
                       └──────────────────────────────────│──────────┘
                                                                 ▼
 Users ── HTTPS ──▶ Route 53 ──▶ CloudFront (ACM cert, OAC) ──▶ S3 (private bucket)
                       ┌───────────── Hosting (terraform/) ──────────────┘
```

## Repository layout

```
.
├── frontend/        # Vue 3 + Vite SPA (the application being deployed)
├── terraform/       # Hosting infra: S3, CloudFront (OAC), ACM, Route 53
├── terraform-cicd/  # Pipeline infra: CodePipeline, CodeBuild, IAM
└── buildspec.yml    # CodeBuild build + deploy instructions
```

Both Terraform roots are modular (`modules/`), use remote state (`backend.tf`), pin providers, and take variables via `terraform.tfvars` (see the `terraform.tfvars.example` in each).

## Deployment

**Prerequisites:** Terraform >= 1.5, AWS credentials, an existing Route 53 hosted zone for your domain, and a GitHub connection (CodeStar) for CodePipeline.

```sh
# 1. Provision hosting infrastructure
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in domain, hosted_zone_id, ...
terraform init && terraform apply

# 2. Provision the pipeline (consumes outputs from step 1)
cd ../terraform-cicd
cp terraform.tfvars.example terraform.tfvars   # fill in repo, bucket, distribution id
terraform init && terraform apply

# 3. Push to main — the pipeline builds frontend/ and deploys it
```

## Deploy details worth noting

- The S3 bucket is **private**; CloudFront reaches it via **Origin Access Control**, and the ACM certificate lives in `us-east-1` as CloudFront requires.
- `buildspec.yml` ships hashed assets with `cache-control: public, max-age=31536000, immutable`, while `index.html` is deployed with `no-cache` — so releases are instant but assets stay cached at the edge.
- Every deploy ends with a targeted **CloudFront invalidation**.
- `node_modules` is cached between CodeBuild runs.

## Related repositories

The three concerns also exist as standalone repos: [sample_web_app_vue](https://github.com/binujacobc/sample_web_app_vue) (app), [Terraform_FrontEnd_Infra](https://github.com/binujacobc/Terraform_FrontEnd_Infra) (hosting), [Terraform_FrontEnd_CI_CD](https://github.com/binujacobc/Terraform_FrontEnd_CI_CD) (pipeline). This repo is the integrated monorepo version.

## License

[MIT](LICENSE)
