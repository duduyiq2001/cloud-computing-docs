# CI/CD Pipeline

## Pipeline Flow

```mermaid
flowchart TD
    subgraph "Developer"
        Dev[Developer]
    end

    subgraph "GitHub"
        Main[main branch]
        Release[release branch]
        PR[Pull Request]
    end

    subgraph "GitHub Actions"
        GA[Security Scanning<br/>- Brakeman<br/>- Bundler-audit<br/>- Rubocop]
    end

    subgraph "Jenkins CI Pipeline"
        CI1[Initialize<br/>Clone hext, start containers]
        CI2[Setup Test DB<br/>db:create db:schema:load]
        subgraph "Parallel Tests"
            T1[Models Tests]
            T2[Controllers Tests]
            T3[Integration Tests]
        end
        CI3[Report to GitHub]
    end

    subgraph "Jenkins CD Pipeline"
        CD1[Initialize<br/>Configure kubectl for EKS]
        CD2[Build & Push<br/>docker build + push]
        CD3{Commit has<br/>migrate tag?}
        CD4[Run Migration Job<br/>kubectl wait]
        CD5[Deploy to EKS<br/>helm upgrade --install]
        CD6[Verify Pods<br/>kubectl get pods]
    end

    subgraph "EKS Cluster"
        MigJob[Migration Job<br/>rails db:migrate]
        Deploy[Rolling Deployment<br/>2 replicas]
    end

    Dev -->|push| Main
    Dev -->|push| Release
    Dev -->|open| PR

    Main --> GA
    PR --> GA
    Main --> CI1
    PR --> CI1

    CI1 --> CI2 --> T1 & T2 & T3 --> CI3

    Release --> CD1
    CD1 --> CD2 --> CD3
    CD3 -->|Yes| CD4 --> CD5
    CD3 -->|No| CD5
    CD5 --> CD6

    CD4 --> MigJob
    CD5 --> Deploy
```

## CI Pipeline (Jenkins)

**Triggers:** Push to `main`, PRs to `main`

### Stages

1. **Initialize**
   - Clone hext CLI repo
   - Start Rails + PostgreSQL containers
   - Create `.env` with secrets

2. **Setup Test DB**
   - `db:drop db:create db:schema:load`
   - Uses `e_ren_test` database

3. **Parallel Tests**
   - `spec/models` - Model unit tests
   - `spec/controllers` - Controller tests
   - `spec/requests`, `spec/views`, `spec/integration` - Integration tests

4. **Cleanup**
   - `hext down` - Stop containers

**Location:** `.jenkins/ci.Jenkinsfile`

---

## CD Pipeline (Jenkins)

**Triggers:** Push to `release` branch

### Stages

1. **Initialize**
   - Configure `kubectl` for EKS cluster
   - Verify Docker, AWS CLI, kubectl, Helm

2. **Build & Push**
   - `hext push` - Build and push to DockerHub
   - Image: `duyiqun/eren:latest`

3. **Check Migrations**
   - Inspect commit message for `[migrate]` tag
   - Sets `HAS_MIGRATIONS` env var

4. **Deploy**
   - If `[migrate]`: Run migration job, wait for completion
   - `hext deploy` - Helm upgrade to EKS

5. **Verify**
   - `kubectl get pods -l app=e-ren`

**Location:** `.jenkins/cd.Jenkinsfile`

---

## GitHub Actions (Security)

**Triggers:** All pushes and PRs

### Checks
- **Brakeman** - Rails security vulnerabilities
- **Bundler-audit** - Gem vulnerabilities
- **Importmap audit** - JS dependency vulnerabilities
- **Rubocop** - Code style linting

**Location:** `.github/workflows/ci.yml`

---

## Migration Strategy

### Triggering Migrations

Include `[migrate]` in your commit message:

```bash
git commit -m "Add user preferences table [migrate]"
```

### What Happens

1. CD pipeline detects `[migrate]` tag
2. Creates Kubernetes Job: `e-ren-migrate-vN`
3. Job runs: `bundle exec rails db:migrate`
4. Pipeline waits for job completion (5-min timeout)
5. Job auto-deletes after 10 minutes
6. Then deploys app pods

### Manual Migration

```bash
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/e-ren-migrate-vN --timeout=300s
```
