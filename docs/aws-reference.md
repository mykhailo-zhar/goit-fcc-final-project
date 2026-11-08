# AWS infrastructure reference

Canonical context for agents working on this project's AWS deployment.
Diagram: [aws-architecture.drawio](./aws-architecture.drawio).

Do **not** recreate VPC, ElastiCache, or ECR unless the user asks. Prefer modifying existing resources in **one** VPC. RDS may be recreated to fix AZ placement (see below). Never commit secrets, passwords, or `DATABASE_URL` with credentials.

## Account and region

| Item | Value |
|---|---|
| Region | `eu-central-1` |
| Account ID | `083665737973` |
| Plan constraint | Free Tier: Single-AZ data plane, no Multi-AZ RDS/ElastiCache, no EKS |

## Architecture decisions

- Compute: **ECS on Fargate** + image in **ECR**. Not EKS (control-plane cost), not Elastic Beanstalk.
- Data: **RDS PostgreSQL 16** + **ElastiCache Valkey** (Redis-compatible, `ioredis` / `REDIS_URL`).
- Credentials: user-managed. Secrets in **SSM Parameter Store** (SecureString), not Secrets Manager. Injected into the task via `valueFrom`.
- VPC spans **2 AZs** (ALB + DB/cache subnet groups). **RDS, Valkey, and ECS tasks run in `eu-central-1a` only** to avoid Free Tier Multi-AZ cost and cross-AZ traffic.
- Scanner (`src/index.ts`) runs **inside the API process**. Keep `desiredCount = 1` unless a distributed lock is added; two tasks can send duplicate emails.
- App listens on **`PORT=3000`** (HTTP) and **`GRPC_PORT=50051`**. Dockerfile `EXPOSE 3000` and `50051`. ALB target must be **container port 3000**, not 80.
- SMTP is **egress only** (587/465). Do not open inbound SMTP on security groups.
- Production requires `APP_BASE_URL` (`src/config.ts`). Set it to the ALB URL (e.g. `http://<alb-dns>`).
- `SMTP_FROM` must be a real email (`z.string().email()`), not a bare domain.

## VPC (`final-project-vpc`)

| Item | Value |
|---|---|
| Name | `final-project-vpc` |
| VPC ID | `vpc-056e5484f0469d7f9` |
| IPv4 CIDR | `10.0.0.0/16` |
| DNS | hostnames + resolution **enabled** |
| NAT | Regional NAT `final-project-regional-nat` |
| IGW | `final-project-igw` |
| S3 endpoint | Gateway `final-project-vpce-s3` (attach to **both** private route tables) |

### Subnets

| Name | Role | AZ | CIDR (known) |
|---|---|---|---|
| `final-project-subnet-public1-eu-central-1a` | ALB, NAT | `eu-central-1a` | public |
| `final-project-subnet-private1-eu-central-1a` | ECS, RDS instance, Valkey node | `eu-central-1a` | `10.0.128.0/20` (`subnet-0eb2d424c6822ae4e`) |
| `final-project-subnet-public2-eu-central-1b` | Second ALB subnet | `eu-central-1b` | public |
| `final-project-subnet-private2-eu-central-1b` | Empty (subnet groups / optional scale-out) | `eu-central-1b` | private |

Private route tables: `final-project-rtb-private1-eu-central-1a`, `final-project-rtb-private2-eu-central-1b`. Public: `final-project-rtb-public`.

**Stale VPCs (do not use):** `vpc-01d0669d36357d0ed`, `vpc-08f69843a324f3f4c`. Mixing SG/RDS across VPCs fails with *DB instance and EC2 security group are in different VPCs*.

## Security groups

All of these **must** live in `vpc-056e5484f0469d7f9`.

Traffic:

```text
Internet --:80--> [fp-sg-alb] --:3000--> [fp-sg-ecs] --:5432--> [fp-sg-rds]
                                           |------:6379--> [fp-sg-elastic]
                                           |------:443/:587/:465--> NAT --> Internet
```

