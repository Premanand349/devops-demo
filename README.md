# DevOps Demo: CI/CD, Docker and Terraform on Azure

> **This is a demo project.** It is a portfolio sample built to show my DevOps workflow. It is not client work and not a production system.

A small Flask API taken through a complete delivery pipeline:

| Stage | What it shows | Where |
|---|---|---|
| Test | Automated tests on every push and PR | `tests/`, `ci.yml` |
| Build | Slim, non-root Docker image with a healthcheck | `Dockerfile` |
| Publish | Image pushed to GitHub Container Registry from `main` | `ci.yml` |
| Validate infra | `terraform fmt` + `validate` in CI | `infra/`, `ci.yml` |
| Deploy | Azure Container Apps via Terraform, scale to zero | `infra/` |

## Run locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
pytest -q
python app/main.py        # http://localhost:8000/health
```

## Run in Docker

```bash
docker build -t devops-demo .
docker run --rm -p 8000:8000 devops-demo
curl localhost:8000/health
```

## Pipeline (`.github/workflows/ci.yml`)

1. `test`: install deps, run pytest.
2. `terraform`: format check, init without backend, validate.
3. `image`: build the Docker image with layer caching. On `main` only, log in to GHCR and push tags `latest` and the commit SHA.

After the first push to `main`, set the package to **public** (GitHub → Packages → devops-demo → Package settings) so Azure can pull it without credentials.

## Deploy to Azure

```bash
cd infra
cp terraform.tfvars.example terraform.tfvars   # fill in subscription_id and image
az login
terraform init
terraform plan
terraform apply
```

Outputs `app_url`. Container Apps scales to zero, so an idle demo costs very little. Run `terraform destroy` when you are done.

## Design choices

- Non-root container user, pinned dependency versions, `.dockerignore` to keep the image small.
- Least-privilege workflow permissions (`contents: read`, `packages: write` only on the image job).
- Terraform split into versions, variables, main and outputs; resources tagged; no secrets in code, `*.tfvars` is git-ignored.
- Scale-to-zero (`min_replicas = 0`) to control cost.

## Possible next steps

Remote Terraform state in Azure Storage, OIDC login from GitHub Actions to Azure for automated deploys, Prometheus/Grafana monitoring, and a staging environment.
