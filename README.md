# E-Ren Documentation

Technical documentation for E-Ren, a campus event engagement platform built with Ruby on Rails, deployed on AWS EKS.

## Quick Links

| Area | Description |
|------|-------------|
| [Backend Architecture](architecture/backend.md) | Rails app structure, models, ERD |
| [Cloud Infrastructure](architecture/cloud-infra.md) | AWS/EKS setup, Kubernetes resources |
| [CI/CD Pipeline](architecture/cicd.md) | Jenkins pipelines, GitHub Actions |
| [Local Development](development/local-setup.md) | Docker setup, e_ren CLI |

---

## Documentation Index

### Architecture

| Document | Description |
|----------|-------------|
| [backend.md](architecture/backend.md) | Rails application structure, database ERD, tech stack |
| [cloud-infra.md](architecture/cloud-infra.md) | AWS EKS cluster, VPC, RDS, Kubernetes resources |
| [cicd.md](architecture/cicd.md) | CI/CD pipeline flow, Jenkins, GitHub Actions |

### Flows

| Document | Description |
|----------|-------------|
| [registration.md](flows/registration.md) | Event registration request flow |
| [admin-deletion.md](flows/admin-deletion.md) | Async deletion with audit trail |
| [waitlist-promotion.md](flows/waitlist-promotion.md) | Automatic waitlist promotion logic |
| [authentication.md](flows/authentication.md) | Devise email confirmation flow |

### Development

| Document | Description |
|----------|-------------|
| [local-setup.md](development/local-setup.md) | Docker environment, e_ren CLI commands |
| [hext-cli.md](development/hext-cli.md) | CI/CD tooling, push/deploy commands |
| [testing.md](development/testing.md) | RSpec, mocking strategy, FactoryBot |

### Technical Notes

| Document | Description |
|----------|-------------|
| [active-jobs.md](technical-notes/active-jobs.md) | Background job architecture |
| [email-notifications.md](technical-notes/email-notifications.md) | Conditional email logic |
| [counter-cache.md](technical-notes/counter-cache.md) | Custom counter cache for confirmed registrations |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | Ruby on Rails 8.1 |
| Database | PostgreSQL (AWS RDS) |
| Container | Docker + Kubernetes (EKS) |
| CI/CD | Jenkins + GitHub Actions |
| Monitoring | Datadog APM |
| Email | SendGrid |

---

## Key Design Decisions

### Geocoding Disabled
Google Maps API costs led us to disable geocoding. Events use default WashU campus coordinates (38.6488, -90.3108). Tests use stubbed SF coordinates.

### Async Queue Adapter
We use Rails' `:async` adapter instead of Solid Queue to avoid a separate database dependency. Trade-off: jobs are lost on restart.

### Counter Cache for Confirmed Only
`EventPost.registrations_count` only counts confirmed registrations (not pending/waitlisted). Requires manual increment on waitlist promotion.

### Audit Before Async Delete
Admin deletions create an audit log synchronously before queuing the async deletion job. Ensures audit trail exists even if job fails.

### Email State Transitions
Uses Rails dirty tracking (`status_before_last_save`) to detect specific state transitions and send the correct email type.

---

## Directory Structure

```
eren_docs/
├── README.md                    # This file
├── architecture/
│   ├── backend.md              # Rails + ERD
│   ├── cloud-infra.md          # AWS/EKS + K8s
│   └── cicd.md                 # Pipelines
├── flows/
│   ├── registration.md         # Event registration
│   ├── admin-deletion.md       # Cascade deletion
│   ├── waitlist-promotion.md   # Waitlist logic
│   └── authentication.md       # Devise flow
├── development/
│   ├── local-setup.md          # Docker + CLI
│   ├── hext-cli.md             # CI/CD tool
│   └── testing.md              # RSpec + mocking
└── technical-notes/
    ├── active-jobs.md          # Background jobs
    ├── email-notifications.md  # Email logic
    └── counter-cache.md        # Counter cache
```

---

## Viewing Diagrams

All diagrams use [Mermaid](https://mermaid.js.org/) syntax. To view:

- **GitHub**: Renders automatically
- **VS Code**: Install "Markdown Preview Mermaid Support" extension
- **CLI**: Use `mmdc` (Mermaid CLI) to generate images

---

## Related Repositories

| Repo | Description |
|------|-------------|
| `e_ren` | Main Rails application |
| `e_ren_infra` | Docker, Terraform, CLI tools |
| `hext` | CI/CD tooling, Helm charts |