| Name | ID (if known) | Inbound | Outbound |
|---|---|---|---|
| `fp-sg-alb` | | TCP 80 from `0.0.0.0/0` (optional 443) | TCP 3000 to `fp-sg-ecs` |
| `fp-sg-ecs` | `sg-05cbb82dccb7af6d3` | TCP **3000** from `fp-sg-alb` only (optional 50051 from ALB) | TCP 5432 → `fp-sg-rds`, TCP 6379 → elastic SG, **HTTPS 443** and SMTP 587/465 to `0.0.0.0/0` (or All egress for demo) |
| `fp-sg-rds` | `sg-08d2493890dd0aa63` | TCP **5432** from **`fp-sg-ecs`** (not only a home IP) | default / none special |
| `fp-sg-elastic` / `fp-sg-elastic-service` | `sg-05631f77e2fdfb97e` | TCP **6379** from `fp-sg-ecs` (not All traffic) | default |

### Known SG mistakes

- `fp-sg-ecs` outbound **only** 5432+6379 → Fargate cannot reach ECR, SSM, CloudWatch, GitHub, SMTP. Add 443 (+ SMTP) or All egress.
- `fp-sg-rds` inbound only `91.215.57.159/32` → ECS cannot connect. **Must** allow `fp-sg-ecs` on 5432.
- ECS inbound 80 from `0.0.0.0/0` is wrong. App port is 3000; source is ALB SG.
- Do not reuse SGs from old VPCs.

## RDS PostgreSQL

Canonical placement: **same AZ as ECS and Valkey** → `eu-central-1a`.

| Item | Value |
|---|---|
| Template | Free tier |
| Engine | PostgreSQL **16.15-R1** |
| Identifier | `fp-rds-postgres-1` (current recreate; older names: `fp-rds-postgres`, `fpdatabase`) |
| Class | `db.t4g.micro` |
| Storage | gp2, 20 GiB |
| Multi-AZ | **No** (disabled on Free tier template) |
| Instance AZ | **`eu-central-1a`** (required; do not leave in `eu-central-1b`) |
| Public access | **No** |
| Master user | `postgres` (self-managed password; not Secrets Manager) |
| VPC | `final-project-vpc` (`vpc-056e5484f0469d7f9`) |
| DB subnet group | `default-vpc-056e5484f0469d7f9` (4 subnets / 2 AZs) or custom with private 1a+1b |
| Security group | **`fp-sg-rds` only** |
| Proxy / Extended Support | Off |
| CA | `rds-ca-rsa2048-g1` |
| App DB name | `release_notifications` (set Initial database name on create) |
| Port | 5432 |
| Deletion protection | Off for coursework |

`DATABASE_URL` shape (store only in SSM `fp-ecs-database-url`, never in git):

```text
postgresql://postgres:<PASSWORD>@<rds-endpoint>:5432/release_notifications?sslmode=require
```

Subnet group lists private subnets in **both** AZs; the **instance** must still be created with Availability Zone = `eu-central-1a`.

### AZ change / recreate

Single-AZ Free tier has no Multi-AZ failover path that is free. To move off a wrong AZ (e.g. instance landed in `eu-central-1b` or stuck in `moving-to-vpc`):

1. Create a **new** instance with the table above and AZ **`eu-central-1a`**.
2. After `available`, update SSM `fp-ecs-database-url` to the **new endpoint**.
3. Confirm `fp-sg-rds` inbound TCP **5432** from **`fp-sg-ecs`** (not only a home IP `/32`).
4. Force new ECS deployment so tasks reload `DATABASE_URL`.
5. Delete the old instance (optional snapshot first).

Do not start another Modify while status is `moving-to-vpc` / `modifying`.

## ElastiCache Valkey

| Item | Value |
|---|---|
| Cluster | `fp-valkey-cluster` |
| Engine | Valkey, node-based, Easy create **Demo** |
| Node | `cache.t4g.micro` |
| Subnet group | `valkey-subnet-group` |
| Placement | private `eu-central-1a` only |
| Monitoring | Standard |
| Port | 6379 |

App env (plaintext is OK if no AUTH; use `rediss://` if TLS/AUTH enabled):

```text
REDIS_URL=redis://<primary-endpoint>:6379
```

Do not enable Multi-AZ replica on Free Tier.

## ECR

| Item | Value |
|---|---|
| Repository | `fcc/final-project` (private) |
| URI | `083665737973.dkr.ecr.eu-central-1.amazonaws.com/fcc/final-project` |
| Mutability | Mutable |
| Encryption | AES-256 |
| Typical tag | `latest` (~68 MB, 2026-11-07) |

