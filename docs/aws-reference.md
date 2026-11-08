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

## ECS Task Definition

Task definition is the blueprint for the Fargate task: image, ports, CPU/memory, IAM roles, and environment (including secrets).

| Item | Value |
|---|---|
| Family | `goit-fp-ecs-task` (revisions `:1`, `:2`, `:3`, …) |
| Launch type / compatibility | Fargate |
| Container name | `Main` (essential) |
| Image | `083665737973.dkr.ecr.eu-central-1.amazonaws.com/fcc/final-project` (tag `latest` or digest) |
| Port mapping | **Container port `3000`**, protocol TCP, app protocol HTTP (not 80) |
| CPU / memory (observed) | 0.5 vCPU / 1 GiB (prefer ≤ this on Free Tier) |
| Task role | **None** |
| Task execution role | **`fp-ecs-task-execution-role`** |
| Log driver | `awslogs` → CloudWatch Logs (via execution role) |

### IAM roles on the task definition

| Field | Role | Purpose |
|---|---|---|
| **Task execution role** | `fp-ecs-task-execution-role` | Used by the **ECS agent**: pull image from ECR, write CloudWatch logs, read SSM SecureString and decrypt with KMS at task start |
| **Task role** | `None` | Used by **application code** (AWS SDK). This API talks to Postgres, Valkey, SMTP, GitHub — not AWS APIs — so no task role |
| (not on task def) | `goit-fp-ecs-role` | ECS infrastructure / Service Gateway only — do **not** select it as execution/task role |

`fp-ecs-task-execution-role` configuration:

- Trust policy principal: `ecs-tasks.amazonaws.com`
- Managed policy: `AmazonECSTaskExecutionRolePolicy` (ECR + logs)
- Inline / custom policy for secrets:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameters", "ssm:GetParameter"],
      "Resource": [
        "arn:aws:ssm:eu-central-1:083665737973:parameter/fp-ecs-database-url",
        "arn:aws:ssm:eu-central-1:083665737973:parameter/fp-ecs-smtp-pass"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:eu-central-1:083665737973:key/590c8b9a-7e8c-470c-929b-646b39173fb4"
    }
  ]
}
```

KMS key above is the account default **AWS managed `aws/ssm`** key used by SecureString parameters.

### Secrets storage (SSM Parameter Store)

Sensitive values are **not** baked into the image and **not** stored as plaintext in the task definition. They live in **Systems Manager Parameter Store** (Standard tier, type **SecureString**), encrypted with `aws/ssm`.

| Parameter name | Type | Injected as env | Value type in task def |
|---|---|---|---|
| `fp-ecs-database-url` | SecureString | `DATABASE_URL` | **ValueFrom** (SSM ARN / name) |
| `fp-ecs-smtp-pass` | SecureString | `SMTP_PASS` | **ValueFrom** |

At task start the execution role fetches and decrypts these parameters; ECS injects them into the container environment. Secrets Manager is **not** used (user choice).

### Non-secret environment variables (Value)

| Key | Typical value / notes |
|---|---|
| `NODE_ENV` | `production` |
| `PORT` | `3000` |
| `GRPC_PORT` | `50051` |
| `APP_BASE_URL` | ALB URL, e.g. `http://<alb-dns>` (**required** in production) |
| `REDIS_URL` | `redis://<valkey-primary-endpoint>:6379` |
| `SMTP_HOST` | e.g. `sandbox.smtp.mailtrap.io` |
| `SMTP_PORT` | **`587`** (STARTTLS). Avoid `465` with Mailtrap from ECS (`secure: true` path failed in practice) |
| `SMTP_USER` | SMTP username |
| `SMTP_FROM` | Valid email address (`z.string().email()`) |
| `CACHE_TTL_SECONDS` / `SCAN_INTERVAL_SECONDS` | optional; defaults in `src/config.ts` |
| `GITHUB_TOKEN` | optional; higher GitHub rate limit |

Flow:

```text
SSM SecureString  --(execution role: ssm + kms)-->  Task Definition valueFrom
Plain env Value   -------------------------------->  Container process.env
ECR image         --(execution role: ecr)--------->  Fargate task
```

## ECS Service + ALB + Auto Scaling

Long-running service that keeps Fargate tasks at the desired count, registers them with the ALB target group, and scales on CPU.

### Service

