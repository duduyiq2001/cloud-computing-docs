# Cloud Infrastructure (AWS + EKS)

## High-Level Overview

```mermaid
graph LR
    Users[Users] --> ALB[ALB]
    ALB --> Deploy[Deployment<br/>Rails x2]
    Deploy --> RDS[(RDS<br/>PostgreSQL)]
    Deploy --> SendGrid[SendGrid]
    DD[DaemonSet<br/>Datadog] -.-> Deploy
    Jenkins[Jenkins] --> DockerHub[DockerHub]
    Jenkins --> Deploy
```

---

## Architecture Overview

```mermaid
graph TB
    subgraph "Internet"
        Users[Users]
        GitHub[GitHub]
    end

    subgraph "AWS Cloud"
        subgraph "us-east-1"
            ALB[Application Load Balancer<br/>www.erenspace.com<br/>SSL/TLS]

            subgraph "VPC 10.0.0.0/16"
                subgraph "Public Subnets"
                    NAT[NAT Gateway]
                end

                subgraph "Private Subnets"
                    subgraph "EKS Cluster: e-ren-cluster"
                        subgraph "Node Group: t4g.small x2"
                            subgraph "App Pods"
                                Pod1[e-ren Pod 1<br/>Rails + Puma]
                                Pod2[e-ren Pod 2<br/>Rails + Puma]
                            end
                            subgraph "System Pods"
                                DD[Datadog Agent]
                                CoreDNS[CoreDNS]
                                AWSLBC[AWS LB Controller]
                            end
                        end
                    end

                    RDS[(RDS PostgreSQL<br/>db.t4g.micro<br/>20GB gp2)]
                end
            end

            ACM[ACM Certificate]
            IAM[IAM Roles<br/>- EKS Node Role<br/>- LB Controller Role<br/>- Pod Identity]
        end
    end

    subgraph "External Services"
        DockerHub[DockerHub<br/>duyiqun/eren]
        SendGridSvc[SendGrid]
        DatadogSvc[Datadog Cloud]
    end

    Users --> ALB
    ALB --> Pod1 & Pod2
    Pod1 & Pod2 --> RDS
    Pod1 & Pod2 --> NAT --> SendGridSvc
    DD --> DatadogSvc
    ACM -.->|SSL| ALB
    IAM -.->|IRSA| Pod1 & Pod2
    GitHub -.->|webhook| Jenkins
    DockerHub -.->|pull| Pod1 & Pod2

    subgraph "Jenkins EC2"
        Jenkins[Jenkins Server]
    end
    Jenkins --> DockerHub
    Jenkins -->|kubectl| Pod1
```

## AWS Services Used

| Service | Purpose | Configuration |
|---------|---------|----------------|
| **EKS** | Kubernetes cluster | 2 nodes (t4g.small), v1.29 |
| **EC2** | Node instances | 2x t4g.small (ARM) |
| **VPC** | Networking | 10.0.0.0/16, 2 AZs, NAT gateway |
| **RDS** | PostgreSQL database | db.t4g.micro, 20GB gp2, Single-AZ |
| **ALB** | Load balancer | Ingress controller managed |
| **ACM** | SSL certificate | Managed by AWS |
| **IAM** | Access control | IRSA for EKS pods |

## Estimated Monthly Cost

| Component | Specification | Cost/Month |
|-----------|---------------|-----------|
| EKS Cluster | 1.29, 2x t4g.small nodes | ~$110 |
| RDS PostgreSQL | db.t4g.micro, 20GB | ~$14 |
| VPC/NAT | Single NAT gateway, 2 AZs | ~$40 |
| ALB | AWS Load Balancer | ~$16 |
| **Total** | | **~$180** |

---

## Kubernetes Resources

```mermaid
graph TB
    subgraph "Namespace: default"
        subgraph "Workloads"
            Deploy[Deployment: e-ren<br/>replicas: 2<br/>image: duyiqun/eren:latest]
            MigJob[Job: e-ren-migrate-vN<br/>ttlSecondsAfterFinished: 600]
        end

        subgraph "Networking"
            Svc[Service: e-ren<br/>type: ClusterIP<br/>port: 80 -> 3000]
            Ing[Ingress: e-ren<br/>ALB class<br/>host: www.erenspace.com]
        end

        subgraph "Configuration"
            Secret[Secret: e-ren-secrets<br/>- DATABASE_URL<br/>- SECRET_KEY_BASE<br/>- SENDGRID]
            SA[ServiceAccount: e-ren<br/>IRSA annotation]
        end

        subgraph "Observability"
            DD[DaemonSet: datadog-agent<br/>APM + Logs + Profiling]
        end
    end

    Ing -->|routes| Svc
    Svc -->|selects| Deploy
    Deploy -->|mounts| Secret
    Deploy -->|uses| SA
    MigJob -->|mounts| Secret
    DD -->|collects from| Deploy
```

## Resource Specifications

### Deployment
- **Replicas:** 2 (fixed, no autoscaling)
- **Image:** `duyiqun/eren:latest`
- **CPU:** 250m request / 500m limit
- **Memory:** 256Mi request / 512Mi limit
- **Probes:** Liveness & readiness on `/up`

### Migration Job
- **Trigger:** `[migrate]` tag in commit message
- **Command:** `bundle exec rails db:migrate`
- **TTL:** Auto-cleanup after 10 minutes

### Secrets (Kubernetes)
- `DATABASE_URL` - RDS connection string
- `SECRET_KEY_BASE` - Rails encryption key
- `SENDGRID` - Email API key
