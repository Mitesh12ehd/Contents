# Payload CMS on AWS — Assignment Demo Runbook

**Goal:** Build a basic Payload CMS admin app, store data in Amazon RDS PostgreSQL from the first `create-payload-app` command (no local database), Dockerize it, deploy it with Jenkins on EC2, put two application EC2 instances behind an Application Load Balancer, and make the app reachable on the ALB DNS name.

**Primary URL you will demonstrate**

```text
http://<ALB-DNS>/admin
```

No custom domain is required. The ALB DNS name is the assignment endpoint.

---

## How to use this document

1. Work **in order**. Later steps assume earlier resources exist.
2. Keep a **values sheet** (Section 2). Fill every row as you create resources. You will need those values in Jenkins, Docker, and the demo.
3. Prefer the **AWS Console** path in each step. CLI snippets are optional backups.
4. Use **one AWS region** for everything. Recommended: `ap-south-1` (Mumbai) if you are in India.
5. After the build, use **Section 14 (Live demo script)** as the speaking notes.

---

## 1. Requirement map (what the evaluator should see)

| # | Assignment requirement | What you build | What you show in the demo |
|---|------------------------|----------------|---------------------------|
| 1 | Basic Payload CMS + dashboard | `create-payload-app` blank template, Admin UI at `/admin` | Login, Users collection, create one record |
| 2 | RDS PostgreSQL from the start, no local DB | RDS created **before** app init; `DATABASE_URL` points at RDS | RDS console + Payload still works after EC2 restart |
| 3 | Dockerize | Multi-stage `Dockerfile`, image in Amazon ECR | `docker images` / ECR repository |
| 4 | Jenkins CI/CD on EC2 | Dedicated Jenkins EC2, `Jenkinsfile`, GitHub webhook | Push to GitHub → Jenkins green build → both app EC2s update |
| 5 | ALB | Internet-facing ALB, listener `:80`, target group of 2 instances | Open ALB DNS in browser |
| 6 | Two EC2 instances | `payload-app-1` and `payload-app-2` | ALB target group: 2 healthy targets |
| 7 | Accessible endpoint, domain not required | `http://<ALB-DNS>/admin` | Browser load of admin dashboard |
| 8 | IAM and VPC as required | Custom VPC, subnets in 2 AZs, 4 security groups, 3 IAM roles | VPC diagram + IAM roles attached to instances |

**Instance count (read this once):** Jenkins must run on EC2, and the app must run on **two** EC2 instances behind the ALB. That is **three** EC2 instances total:

| Instance | Role |
|----------|------|
| `payload-jenkins` | CI/CD server (Jenkins + Docker engine) |
| `payload-app-1` | Application node in AZ-a |
| `payload-app-2` | Application node in AZ-b |

If an evaluator insists on “only two EC2s,” they almost always mean the **two application servers**. Jenkins is a separate required host.

---

## 2. Values sheet (fill as you go)

Copy this table into Notepad and keep it open.

| Item | Value |
|------|--------|
| AWS Account ID | |
| Region | e.g. `ap-south-1` |
| Your public IP | from https://checkip.amazonaws.com |
| Key pair name | `payload-demo-key` |
| Path to `.pem` file | |
| VPC ID | `vpc-...` |
| VPC CIDR | `10.0.0.0/16` |
| Public subnet AZ-a ID | `subnet-...` (`10.0.1.0/24`) |
| Public subnet AZ-b ID | `subnet-...` (`10.0.2.0/24`) |
| Private subnet AZ-a ID | `subnet-...` (`10.0.11.0/24`) |
| Private subnet AZ-b ID | `subnet-...` (`10.0.12.0/24`) |
| IGW ID | `igw-...` |
| SG Jenkins | `sg-...` |
| SG ALB | `sg-...` |
| SG App | `sg-...` |
| SG RDS | `sg-...` |
| RDS endpoint | `payload-pg.xxxx.ap-south-1.rds.amazonaws.com` |
| RDS DB name | `payload` |
| RDS master username | `payloadadmin` |
| RDS master password | *(store in a password manager, never Git)* |
| `DATABASE_URL` | `postgresql://payloadadmin:<URL-ENCODED-PASSWORD>@<RDS-ENDPOINT>:5432/payload?sslmode=require` |
| `PAYLOAD_SECRET` | 64-hex chars from Section 6.2 |
| Jenkins public IP | |
| Jenkins URL | `http://<JENKINS-IP>:8080` |
| App-1 private IP | |
| App-2 private IP | |
| App-1 instance ID | `i-...` |
| App-2 instance ID | `i-...` |
| ECR repo URI | `<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/payload-cms-app` |
| ALB DNS | `payload-alb-xxxxx.ap-south-1.elb.amazonaws.com` |
| GitHub repo URL | `https://github.com/<you>/payload-cms-app.git` |
| First admin email | |
| First admin password | |

---

## 3. Target architecture

```text
                         Internet users
                               |
                               |  TCP 80
                               v
                    Application Load Balancer
                    (public subnets, 2 AZs)
                               |
                    Target Group :3000
                     /                   \
                    /                     \
           EC2 payload-app-1        EC2 payload-app-2
           (Docker: payload app)    (Docker: payload app)
                    \                     /
                     \                   /
                      \                 /
                       RDS PostgreSQL
                   (private subnets, 2 AZs)
                      port 5432 only from App SG
                            and your IP (during init)

GitHub  --webhook-->  EC2 payload-jenkins :8080
                         |  docker build
                         |  docker push ECR
                         |  SSH to app-1 and app-2
                         v
                      Amazon ECR
```

**Traffic rules (this is the IAM/VPC story):**

| From | To | Port | Why |
|------|----|------|-----|
| Your IP | Jenkins EC2 | 22, 8080 | SSH + Jenkins UI |
| Your IP | App EC2 | 22 | SSH / troubleshooting |
| Your IP | RDS | 5432 | Laptop init of Payload (**temporary**) |
| 0.0.0.0/0 | ALB | 80 | Public endpoint |
| ALB SG | App SG | 3000 | Load balancer to containers |
| App SG | RDS SG | 5432 | App to database |
| Jenkins SG | App SG | 22 | Jenkins deploy over SSH |
| App / Jenkins | 443 outbound | 443 | Pull images, Git, AWS APIs |