| Item | Value |
|---|---|
| Cluster | `goit-fp-ecs-cluster` |
| Service | `goit-fp-ecs` |
| Task definition | `goit-fp-ecs-task` revision **5** (Latest) |
| Scheduling | **REPLICA** |
| Desired tasks | **1** |
| AZ rebalancing | **Off** (tasks stay in `eu-central-1a` with RDS/Valkey) |
| Health check grace period | Console showed `0` — prefer **60–120 s** (Drizzle migrations on start) |
| Deployment controller | ECS |
| Deployment strategy | **Rolling update** |
| Min running tasks % | `100` |
| Max running tasks % | `200` |
| Deployment bake time | `0` min |

### Networking (service)

| Item | Value |
|---|---|
| VPC | `final-project-vpc` (`vpc-056e5484f0469d7f9`) |
| Subnet | `final-project-subnet-private1-eu-central-1a` (`subnet-0eb2d424c6822ae4e`, `10.0.128.0/20`) |
| Security group | `fp-sg-ecs` (`sg-05cbb82dccb7af6d3`) |
| Public IP | **Off** |

### Load balancing (ALB)

```text
Internet --HTTP:80--> [fp-alb / fp-sg-alb] --HTTP:3000--> [fp-tg-1] --> task Main:3000 / fp-sg-ecs
```

| Item | Value |
|---|---|
| Load balancer | `fp-alb` (Application Load Balancer) |
| Listener | **HTTP:80** |
| Target group | `fp-tg-1` |
| Container | **`Main:3000`** (ALB :80 ≠ container :3000 — correct) |
| Health check | `GET /health` on port **3000** |
| TG VPC | same as service (`vpc-056e5484f0469d7f9`) |

ALB must use **both** public subnets (`1a` + `1b`). Do not set TG or registered target port to 80.

### Security groups (service path)

| SG | Role in this path |
|---|---|
| `fp-sg-alb` | Inbound TCP 80 from `0.0.0.0/0`; outbound/target path to ECS on **3000** |
| `fp-sg-ecs` | Inbound TCP **3000** from `fp-sg-alb`; outbound to RDS `5432`, Valkey `6379`, HTTPS/SMTP via NAT |
| `fp-sg-rds` / elastic SG | Unchanged — only from `fp-sg-ecs` |

### Service Auto Scaling

| Item | Value |
|---|---|
| Enabled | Yes |
| Min tasks | **1** |
| Max tasks | **2** |
| Policy | Target tracking `fp-ecs-scale-policy` |
| Metric | `ECSServiceAverageCPUUtilization` |
| Target value | **80** |
| Scale-out cooldown | 300 s |
| Scale-in cooldown | 300 s |
| Scale-in | Enabled |

**Caveat:** release scanner runs in-process. At 2 tasks, duplicate notification emails are possible. Min 1 / max 2 satisfies the assignment Auto Scaling requirement for demo; keep desired at 1 for quiet email behavior.

### CloudWatch alarm + SNS

| Item | Value |
|---|---|
| Alarm | `fp-alarm` |
| Namespace | `AWS/ECS` |
| Dimensions | ServiceName=`goit-fp-ecs`, ClusterName=`goit-fp-ecs-cluster` |
| Metric | `CPUUtilization` Average, period 5 min |
| Threshold | `CPUUtilization > 95` for 1 datapoint within 5 minutes |
| SNS topic | `fp-aws-topiic` (spelling as created) |
| Example subscription ARN | `arn:aws:sns:eu-central-1:083665737973:fp-aws-topiic:c09afb5b-657e-4f2d-ba8a-0d46a2092093` |

Alarm notifies via SNS on high CPU; Application Auto Scaling uses the separate target-tracking policy (target 80%). Both are valid for monitoring + scaling coursework.

Public URL: `http://<fp-alb-dns-name>/` (set as `APP_BASE_URL`). Verify `GET /health`.

## App runtime notes

- Drizzle migrations run on process start (`src/index.ts`). Concurrent tasks can race; prefer desired = 1 outside load tests.
- Redis cache keys: `gh:repo-exists:*`, `gh:latest-release:*`.
- Remaining assignment pieces if missing: AWS Budgets + Cost Explorer screenshots / write-up.

## Agent do / don't

- **Do** attach resources only to `vpc-056e5484f0469d7f9`.
- **Do** source SG rules from other SGs, not `0.0.0.0/0`, for 3000/5432/6379.
- **Don't** enable RDS/ElastiCache Multi-AZ on this account.
- **Don't** put Secrets Manager in unless the user asks (they declined it).
- **Don't** open inbound SMTP or public RDS/Valkey.
- **Don't** scale ECS above 1 without warning about duplicate scanner emails.