Build/push (linux/amd64 for Fargate):

```bash
aws ecr get-login-password --region eu-central-1 \
  | docker login --username AWS --password-stdin 083665737973.dkr.ecr.eu-central-1.amazonaws.com
docker build -t fcc/final-project .
docker tag fcc/final-project:latest 083665737973.dkr.ecr.eu-central-1.amazonaws.com/fcc/final-project:latest
docker push 083665737973.dkr.ecr.eu-central-1.amazonaws.com/fcc/final-project:latest
```

Task definition may pin a digest: `.../fcc/final-project@sha256:...`.

## IAM / SSM

| Role | Use |
|---|---|
| `fp-ecs-task-execution-role` | **Task execution role** (required) |
| Task role | **None** — app does not call AWS APIs |
| `goit-fp-ecs-role` | ECS infrastructure / Service Gateway — **not** the task execution role |

Execution role:

- Trust: `ecs-tasks.amazonaws.com`
- Managed: `AmazonECSTaskExecutionRolePolicy`
- Extra: `ssm:GetParameter`, `ssm:GetParameters` on:

  - `arn:aws:ssm:eu-central-1:083665737973:parameter/fp-ecs-database-url`
  - `arn:aws:ssm:eu-central-1:083665737973:parameter/fp-ecs-smtp-pass`

- Extra: `kms:Decrypt` on default SSM key:

  `arn:aws:kms:eu-central-1:083665737973:key/590c8b9a-7e8c-470c-929b-646b39173fb4`

### Parameter Store (SecureString)

| Name | Maps to env |
|---|---|
| `fp-ecs-database-url` | `DATABASE_URL` (`valueFrom`) |
| `fp-ecs-smtp-pass` | `SMTP_PASS` (`valueFrom`) |

Non-secret task env typically: `NODE_ENV=production`, `PORT=3000`, `GRPC_PORT=50051`, `SMTP_HOST`, `SMTP_PORT=587`, `SMTP_USER`, `SMTP_FROM` (valid email), `REDIS_URL`, **`APP_BASE_URL`** (required in production). Mailtrap sandbox was used (`sandbox.smtp.mailtrap.io`). Use **`SMTP_PORT=587`** (STARTTLS); `465` enables `secure: true` in `email-notifier.ts` and failed against Mailtrap from ECS.

## ECS

| Item | Value |
|---|---|
| Cluster | `goit-fp-ecs-cluster` |
| Service | `golt-fp-ecs` (typo `golt` vs `goit` is cosmetic) |
| Task definition | `golt-fp-ecs-task` revision 1+ |
| Launch | Fargate, capacity provider `FARGATE` weight 1 |
| Desired count | **1** |
| AZ rebalancing | Off |
| Networking | private subnet `subnet-0eb2d424c6822ae4e`, SG `fp-sg-ecs`, **Public IP off** |
| Container name | `Main` (essential) |
| Container port | **3000** / HTTP (not 80) |
| CPU / memory | Prefer 0.25 vCPU / 0.5–1 GB on Free Tier (1 vCPU / 3 GB is expensive) |

Health check path for ALB: `GET /health`. After attaching ALB, set health check grace period ~60–120s (migrations on startup).

ALB: at least **two public subnets** (1a + 1b). Listener 80 → target group HTTP:3000. gRPC 50051 is optional (needs HTTP/2 listener); REST is enough for the assignment public URL.

## App runtime notes

- Drizzle migrations run on process start (`src/index.ts`). Concurrent tasks can race; keep one replica.
- Redis cache keys: `gh:repo-exists:*`, `gh:latest-release:*`.
- Next assignment pieces if missing: ALB + public URL, CloudWatch logs/alarms + SNS, ECS Service Auto Scaling (max 2 for demo), AWS Budgets + Cost Explorer.

## Agent do / don't

- **Do** attach resources only to `vpc-056e5484f0469d7f9`.
- **Do** source SG rules from other SGs, not `0.0.0.0/0`, for 3000/5432/6379.
- **Don't** enable RDS/ElastiCache Multi-AZ on this account.
- **Don't** put Secrets Manager in unless the user asks (they declined it).
- **Don't** open inbound SMTP or public RDS/Valkey.
- **Don't** scale ECS above 1 without warning about duplicate scanner emails.