No NAT Gateway is used. That keeps student cost down. App and Jenkins instances sit in **public subnets** with public IPs so they can pull from ECR. RDS sits in **private subnets** but is marked **Publicly accessible = Yes** only so your laptop can initialize Payload against RDS. The RDS security group still allows `5432` only from your IP and the App security group — not from the whole internet.

After the app is running, you can remove your laptop IP from the RDS security group. Keep App SG → RDS SG.

---

## 4. Cost, sizing, and safety

Approximate monthly cost if you **leave it running**. For a demo, create, show, then delete (Section 15).

| Resource | Suggested size | Notes |
|----------|----------------|-------|
| Jenkins EC2 | `t3.medium` (2 vCPU, 4 GB), 30 GB gp3 | Next.js Docker builds need RAM. `t3.micro` will fail or swap. |
| App EC2 × 2 | `t3.small` (2 GB), 20 GB gp3 | `t3.micro` (1 GB) often OOMs with Payload + Next.js. |
| RDS | `db.t3.micro`, PostgreSQL 16, 20 GB gp3, Single-AZ | Enough for a demo. |
| ALB | 1 internet-facing ALB | Charged hourly + LCU. |
| ECR | 1 private repo | Tiny for this image. |
| NAT Gateway | **Do not create** | Expensive and not needed here. |
| Elastic IP | **Do not allocate extras** | ALB DNS is the endpoint. |

**Safety rules**

- Do not open RDS `5432` to `0.0.0.0/0`.
- Do not open Jenkins `8080` to `0.0.0.0/0` if you can avoid it. Restrict to your IP.
- Do not commit `.env`, `.pem`, or passwords.
- Stop or terminate resources the same day as the demo if this is a personal account.

---

## 5. Prerequisites on your laptop (Windows)

### 5.1 Accounts and software

