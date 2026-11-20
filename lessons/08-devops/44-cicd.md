# Lesson 44. CI/CD concepts and pipelines

## Goal
Design a pipeline that takes a commit to production safely, and know the deployment strategies.

## Terms

- **Continuous Integration (CI)**: every change is merged often and automatically built and tested.
- **Continuous Delivery**: every green build *can* be released with one click.
- **Continuous Deployment**: every green build *is* released automatically.
- **Pipeline**: stages that run on each push or PR: lint → test → build → scan → publish → deploy.
- **Artifact**: what the build produces: a container image, a binary, a package. **Build once, deploy the same artifact everywhere** (dev → staging → prod). Only configuration differs.

## A typical pipeline

```
push / PR
  │
  ├─ lint        (shellcheck, hadolint, yamllint, terraform fmt, language linters)
  ├─ test        (unit, then integration with real DB in a service container)
  ├─ build       (docker build, tag with git SHA)
  ├─ scan        (trivy image, dependency audit, secret scan)
  ├─ publish     (push image to registry)            ← only on main
  ├─ deploy dev  (automatic)
  ├─ deploy stg  (automatic, smoke tests)
  └─ deploy prod (manual approval or automatic + canary)
```

Properties of a good pipeline:
- **Fast**: under ~10 minutes for feedback. Cache dependencies and Docker layers, run jobs in parallel.
- **Deterministic**: pinned tool versions, no "works on my runner".
- **Secure**: secrets in the CI secret store, short-lived cloud credentials via OIDC, least-privilege tokens, no secrets printed in logs, untrusted PRs from forks get no secrets.
- **Visible**: failures say clearly what broke.

## Example: GitHub Actions

File `.github/workflows/ci.yml` in an application repo:

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  test:
    runs-on: ubuntu-24.04
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
        options: --health-cmd "pg_isready" --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check . && pytest -q
        env: { DATABASE_URL: "postgres://postgres:test@localhost:5432/postgres" }

  image:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:sha-${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Pin third-party actions to a version (better: to a commit SHA).

## Example: GitLab CI

`.gitlab-ci.yml`:

```yaml
stages: [test, build, deploy]

test:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - pytest -q

build:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy_prod:
  stage: deploy
  script: ./deploy.sh "$CI_COMMIT_SHORT_SHA"
  environment: production
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

Jenkins, Argo Workflows, Tekton, Buildkite and others follow the same ideas.

## Deployment strategies

| Strategy | How | Pros | Cons |
|---|---|---|---|
| Recreate | stop old, start new | simple | downtime |
| Rolling | replace instances a few at a time | no downtime, default in Kubernetes | old and new run together; slow rollback |
| Blue-green | full new environment, switch traffic at once | instant rollback | double resources |
| Canary | send 1% → 10% → 50% → 100% to the new version, watching metrics | limits blast radius | needs good metrics and traffic routing |
| Feature flags | deploy code dark, enable per user/percentage | decouples deploy from release | flag debt |

Rules that make any strategy safe:
- **Backward-compatible database migrations** (expand → migrate → contract). Old and new code must both work with the schema during the rollout.
- Health checks and readiness gates before traffic.
- Automatic rollback on error-rate/latency SLO breach (lesson 57).
- Every deploy is recorded (who, what version, when) and visible on dashboards.

## Push vs pull deployment

- **Push**: the CI job connects to servers/cluster and deploys (`ssh`, `kubectl apply`, `helm upgrade`). Simple, but CI needs production credentials.
- **Pull (GitOps)**: CI only updates the desired version in a Git repo; an agent inside the cluster (Argo CD, Flux) pulls and applies it (lesson 53). No cluster credentials in CI, Git is the audit log.

## DORA metrics

How to tell if delivery is healthy:
- **Deployment frequency**: how often you deploy.
- **Lead time for changes**: commit to production.
- **Change failure rate**: % of deploys causing incidents.
- **Time to restore**: how fast you recover.

## Practice

1. Take the app from lessons 42–43 into a Git repo. Write a pipeline (GitHub Actions or GitLab CI) with lint, test and image build tagged with the commit SHA.
2. Add a `trivy` scan step that fails on CRITICAL vulnerabilities.
3. Add a deploy job that runs only on `main` and needs manual approval.
4. Make the pipeline faster with dependency and Docker layer caching. Measure before/after.
5. Write down how you would roll back a bad deploy in under 5 minutes.

## Check yourself

- Why build the artifact once instead of per environment?
- Canary vs blue-green: when do you choose which?
- Why must DB migrations be backward compatible during a rolling deploy?

## Next
[Lesson 45. Ansible](45-ansible.md)
