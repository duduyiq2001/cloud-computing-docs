# hext CLI (CI/CD Tooling)

## Overview

`hext` is a Python CLI that wraps Docker Compose and Helm for consistent dev/CI/CD operations.

```mermaid
graph TB
    subgraph "hext CLI Commands"
        Up[hext up<br/>Start dev containers]
        Test[hext test<br/>Run RSpec in container]
        Push[hext push<br/>Build & push Docker image]
        Deploy[hext deploy<br/>Helm deploy to EKS]
    end

    subgraph "CI Pipeline (Jenkins)"
        CI1[Clone hext repo]
        CI2[Modify docker-compose<br/>for workspace paths]
        CI3[hext up]
        CI4[hext test spec/models]
        CI5[hext test spec/controllers]
        CI6[hext test spec/integration]
        CI7[hext down]
    end

    subgraph "CD Pipeline (Jenkins)"
        CD1[Configure kubectl]
        CD2[hext push]
        CD3{migrate tag?}
        CD4[hext deploy --migrate vN]
        CD5[hext deploy]
    end

    subgraph "Docker/K8s"
        DockerHub[(DockerHub<br/>duyiqun/eren)]
        EKS[EKS Cluster]
        Helm[Helm Chart]
    end

    CI1 --> CI2 --> CI3 --> CI4 & CI5 & CI6 --> CI7
    CD1 --> CD2 --> CD3
    CD3 -->|Yes| CD4
    CD3 -->|No| CD5

    Push --> DockerHub
    Deploy --> Helm --> EKS
    CD2 --> DockerHub
    CD4 & CD5 --> EKS
```

## Commands

### `hext up`
Start development containers (Rails + PostgreSQL)

```bash
hext up
# Equivalent to: docker-compose up -d
```

### `hext test [path]`
Run RSpec tests in the Rails container

```bash
hext test                    # All tests
hext test spec/models        # Models only
hext test spec/models/user_spec.rb --format documentation
```

### `hext push`
Build production Docker image and push to DockerHub

```bash
hext push
# Builds: docker build --platform linux/arm64 -t duyiqun/eren:latest .
# Pushes: docker push duyiqun/eren:latest
```

### `hext deploy [--migrate VERSION]`
Deploy to EKS using Helm

```bash
hext deploy                    # Deploy without migrations
hext deploy --migrate v42      # Deploy with migration job
```

### `hext down`
Stop and remove containers

```bash
hext down
# Equivalent to: docker-compose down
```

### `hext shell`
Open bash shell in Rails container

```bash
hext shell
# Equivalent to: docker exec -it e_ren_rails bash
```

## Technical Nuance

**Jenkins Workspace Path Fix**

In CI, the workspace structure differs from local dev:

```
Local:
~/projects/
├── e_ren/        # Rails app
└── e_ren_infra/  # docker-compose lives here
    └── hext

Jenkins:
/var/lib/jenkins/workspace/e_ren_ci/
├── (Rails app files here)
└── hext/         # Cloned in CI
```

So we `sed` the docker-compose.yml:

```bash
# CI Jenkinsfile
sed -i 's|../e_ren:/rails|..:/rails|g' hext/docker-compose.yml
sed -i 's|CONTAINER_NAME = "hext_rails"|CONTAINER_NAME = "e_ren_rails"|g' hext
```

## Source Code

Located at: `https://github.com/duduyiq2001/hext`

```
hext/
├── hext              # Python CLI script
├── setup.sh          # Installation script
├── docker-compose.yml
├── Dockerfile.dev
└── helm-charts/
    └── e-ren/        # Helm chart for EKS deployment
```