1. AWS account with **AdministratorAccess** for your IAM user (or the root user for a student lab — still create an IAM user and use that).
2. GitHub account.
3. Install:
   - [Node.js 22 LTS](https://nodejs.org/) (Payload 3 requires Node `^22.23.4` or `>=24.22.1`)
   - Git for Windows
   - AWS CLI v2
   - (Optional) Docker Desktop — useful for a local image test. Jenkins will also build the image.
4. Confirm Node:

```powershell
node -v
npm -v
git --version
aws --version
```

`node -v` should print `v22.x` or `v24.x`.

### 5.2 AWS CLI login

```powershell
aws configure
```

Enter:

- Access key ID
- Secret access key
- Default region: `ap-south-1`
- Output: `json`

Check:

```powershell
aws sts get-caller-identity
```

Save the Account ID.

### 5.3 Your public IP

Browser: https://checkip.amazonaws.com

If your IP changes (hostel/mobile), security group SSH/Jenkins/RDS rules will break. Update them.

### 5.4 GitHub SSH or HTTPS

HTTPS with a Personal Access Token is enough. Create a token later when you push the repo.

---

## 6. Phase A — IAM (do this first)

You need **three IAM roles** plus your human IAM user.

### 6.1 IAM user for console work (if you currently use root)

1. Console → **IAM** → **Users** → **Create user**.
2. Name: `payload-admin`.
3. Enable console access.
4. Attach policy: `AdministratorAccess` (lab only).
5. Create access keys for AWS CLI (Command Line Interface).
6. Sign out of root. Sign in as this user.

### 6.2 Generate `PAYLOAD_SECRET`

PowerShell:

```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Paste into the values sheet. This is the Payload encryption/signing secret. Never commit it.

### 6.3 Role 1 — App EC2 instance profile (`payload-ec2-app-role`)

This role lets **app instances pull images from ECR** without access keys on disk.

1. IAM → **Roles** → **Create role**.
2. Trusted entity: **AWS service** → **EC2**.
3. Permissions: create an **inline policy** named `payload-ec2-app-ecr-pull`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "ECRPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:ACCOUNT_ID:repository/payload-cms-app"
    }
  ]
}
```

Replace `ACCOUNT_ID` and region. If the ECR repo does not exist yet, you can attach `AmazonEC2ContainerRegistryReadOnly` for the lab.

4. Role name: `payload-ec2-app-role`.
5. Create. You will attach this when launching app EC2s.

Optional but useful: also attach `AmazonSSMManagedInstanceCore` so you can use **Session Manager** if SSH is blocked.

### 6.4 Role 2 — Jenkins EC2 instance profile (`payload-ec2-jenkins-role`)

Jenkins builds and **pushes** images, so it needs write access to ECR.

1. Create role, trusted entity **EC2**.
2. Attach AWS managed policy: `AmazonEC2ContainerRegistryPowerUser`.
3. (Lab shortcut) Also attach `AmazonEC2ReadOnlyAccess` if you want Jenkins to describe instances.
4. Role name: `payload-ec2-jenkins-role`.

Jenkins will still SSH to app instances using a **key pair**, not this IAM role.

### 6.5 Role 3 — (optional) RDS enhanced monitoring

Skip unless the RDS wizard asks for it. Not required for the demo.

### 6.6 What to say in the demo about IAM

> App instances use an instance profile that can only pull from ECR. Jenkins uses a separate instance profile that can push. Database passwords are not stored in IAM; they live in environment files on the instances, not in Git.

---

## 7. Phase B — VPC, subnets, routing, security groups

Do **not** use the default VPC. Creating your own VPC is an assignment requirement.

### 7.1 Create the VPC

1. Console → **VPC** → **Your VPCs** → **Create VPC**.
2. Choose **VPC only** (not “VPC and more”) so you understand each object.  
   If you are short on time, **VPC and more** is acceptable — then skip 7.2–7.4 and still create security groups yourself.
3. Settings:

| Field | Value |
|-------|--------|
| Name | `payload-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 | No |
| Tenancy | Default |

4. Create. Copy VPC ID.

### 7.2 Internet Gateway

1. VPC → **Internet gateways** → **Create**.
2. Name: `payload-igw`.
3. Select it → **Actions** → **Attach to VPC** → `payload-vpc`.

### 7.3 Subnets (two AZs — ALB requires this)

Pick two AZs, for example `ap-south-1a` and `ap-south-1b`.

| Name | AZ | CIDR | Auto-assign public IPv4 |
|------|----|------|-------------------------|
| `payload-public-a` | AZ-a | `10.0.1.0/24` | **Enable** |
| `payload-public-b` | AZ-b | `10.0.2.0/24` | **Enable** |
| `payload-private-a` | AZ-a | `10.0.11.0/24` | Disable |
| `payload-private-b` | AZ-b | `10.0.12.0/24` | Disable |

Steps for each:

1. VPC → **Subnets** → **Create subnet**.
2. VPC: `payload-vpc`.
3. Fill name, AZ, CIDR.
4. After public subnets exist: select each public subnet → **Actions** → **Edit subnet settings** → enable **Auto-assign public IPv4 address**.

### 7.4 Route tables

**Public route table**

1. **Route tables** → **Create**.
2. Name: `payload-public-rt`. VPC: `payload-vpc`.
3. Select it → **Routes** → **Edit routes** → **Add route**:
   - Destination `0.0.0.0/0`
   - Target: Internet Gateway `payload-igw`
4. **Subnet associations** → associate `payload-public-a` and `payload-public-b`.

**Private route table** (local only — no NAT)

1. Create `payload-private-rt`.
2. Leave the default local `10.0.0.0/16` route.
3. Associate `payload-private-a` and `payload-private-b`.

RDS in these private subnets can still be reached from the VPC. “Publicly accessible” on RDS makes the hostname resolve to a public IP **from the internet**, but the security group still filters who can connect.

### 7.5 Security groups (create all four now)

VPC → **Security groups** → **Create**. VPC must be `payload-vpc` every time.

**SG-1 `payload-alb-sg`**

Inbound:

| Type | Port | Source |
|------|------|--------|
| HTTP | 80 | `0.0.0.0/0` |

Outbound: default allow all.

**SG-2 `payload-app-sg`**

Inbound:

| Type | Port | Source | Reason |
|------|------|--------|--------|
| Custom TCP | 3000 | `payload-alb-sg` | ALB health checks and traffic |
| SSH | 22 | Your IP `/32` | You |
| SSH | 22 | `payload-jenkins-sg` | Jenkins deploy |

You cannot select `payload-jenkins-sg` until it exists. Create Jenkins SG next, then **come back** and add the Jenkins SSH rule.

Outbound: default allow all (needed for ECR + RDS).

**SG-3 `payload-jenkins-sg`**

Inbound:

| Type | Port | Source |
|------|------|--------|
| SSH | 22 | Your IP `/32` |
| Custom TCP | 8080 | Your IP `/32` |

If GitHub webhooks cannot reach Jenkins (common on home IPs), add:

| Type | Port | Source |
|------|------|--------|
| Custom TCP | 8080 | `0.0.0.0/0` |

Prefer restricting this. GitHub publishes webhook IP ranges; for a lab, opening 8080 to the world for a few hours is common. Change the Jenkins admin password to something strong.

**SG-4 `payload-rds-sg`**

Inbound:

| Type | Port | Source |
|------|------|--------|
| PostgreSQL | 5432 | Your IP `/32` |
| PostgreSQL | 5432 | `payload-app-sg` |

Do **not** add `0.0.0.0/0`.

After Jenkins SG exists, edit `payload-app-sg` and add SSH from `payload-jenkins-sg`.

### 7.6 Key pair

1. EC2 → **Key pairs** → **Create key pair**.
2. Name: `payload-demo-key`.
3. Type: RSA. Format: `.pem`.
4. Download. Move it to a folder you will not lose, e.g. `C:\Users\miteshch\Documents\Personal\payloadcms\payload-demo-key.pem`.
5. Restrict permissions (PowerShell):

```powershell
icacls "C:\Users\miteshch\Documents\Personal\payloadcms\payload-demo-key.pem" /inheritance:r
icacls "C:\Users\miteshch\Documents\Personal\payloadcms\payload-demo-key.pem" /grant:r "$($env:USERNAME):(R)"
```

You will use this same key for Jenkins and both app instances so Jenkins can SSH with one credential.

---

## 8. Phase C — RDS PostgreSQL (before any Payload command)

This is the assignment’s special constraint: **no local Postgres**. RDS must exist first.

### 8.1 Subnet group

1. Console → **RDS** → **Subnet groups** → **Create DB subnet group**.
2. Name: `payload-rds-subnets`.
3. VPC: `payload-vpc`.
4. Add **both private subnets** (`payload-private-a`, `payload-private-b`).
5. Create.

A subnet group must span **two AZs** even for Single-AZ RDS.

### 8.2 Create the database

1. RDS → **Databases** → **Create database**.
2. **Standard create**.
3. Engine: **PostgreSQL**.
4. Version: **PostgreSQL 16** (any 16.x is fine).
5. Templates: **Free tier** if shown, otherwise **Dev/Test**.
6. Settings:

| Field | Value |
|-------|--------|
| DB instance identifier | `payload-pg` |
| Master username | `payloadadmin` |
| Master password | Strong password. Avoid `@ # : / % ?` if you can — they break URI encoding. Use letters, numbers, and `_`. |
| Instance class | `db.t3.micro` |
| Storage | gp3, 20 GiB, no autoscaling needed |
| Multi-AZ | No (Single-AZ) |
| VPC | `payload-vpc` |
| DB subnet group | `payload-rds-subnets` |
| Public access | **Yes** (so your laptop can init Payload) |
| VPC security group | Choose existing → `payload-rds-sg` |
| Availability Zone | No preference |
| Database port | 5432 |
| Initial database name | `payload` |
| Backup retention | 1 day (or 0 if the console allows, to save cost) |
| Encryption | Enable (default) |
| Performance Insights | Off |
| Enhanced monitoring | Off |
| Deletion protection | Off (so you can clean up) |

7. Create. Wait 5–10 minutes until **Available**.
8. Open the instance → copy **Endpoint**. Example:

```text
payload-pg.abcd1234.ap-south-1.rds.amazonaws.com
```

### 8.3 Build `DATABASE_URL`

If the password is `MyPass_123`:

```text
postgresql://payloadadmin:MyPass_123@payload-pg.abcd1234.ap-south-1.rds.amazonaws.com:5432/payload?sslmode=require
```

`sslmode=require` is important. RDS uses SSL.

If the password contains special characters, URL-encode them:

| Char | Encoded |
|------|---------|
| `@` | `%40` |
| `#` | `%23` |
| `%` | `%25` |
| `/` | `%2F` |
| `:` | `%3A` |
| `?` | `%3F` |
| space | `%20` |

### 8.4 Test RDS from the laptop (do not skip)

Install a client. Easiest: [pgAdmin](https://www.pgadmin.org/) or `psql`.

If you have Docker Desktop:

```powershell
docker run --rm -it postgres:16 psql "postgresql://payloadadmin:MyPass_123@payload-pg.abcd1234.ap-south-1.rds.amazonaws.com:5432/payload?sslmode=require"
```

Inside `psql`:

```sql
SELECT version();
\dt
\q
```

If this fails:

- RDS not Available yet
- Security group does not include your **current** public IP
- Public access is No
- Password / URL encoding
- Corporate network blocking outbound 5432 (use mobile hotspot to confirm)

**Do not continue until this connection works.** Payload init will fail the same way.

---

## 9. Phase D — Create the Payload CMS app against RDS

### 9.1 Scaffold

In PowerShell:

```powershell
cd C:\Users\miteshch\Documents\Personal\payloadcms
npx create-payload-app@latest
```

Answer prompts:

| Prompt | Answer |
|--------|--------|
| Project name | `payload-cms-app` |
| Template | **blank** |
| Database | **PostgreSQL** |
| Connection string | paste your full `DATABASE_URL` from Section 8.3 |
| Package manager | **pnpm** (recommended) or npm |

Non-interactive equivalent:

```powershell
npx create-payload-app@latest payload-cms-app -t blank --db postgres --db-connection-string "postgresql://payloadadmin:MyPass_123@payload-pg.xxxx.ap-south-1.rds.amazonaws.com:5432/payload?sslmode=require" --use-pnpm
```

This writes the connection string into `.env`. Confirm `.env` contains `DATABASE_URL` (some templates historically used `DATABASE_URI` — if both exist, keep them identical).

### 9.2 Confirm `.env`

`payload-cms-app/.env` should contain at least:

```env
DATABASE_URL=postgresql://payloadadmin:MyPass_123@payload-pg.xxxx.ap-south-1.rds.amazonaws.com:5432/payload?sslmode=require
PAYLOAD_SECRET=paste-the-64-hex-secret
```

If `create-payload-app` generated a different `PAYLOAD_SECRET`, you may keep that one — copy it into the values sheet.

**Confirm there is no local Postgres container, no SQLite file being used as the primary DB, and no `localhost:5432` in `.env`.**

### 9.3 Confirm the Postgres adapter

Open `src/payload.config.ts` (path may be `payload.config.ts` depending on template). You should see:

```ts
import { postgresAdapter } from '@payloadcms/db-postgres'

export default buildConfig({
  db: postgresAdapter({
    pool: {
      connectionString: process.env.DATABASE_URL,
    },
  }),
  // ...
})
```

If the file reads `DATABASE_URI`, change it to `DATABASE_URL` **or** add `DATABASE_URI` to `.env` with the same value. They must match.

### 9.4 Run locally against RDS

```powershell
cd C:\Users\miteshch\Documents\Personal\payloadcms\payload-cms-app
pnpm dev
```

Wait until the terminal says it is ready (usually `http://localhost:3000`).

Browser: http://localhost:3000/admin

First visit: **create the first admin user**. Save that email/password in the values sheet.

You should see the Payload dashboard (Users, and whatever the blank template includes).

In another window, connect with `psql` / pgAdmin and run `\dt`. You should now see Payload tables (`users`, `payload_migrations`, etc.). That is proof there is **no local DB**.

Stop the dev server with `Ctrl+C`.

### 9.5 GitHub repository

In `payload-cms-app`:

1. Confirm `.gitignore` includes `.env`. If not, add:

```gitignore
.env
.env.*
!.env.example
```

2. Create `.env.example` (no secrets):

```env
DATABASE_URL=postgresql://payloadadmin:CHANGE_ME@payload-pg.xxxx.region.rds.amazonaws.com:5432/payload?sslmode=require
PAYLOAD_SECRET=change-me-to-a-long-random-string
```

3. Create an empty GitHub repo named `payload-cms-app` (no README).
4. Push:

```powershell
cd C:\Users\miteshch\Documents\Personal\payloadcms\payload-cms-app
git add .
git status
git commit -m "Initial Payload CMS app using RDS PostgreSQL"
git branch -M main
git remote add origin https://github.com/<your-user>/payload-cms-app.git
git push -u origin main
```

---

## 10. Phase E — Dockerize (no local database service)

The container talks to RDS. `docker-compose` must **not** start Postgres.

### 10.1 Enable Next.js standalone output

In `next.config.mjs` or `next.config.js`, set:

```js
const nextConfig = {
  output: 'standalone',
}

export default nextConfig
```

If the file already exports a config object, add `output: 'standalone'` to it. Do not replace Payload-specific webpack/headers config if present.

### 10.2 `.dockerignore`

Create `payload-cms-app/.dockerignore`:

```text
node_modules
.next
.git
.gitignore
.env
.env.*
!.env.example
Dockerfile
docker-compose.yml
README.md
**/*.md
.vscode
.idea
coverage
```

`.env` is ignored on purpose. Secrets are injected at **run** time, not baked into the image. The **build** still needs dummy or real env vars because Payload/Next may read them during `next build`.

### 10.3 `Dockerfile`

Create `payload-cms-app/Dockerfile`. This matches Payload’s documented production image (Node 22 Alpine, standalone output):

```dockerfile
FROM node:22.23.4-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG DATABASE_URL
ARG PAYLOAD_SECRET
ENV DATABASE_URL=$DATABASE_URL
ENV PAYLOAD_SECRET=$PAYLOAD_SECRET
ENV NEXT_TELEMETRY_DISABLED=1

RUN \
  if [ -f yarn.lock ]; then yarn run build; \
  elif [ -f package-lock.json ]; then npm run build; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm run build; \
  else echo "Lockfile not found." && exit 1; \
  fi

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME=0.0.0.0

RUN addgroup --system --gid 1001 nodejs \
  && adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
RUN mkdir .next && chown nextjs:nodejs .next
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

`DATABASE_URL` / `PAYLOAD_SECRET` as `ARG` exist because `next build` for Payload often needs them. They do **not** need to stay in the final image as baked secrets if you also pass `-e` at runtime (recommended). Runtime `-e` wins.

### 10.4 Production compose file (app only)

Create `payload-cms-app/docker-compose.yml`:

```yaml
services:
  payload:
    image: ${IMAGE:-payload-cms-app:local}
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: ${DATABASE_URL}
      PAYLOAD_SECRET: ${PAYLOAD_SECRET}
    restart: unless-stopped
```

There is **no** `postgres` service.

### 10.5 Optional local Docker test (still using RDS)

If Docker Desktop is installed:

```powershell
cd C:\Users\miteshch\Documents\Personal\payloadcms\payload-cms-app
docker build --build-arg DATABASE_URL="$env:DATABASE_URL" --build-arg PAYLOAD_SECRET="$env:PAYLOAD_SECRET" -t payload-cms-app:local .
```

Easier: copy values inline. Then:

```powershell
docker run --rm -p 3000:3000 `
  -e DATABASE_URL="postgresql://..." `
  -e PAYLOAD_SECRET="..." `
  payload-cms-app:local
```

Open http://localhost:3000/admin and log in with the same admin user created in 9.4. Same RDS, same users.

If the build fails with a database error, your laptop cannot reach RDS (security group / SSL / URL). Fix that before Jenkins.

### 10.6 Commit Docker files

```powershell
git add Dockerfile .dockerignore docker-compose.yml next.config.mjs
git commit -m "Add production Docker image using RDS, no local database"
git push
```

Do not add `.env`.

---

## 11. Phase F — Amazon ECR

1. Console → **ECR** → **Repositories** → **Create**.
2. Visibility: **Private**.
3. Name: `payload-cms-app`.
4. Tag immutability: Disabled (easier for `:latest` in a lab).
5. Scan on push: optional.
6. Create. Copy the URI:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/payload-cms-app
```

You do not need to push from the laptop. Jenkins will push.

---

## 12. Phase G — Two application EC2 instances

### 12.1 User data (same on both)

Amazon Linux 2023 user data:

```bash
#!/bin/bash
set -euxo pipefail
dnf update -y
dnf install -y docker git
systemctl enable docker
systemctl start docker
usermod -aG docker ec2-user

# Wait until the instance profile can talk to AWS
dnf install -y aws-cli

cat >/home/ec2-user/app.env <<'EOF'
DATABASE_URL=postgresql://payloadadmin:MyPass_123@RDS_ENDPOINT:5432/payload?sslmode=require
PAYLOAD_SECRET=REPLACE_WITH_SECRET
EOF
chown ec2-user:ec2-user /home/ec2-user/app.env
chmod 600 /home/ec2-user/app.env
```

Replace the two secrets before launch. This file is only on the instance, not in Git.

### 12.2 Launch `payload-app-1`

EC2 → **Launch instance**.

| Field | Value |
|-------|--------|
| Name | `payload-app-1` |
| AMI | Amazon Linux 2023 |
| Instance type | `t3.small` |
| Key pair | `payload-demo-key` |
| VPC | `payload-vpc` |
| Subnet | `payload-public-a` |
| Auto-assign public IP | Enable |
| Security group | `payload-app-sg` (do not use a new default SG) |
| Storage | 20 GiB gp3 |
| IAM instance profile | `payload-ec2-app-role` |
| User data | paste from 12.1 |

Launch.

### 12.3 Launch `payload-app-2`

Same as 12.2 except:

- Name: `payload-app-2`
- Subnet: `payload-public-b`

Different AZ is required so the ALB is actually multi-AZ.

### 12.4 Wait and verify SSH

From PowerShell (use each instance’s **public** IPv4):

```powershell
ssh -i "C:\Users\miteshch\Documents\Personal\payloadcms\payload-demo-key.pem" ec2-user@APP1_PUBLIC_IP
```

First connection: type `yes`. Then:

```bash
sudo docker ps
ls -l /home/ec2-user/app.env
aws sts get-caller-identity
exit
```

Repeat for app-2.

If IAM identity fails, the instance profile is missing — **Actions → Security → Modify IAM role**.

Do **not** run the app container yet. There is no image in ECR until Jenkins builds it. You can do a one-time manual deploy later if you want a smoke test before Jenkins.

---

## 13. Phase H — Application Load Balancer

ALB needs two public subnets in different AZs. You already have them.

### 13.1 Target group

1. EC2 → **Target Groups** → **Create**.
2. Type: **Instances**.
3. Name: `payload-app-tg`.
4. Protocol: HTTP. Port: **3000**.
5. VPC: `payload-vpc`.
6. Health check protocol: HTTP.
7. Health check path: `/admin`.
8. Advanced:
   - Healthy threshold: 2
   - Unhealthy threshold: 3
   - Timeout: 5
   - Interval: 15
   - Success codes: `200-399` (Payload may redirect)
9. Register `payload-app-1` and `payload-app-2` on port 3000.
10. Create.

Targets will be **unhealthy** until containers listen on 3000. That is expected.

### 13.2 Load balancer

1. EC2 → **Load Balancers** → **Create** → **Application Load Balancer**.
2. Name: `payload-alb`.
3. Scheme: **Internet-facing**.
4. IP address type: IPv4.
5. VPC: `payload-vpc`.
6. Mappings: select **both** public subnets (`payload-public-a`, `payload-public-b`).
7. Security group: `payload-alb-sg` only. Remove the default SG if the wizard attached it.
8. Listener: HTTP `:80` → forward to `payload-app-tg`.
9. Create.

Copy **DNS name** into the values sheet. This is the assignment endpoint:

```text
http://payload-alb-0123456789.ap-south-1.elb.amazonaws.com/admin
```

It will fail until Jenkins deploys containers. Keep the tab ready for the demo.

---

## 14. Phase I — Jenkins on EC2

### 14.1 Launch Jenkins instance

| Field | Value |
|-------|--------|
| Name | `payload-jenkins` |
| AMI | Amazon Linux 2023 |
| Instance type | `t3.medium` |
| Key pair | `payload-demo-key` |
| VPC | `payload-vpc` |
| Subnet | `payload-public-a` |
| Public IP | Enable |
| Security group | `payload-jenkins-sg` |
| Storage | **30 GiB gp3** (Docker images + Jenkins home) |
| IAM instance profile | `payload-ec2-jenkins-role` |

User data:

```bash
#!/bin/bash
set -euxo pipefail
dnf update -y
dnf install -y java-21-amazon-corretto docker git
systemctl enable docker
systemctl start docker
usermod -aG docker ec2-user

# Jenkins
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
dnf install -y jenkins
usermod -aG docker jenkins
systemctl enable jenkins
systemctl start jenkins
```

If `dnf install jenkins` fails because Corretto vs Jenkins Java requirements changed, install the Java version Jenkins asks for in the error, then `systemctl restart jenkins`.

### 14.2 Open Jenkins

1. Wait 2–3 minutes.
2. Browser: `http://JENKINS_PUBLIC_IP:8080`
3. SSH in and get the initial password:

```powershell
ssh -i "C:\Users\miteshch\Documents\Personal\payloadcms\payload-demo-key.pem" ec2-user@JENKINS_PUBLIC_IP
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

4. Install **suggested plugins**.
5. Create admin user (`jenkins-admin` + strong password). Save it.
6. Jenkins URL: keep `http://JENKINS_PUBLIC_IP:8080`.

### 14.3 Extra plugins

**Manage Jenkins** → **Plugins** → **Available** → install:

- Pipeline
- Git
- GitHub
- SSH Agent
- Docker Pipeline (optional)
- Credentials Binding
- Pipeline: AWS Steps (optional)

Restart when Jenkins asks.

### 14.4 Tools on the Jenkins host

SSH as `ec2-user`:

```bash
# AWS CLI is usually present on Amazon Linux 2023
aws --version
docker --version
git --version

# Confirm jenkins can use docker
sudo -u jenkins docker ps
```

If `docker ps` fails for `jenkins`, log out/in is not enough — you already added the user to `docker` group; reboot:

```bash
sudo reboot
```

Wait and reopen Jenkins.

### 14.5 SSH key for deploys

Jenkins must SSH to app-1 and app-2 with `payload-demo-key.pem`.

**Never paste the private key into the Jenkinsfile.** Store it as a Jenkins credential.

1. Jenkins → **Manage Jenkins** → **Credentials** → **System** → **Global** → **Add credentials**.
2. Kind: **SSH Username with private key**.
3. ID: `ec2-ssh-key` (exact — the Jenkinsfile uses this).
4. Username: `ec2-user`.
5. Private key: **Enter directly** → paste the full `.pem` contents including `BEGIN` / `END` lines.
6. Save.

Second credential, kind **Secret text**:

| ID | Secret |
|----|--------|
| `database-url` | full `DATABASE_URL` |
| `payload-secret` | `PAYLOAD_SECRET` |

### 14.6 GitHub credentials (if the repo is private)

Add a GitHub Personal Access Token as username/password credential ID `github-pat`. Public repo: skip.

### 14.7 Confirm Jenkins can reach the apps

On the Jenkins instance:

```bash
ssh -i /home/ec2-user/payload-demo-key.pem -o StrictHostKeyChecking=no ec2-user@APP1_PRIVATE_IP hostname
```

You may not have the pem on Jenkins yet — that is OK. The credential injects it during the pipeline. To test networking:

```bash
nc -vz APP1_PRIVATE_IP 22
nc -vz APP2_PRIVATE_IP 22
```

If this fails, `payload-app-sg` is missing SSH from `payload-jenkins-sg`.

Use **private IPs** in the Jenkinsfile. Both instances are in the same VPC; traffic stays inside AWS and does not depend on app public IPs.

---

## 15. Phase J — Jenkinsfile and first pipeline

### 15.1 Add `Jenkinsfile` to the repo

Create `payload-cms-app/Jenkinsfile`. Replace the three placeholders at the top.

```groovy
pipeline {
  agent any

  environment {
    AWS_REGION   = 'ap-south-1'
    AWS_ACCOUNT  = '123456789012'
    ECR_REPO     = 'payload-cms-app'
    ECR_URI      = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    APP1_HOST    = '10.0.1.xx'
    APP2_HOST    = '10.0.2.xx'
    SSH_USER     = 'ec2-user'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build image') {
      steps {
        withCredentials([
          string(credentialsId: 'database-url', variable: 'DATABASE_URL'),
          string(credentialsId: 'payload-secret', variable: 'PAYLOAD_SECRET')
        ]) {
          sh '''
            docker build \
              --build-arg DATABASE_URL="$DATABASE_URL" \
              --build-arg PAYLOAD_SECRET="$PAYLOAD_SECRET" \
              -t ${ECR_URI}:${IMAGE_TAG} \
              -t ${ECR_URI}:latest \
              .
          '''
        }
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${ECR_URI}
          docker push ${ECR_URI}:${IMAGE_TAG}
          docker push ${ECR_URI}:latest
        '''
      }
    }

    stage('Deploy to both EC2s') {
      steps {
        withCredentials([
          string(credentialsId: 'database-url', variable: 'DATABASE_URL'),
          string(credentialsId: 'payload-secret', variable: 'PAYLOAD_SECRET')
        ]) {
          sshagent(credentials: ['ec2-ssh-key']) {
            sh '''
              set -e
              for HOST in ${APP1_HOST} ${APP2_HOST}; do
                echo "Deploying to $HOST"
                ssh -o StrictHostKeyChecking=no ${SSH_USER}@$HOST "\
                  aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI} && \
                  docker pull ${ECR_URI}:${IMAGE_TAG} && \
                  docker stop payload-cms || true && \
                  docker rm payload-cms || true && \
                  docker run -d --name payload-cms --restart unless-stopped \
                    -p 3000:3000 \
                    --env-file /home/ec2-user/app.env \
                    ${ECR_URI}:${IMAGE_TAG} && \
                  docker image prune -f"
              done
            '''
          }
        }
      }
    }

    stage('Smoke check') {
      steps {
        sh '''
          sleep 20
          curl -fsS -o /dev/null -w "%{http_code}\n" http://${APP1_HOST}:3000/admin || true
          curl -fsS -o /dev/null -w "%{http_code}\n" http://${APP2_HOST}:3000/admin || true
        '''
      }
    }
  }
}
```

Notes:

- App security group currently allows `:3000` only from the ALB, **not** from Jenkins. Smoke `curl` to private `:3000` from Jenkins may fail. That is OK — the ALB health check is the real test. Remove the smoke stage or add a temporary `:3000` rule from Jenkins SG if you want it green.
- `--env-file /home/ec2-user/app.env` is the runtime source of RDS credentials.

Commit and push:

```powershell
git add Jenkinsfile
git commit -m "Add Jenkins CI/CD pipeline for ECR and dual-EC2 deploy"
git push
```

### 15.2 Create the Jenkins job

1. Jenkins home → **New Item**.
2. Name: `payload-cms-app`.
3. Type: **Pipeline**.
4. Pipeline definition: **Pipeline script from SCM**.
5. SCM: Git.
6. Repository URL: your GitHub HTTPS URL.
7. Credentials: `github-pat` if private.
8. Branch: `*/main`.
9. Script path: `Jenkinsfile`.
10. Save → **Build Now**.

Watch **Console Output**. First build often fails. Typical fixes:

| Symptom | Fix |
|---------|-----|
| `docker: permission denied` | `jenkins` not in `docker` group; reboot Jenkins EC2 |
| `Cannot perform an interactive login` / ECR 401 | Jenkins instance profile missing; wait 1 minute after attaching role |
| `repository does not exist` | ECR name mismatch |
| `Host key verification failed` | `StrictHostKeyChecking=no` missing |
| SSH timeout | App SG missing Jenkins SG on port 22 |
| `next build` DB error | `DATABASE_URL` credential wrong; RDS SG missing Jenkins public/NAT IP — Jenkins is in the VPC, but **build runs on Jenkins**, which is **not** in `payload-rds-sg`. Add Jenkins SG → RDS SG on 5432 **or** use dummy URL if the build does not actually connect. If the build connects to RDS, add inbound 5432 from `payload-jenkins-sg`. |
| Container starts then dies | `docker logs payload-cms` on the app instance; usually bad `DATABASE_URL` in `app.env` |
| ALB targets unhealthy | Container not listening `0.0.0.0:3000`; check `docker ps` and `curl localhost:3000/admin` **on the app instance** |

Add this RDS rule if the Docker build talks to Postgres:

| Type | Port | Source |
|------|------|--------|
| PostgreSQL | 5432 | `payload-jenkins-sg` |

### 15.3 GitHub webhook (CI on every push)

1. GitHub repo → **Settings** → **Webhooks** → **Add webhook**.
2. Payload URL: `http://JENKINS_PUBLIC_IP:8080/github-webhook/`
3. Content type: `application/json`.
4. Events: Just the push event.
5. Save.

6. Jenkins job → **Configure** → **Build Triggers** → check **GitHub hook trigger for GITScm polling**.

Test: change README, push, Jenkins should start a build without **Build Now**.

If the webhook shows a red delivery, Jenkins `8080` is not reachable from GitHub. Open `8080` to `0.0.0.0/0` for the demo, or use **Poll SCM** `H/5 * * * *` as a fallback.

---

## 16. Phase K — Make the endpoint work end-to-end

After a green Jenkins deploy:

1. On app-1:

```bash
docker ps
docker logs --tail 50 payload-cms
curl -I http://127.0.0.1:3000/admin
```

2. EC2 → Target Groups → `payload-app-tg` → **Targets**. Wait until both are **healthy**.
3. Browser:

```text
http://<ALB-DNS>/admin
```

4. Log in with the admin user created on your laptop in Section 9.4.  
   Same RDS ⇒ same users. That is the punchline of “RDS from the start.”

5. Optional: open the ALB DNS, refresh several times. You may still hit one instance (sticky sessions are off by default; both serve the same DB so the UI looks identical). To prove both nodes:

```bash
# on each app instance
curl -s http://127.0.0.1:3000/admin | head
```

And in the target group, both healthy.

If `/` is a frontend 404 and only `/admin` works, that is normal for the blank template. Use `/admin` as the demo URL.

---

## 17. Live demo script (10–12 minutes)

Use this as speaking notes. Have these tabs pre-opened:

1. This runbook
2. AWS VPC console
3. RDS
4. EC2 (instances + ALB + target group)
5. IAM roles
6. ECR
7. Jenkins
8. GitHub
9. Browser tab for `http://<ALB-DNS>/admin`

### 0:00 — One-sentence architecture

> Users hit an Application Load Balancer. The ALB sends traffic to two EC2 instances running the same Docker image. Both instances use one RDS PostgreSQL database. Jenkins on a third EC2 builds the image, pushes it to ECR, and deploys to both app servers. The VPC and security groups restrict who can talk to whom. IAM instance profiles replace hardcoded AWS keys.

### 0:01 — VPC and IAM

- Show `payload-vpc`, four subnets, IGW, public route `0.0.0.0/0 → igw`.
- Show security groups: ALB 80 from internet, App 3000 from ALB only, RDS 5432 from App SG + laptop IP, Jenkins 8080 from your IP.
- Show IAM roles `payload-ec2-app-role` (ECR pull) and `payload-ec2-jenkins-role` (ECR push) attached to the matching instances.

### 0:03 — RDS from the start

- Show RDS `payload-pg`, engine PostgreSQL, endpoint, VPC, public access, SG.
- Show laptop `.env` uses that endpoint, **not** localhost.
- Optional: pgAdmin `\dt` with Payload tables.
- Line to say: *We never created Docker Postgres or a local database. create-payload-app was given this RDS URL on day one.*

### 0:05 — Application and Docker

- Show GitHub repo: `Dockerfile`, `docker-compose.yml` with a single service, `Jenkinsfile`.
- Show ECR repository with image tags `latest` and a build number.

### 0:07 — Two EC2s + ALB endpoint

- Show `payload-app-1` and `payload-app-2` in two AZs.
- Show target group: **2 healthy**.
- Open `http://<ALB-DNS>/admin`.
- Log in. Dashboard loads. Create a User or edit the admin profile so there is a visible write.

### 0:09 — Jenkins CI/CD

- Show Jenkins job history (green).
- Make a tiny visible change (for example the admin project title in `payload.config.ts` `admin.meta.title`, or README). Push.
- Show webhook / new Jenkins build.
- When it finishes, hard-refresh `/admin` and point at the change **or** show `docker ps` with a new image tag on both instances.

```bash
docker inspect payload-cms --format '{{.Config.Image}}'
```

Run that on both app instances. Same tag.

### 0:11 — Wrap

> Endpoint is the ALB DNS. Data lives in RDS. Two app nodes. Jenkins deploys both. IAM and VPC isolate ECR, SSH, HTTP, and Postgres.

---

## 18. Files the GitHub repo should contain (checklist)

| Path | Purpose |
|------|---------|
| `src/payload.config.ts` | Payload config, `postgresAdapter`, `DATABASE_URL` |
| `next.config.mjs` | `output: 'standalone'` |
| `.env.example` | Placeholder env, no secrets |
| `.gitignore` | Must include `.env` |
| `Dockerfile` | Multi-stage production image |
| `.dockerignore` | Keeps secrets and `node_modules` out of the build |
| `docker-compose.yml` | App service only — **no Postgres service** |
| `Jenkinsfile` | Build → ECR → deploy app-1 and app-2 |
| `README.md` | Short pointer to this runbook |

---

## 19. README you can put in the GitHub repo

```markdown
# payload-cms-app

Basic Payload CMS 3 admin application.

- Database: Amazon RDS PostgreSQL (no local database)
- Runtime: Docker on two EC2 instances
- Entry point: Application Load Balancer (HTTP 80) → `/admin`
- CI/CD: Jenkins on EC2, image registry Amazon ECR

## Local development (still uses RDS)

1. Copy `.env.example` to `.env` and set `DATABASE_URL` to the RDS endpoint.
2. `pnpm install`
3. `pnpm dev`
4. Open http://localhost:3000/admin

## Production URL

http://<ALB-DNS>/admin
```

---

## 20. Troubleshooting (quick)

| Problem | Check |
|---------|--------|
| `ECONNREFUSED 127.0.0.1:5432` | `.env` still points at localhost. Fix `DATABASE_URL`. |
| `SASL ... password must be a string` | Empty password, wrong env name (`DATABASE_URI` vs `DATABASE_URL`), or template mismatch. |
| `no pg_hba.conf entry` / timeout | RDS SG, public access, wrong endpoint, SSL (`sslmode=require`). |
| Payload admin blank / 500 | `PAYLOAD_SECRET` changed after first init — keep the original secret. |
| Docker build OOM on Jenkins | Jenkins instance too small; use `t3.medium` or add 2 GB swap. |
| ALB `502` | Container not running; `docker logs payload-cms`. |
| ALB targets unhealthy, container fine | Health path wrong; use `/admin` and success codes `200-399`. Health check port must be 3000. |
| Logged in on localhost, not on ALB | Same RDS so users exist; if cookie `secure` is on, HTTP ALB will not keep the session. Keep Payload cookie secure **false** unless you add HTTPS. |
| CSRF / cookie issues on ALB HTTP | Set `SERVER_URL=http://<ALB-DNS>` (no trailing slash) in `app.env` and recreate containers. |
| Only one target healthy | App-2 image/env/SG/subnet. SSH and `docker logs`. |
| Webhook never fires | Jenkins 8080 not public; use Poll SCM. |

**Cookie / URL note:** After the app is public, add to `app.env` on both instances:

```env
SERVER_URL=http://payload-alb-xxxx.ap-south-1.elb.amazonaws.com
```

Redeploy. Payload uses this for admin URLs and cookies.

---

## 21. Cleanup (do this after the demo)

Delete in this order so AWS does not block you:

1. Jenkins job (optional) → terminate `payload-jenkins`
2. ALB listener is deleted with the ALB → delete load balancer `payload-alb`
3. Delete target group `payload-app-tg`
4. Terminate `payload-app-1` and `payload-app-2`
5. RDS → delete `payload-pg` → uncheck final snapshot if this is a lab → wait until gone
6. Delete DB subnet group
7. Delete ECR images then repository
8. Release any Elastic IP you accidentally allocated
9. Delete security groups (RDS SG first often fails until RDS is gone)
10. Delete subnets, IGW detach+delete, route tables, VPC
11. Delete IAM roles `payload-ec2-app-role`, `payload-ec2-jenkins-role`
12. Delete key pair in AWS (keep the local `.pem` deleted too)

Confirm **EC2 → Instances**, **RDS**, **Load Balancers**, **NAT Gateways** (should be none) are empty.

---

## 22. Evaluator rubric (self-check before you present)

- [ ] Payload Admin UI loads on the ALB URL
- [ ] First-user / Users data is in **RDS**, not SQLite, not Docker Postgres
- [ ] `.env` / `create-payload-app` used the RDS endpoint from the beginning
- [ ] `docker-compose.yml` has no database service
- [ ] ECR shows the image Jenkins built
- [ ] Jenkinsfile is in Git; a push produces a build
- [ ] Two EC2 app instances, two AZs, both **healthy** in the target group
- [ ] ALB is internet-facing; you did not need Route 53
- [ ] Custom VPC with public + private subnets
- [ ] Four security groups with least-privilege ports
- [ ] App role ≠ Jenkins role (pull vs push)
- [ ] No access keys in the repo; `.pem` and `.env` are gitignored
- [ ] You can explain why RDS is in private subnets but publicly accessible for laptop init
- [ ] You know how to destroy everything

---

## 23. Suggested build order (one page)

```text
Day 1 (2–4 h)
  IAM user + two EC2 roles
  VPC, subnets, IGW, routes, four SGs, key pair
  RDS PostgreSQL
  psql test from laptop
  create-payload-app → RDS URL → pnpm dev → /admin first user
  GitHub repo

Day 2 (2–4 h)
  Dockerfile, standalone output, compose without Postgres
  ECR repo
  Two app EC2s + Jenkins EC2
  ALB + target group
  Jenkins plugins, credentials, Jenkinsfile
  First green build
  http://<ALB-DNS>/admin
  Webhook
  Rehearse Section 17
```

---

## 24. What “good enough” looks like vs extras (do not do extras unless asked)

**In scope**

- Blank Payload template, default dashboard
- Single-AZ RDS
- HTTP only (no ACM certificate, no domain)
- Manual AWS Console setup (Terraform not required)
- Jenkins SSH deploy (not ECS/EKS)

**Out of scope unless you have leftover time**

- Custom collections beyond the template
- HTTPS / Route 53
- NAT Gateway / fully private app subnets
- Auto Scaling Group
- Blue/green
- S3 uploads plugin
- Secrets Manager (nice, not required)

If the evaluator asks “why not private app subnets?” answer:

> Private app subnets need a NAT Gateway so instances can pull from ECR. For this lab we used public subnets plus security groups. RDS remains in private subnets and only accepts 5432 from the app security group and my IP.

That answer shows you understand VPC, which is the point of requirement 8.
