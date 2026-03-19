# mozaiqReadMe
# Mozaiq Infrastructure Overview

**Last Updated:** Dec 8, 2025
**Project:** Mozaiq Wholesale Matchmaking Platform  
**Architecture:** Microservices with AWS Serverless

---

## Table of Contents

1.  [Spin Up local environment](#spin-up-local)
2. [Current AWS Architecture](#current-aws-architecture)
3. [Terraform Strategy](#terraform-strategy)
4. [Deployment Pipeline](#deployment-pipeline)
5. [Resource Tagging](#resource-tagging)
6. [Microservice Patterns](#microservice-patterns)


---
## Local Environment Setup

### Prerequisites

- Docker and Docker Compose installed
- Git configured with access to repositories
- Terminal or command line access
- Node.js and npm installed
- Postico 2 (recommended for database visualization)

### Related Repositories

| Service | Repository |
|---------|-----------|
| Users/Auth Microservice | [wsapp-users](https://github.com/jduffey1990/wsapp-users) |
| Business Verification Microservice | [bus-verify-wsapp](https://github.com/jduffey1990/bus-verify-wsapp) |
| Company Management Microservice | [wsapp-companies](https://github.com/jduffey1990/wsapp-companies) |
| Frontend Application | [wsapp](https://github.com/jduffey1990/wsapp) |
| Infrastructure & Docs | [mozaiqReadMe](https://github.com/jduffey1990/mozaiqReadMe) |

### Step 1: Create Project Directory Structure
```bash
# Create root project directory
mkdir mozaiq-local
cd mozaiq-local

# Create subdirectories for each microservice
mkdir brandora-verify wsapp-companies wsapp-users wsapp mozaiqReadMe
```

### Step 2: Clone Repositories
```bash
# Clone business verification service
cd brandora-verify
git init
git remote add origin https://github.com/jduffey1990/bus-verify-wsapp.git
git pull origin main
cd ..

# Clone company management service
cd wsapp-companies
git init
git remote add origin https://github.com/jduffey1990/wsapp-companies.git
git pull origin main
cd ..

# Clone users/auth service
cd wsapp-users
git init
git remote add origin https://github.com/jduffey1990/wsapp-users.git
git pull origin main
cd ..

# Clone frontend application
cd wsapp
git init
git remote add origin https://github.com/jduffey1990/wsapp.git
git pull origin main
cd ..

# Clone infrastructure repository
cd mozaiqReadMe
git init
git remote add origin https://github.com/jduffey1990/mozaiqReadMe.git
git pull origin main
cd ..
```

### Step 3: Verify Remote Connections
```bash
# Verify each repository is connected correctly
cd brandora-verify && git remote -v && cd ..
cd wsapp-companies && git remote -v && cd ..
cd wsapp-users && git remote -v && cd ..
cd wsapp && git remote -v && cd ..
cd mozaiqReadMe && git remote -v && cd ..
```

**Expected output for each repository:**
```
origin  https://github.com/jduffey1990/[repo-name].git (fetch)
origin  https://github.com/jduffey1990/[repo-name].git (push)
```

### Step 4: Copy Docker Compose Configuration
```bash
# Copy docker-compose.yml from infrastructure repo to project root
cp mozaiqReadMe/docker-compose.yml .
```

### Final Directory Structure

Your project should now look like this:
```
mozaiq-local/
├── brandora-verify/         # Business verification microservice
├── wsapp-companies/         # Company management microservice
├── wsapp-users/            # Users/auth microservice
├── wsapp/                  # Frontend application
├── mozaiqReadMe/           # Infrastructure & documentation
└── docker-compose.yml      # Docker orchestration config (copied from mozaiqReadMe)
```

### Step 5: Build and Start Services
```bash
# Ensure you're in the root directory
cd mozaiq-local

# Build all Docker containers
docker compose build

# Start all services in detached mode
docker compose up -d

# View running containers
docker compose ps

# View logs from all services
docker compose logs -f

# View logs from a specific service
docker compose logs -f [service-name]
```

### Common Docker Compose Commands
```bash
# Stop all services
docker compose down

# Stop and remove volumes (clean slate)
docker compose down -v

# Restart a specific service
docker compose restart [service-name]

# Rebuild a specific service
docker compose build [service-name]

# Run a command in a service container
docker compose exec [service-name] [command]
```

### Step 6: Install Database Visualization Tool

To visualize and manage your databases, install Postico 2 (recommended to maintain consistency across the team):

**macOS:**
```bash
# Download from official website
open https://eggerapps.at/postico2/

# Or install via Homebrew
brew install --cask postico
```

**Alternative Options:**
- pgAdmin
- DBeaver
- TablePlus

> **Note:** Using Postico 2 helps minimize environment-related errors across the team.

### Step 7: Run Database Migrations and Seeds

#### Companies Service
```bash
# Navigate to companies service
cd wsapp-companies

# Run migrations to set up database structure
npm run migrate:up

# Seed initial company data
npm run seed:companies

# Return to root
cd ..
```

#### Users Service
```bash
# Navigate to users service
cd wsapp-users

# Run migrations to set up database structure
npm run migrate:up

# Seed initial user data
npm run seed:users

# Return to root
cd ..
```

### Step 8: Configure Environment Variables

Environment variables contain sensitive configuration data and are not tracked in version control.

Each service repository contains a `.env.example` file listing every required variable with a placeholder value and a comment explaining where to get the real value. Use these as your starting point.

**To set up your environment files:**

1. Copy each example file and rename it to `.env`:
   ```bash
   cp wsapp-companies/.env.example wsapp-companies/.env
   cp wsapp-users/.env.example wsapp-users/.env
   cp brandora-verify/.env.example brandora-verify/.env
   cp wsapp/.env.example wsapp/.env
   ```
   > **Note:** The frontend repository clones into a folder called `Brandora` locally. Adjust the path if yours differs.

2. Fill in each placeholder value. Contact a team member or lead developer for:
   - `JWT_SECRET` — must be the same value across all backend services
   - `RECAPTCHA_SECRET_KEY` / `VITE_RECAPTCHA_SITE_KEY` — paired keys from Google reCAPTCHA console
   - `GOOGLE_PLACES_API_KEY` — Google Cloud Console (Maps Platform > Places API)
   - `GPT_API_KEY` — OpenAI platform
   - `VITE_MAPBOX_TOKEN` — Mapbox account dashboard
   - AWS credentials — use a dedicated IAM user with minimal permissions (SES SendEmail only for local dev), **never use root or personal AWS credentials**

3. For local development, the `DATABASE_URL` defaults in `.env.example` already match the Docker Compose configuration — you typically do not need to change them.

> **Security Note:** Never commit `.env` files to version control. These files should already be listed in `.gitignore`. Never share real AWS credentials directly — create a scoped IAM user instead.

### Step 9: Start the Application
```bash
# In the root directory, start all Docker services
docker compose up

# In a separate terminal, navigate to the frontend
cd wsapp

# Start the frontend development server
npm run dev
```

### Verification Checklist

- [ ] All Docker containers are running (`docker compose ps`)
- [ ] Database migrations completed without errors
- [ ] Seed data visible in Postico 2
- [ ] Frontend accessible at development URL
- [ ] All services responding to health checks

### Troubleshooting

**Containers won't start:**
```bash
# Check logs for specific service
docker compose logs [service-name]

# Restart with fresh build
docker compose down -v
docker compose build --no-cache
docker compose up
```

**Migration errors:**
```bash
# Roll back migrations
npm run migrate:down

# Re-run migrations
npm run migrate:up
```

**Port conflicts:**
```bash
# Check what's using the port
lsof -i :[port-number]

# Kill the process or modify docker-compose.yml ports
```
---

## Current AWS Architecture

### Overview

Mozaiq uses a serverless microservices architecture on AWS with three main services:

1. **Users Service** - Authentication, user management
2. **Companies Service** - Company profiles, invitations, verification
3. **Business Verification Service** - Web scraping, AI enrichment, company validation

### Core AWS Components

#### Lambda Functions
- **Runtime:** Node.js (typically)
- **Memory:** 512MB - 1024MB (varies by service)
- **Timeout:** 30-60 seconds
- **Cold Start** about 1 second
- **Networking:** VPC-enabled for database access
- **Environment Variables:**
  - `DATABASE_URL` - PostgreSQL connection string (Neon)
  - `JWT_SECRET` - Shared secret for JWT validation
  - `NODE_ENV` - production
  - `RECAPTCHA_SECRET_KEY` - For form protection

#### API Gateway (HTTP API)
- **Type:** API Gateway v2 (HTTP API)
- **Integration:** Lambda proxy integration
- **CORS:** Configured for frontend domains
- **Routes:** Defined per microservice
- **Custom Domain:** TBD

#### CodePipeline + CodeBuild
**Purpose:** Continuous deployment from GitHub to Lambda

**Pipeline Stages:**
1. **Source Stage:** Monitors GitHub repository
2. **Build Stage:** Runs CodeBuild project

**CodeBuild Process:**
- Uses `buildspec.yml` from repository
- Installs dependencies (`npm ci`)
- Runs database migrations (`npm run migrate:up`)
- Compiles TypeScript (`npm run build`)
- Strips dev dependencies
- Executes `deploy.js` to update Lambda code

**Key Files:**
- `buildspec.yml` - Build commands and phases
- `deploy.js` - Lambda code update logic (uses AWS SDK)

#### RDS PostgreSQL (External - Neon)
- **Hosting:** Neon (external managed service)
- **Connection:** SSL required with channel binding
- **Migrations:** Managed via `node-pg-migrate`
- **Per-Service Databases:**
  - `wsapp` (users)
  - `wsapp_companies` (companies)
  - Separate DBs for isolation and migration independence

#### S3 Buckets
- **Pipeline Artifacts:** Stores CodePipeline build artifacts
- **Naming:** 
    - `brandora-jduffey` (primary CDN bucket) 
    - `brandora-companies-pipeline-artifacts` (Terraform image bucket) 
- **Versioning:** Enabled for artifact history

#### IAM Roles

**Lambda Execution Role:**
- CloudWatch Logs access
- VPC network interface management
- Secrets Manager access (for DB credentials)

**CodeBuild Service Role:**
- S3 artifact access
- CloudWatch Logs
- Lambda function update permissions
- ECR access (if using Docker)

**CodePipeline Service Role:**
- S3 artifact access
- CodeBuild project execution
- GitHub source access (via OAuth token)

---

## Terraform Strategy

### Philosophy

**Terraform creates infrastructure, CodePipeline deploys code.**

Terraform is used for **one-time infrastructure setup** and **updates to infrastructure configuration**, not for continuous application deployment. Once infrastructure exists, your existing `buildspec.yml` and `deploy.js` handle all code deployments automatically.

### Current Approach: Self-Contained Per-Service

Each microservice repository contains its own complete, self-contained Terraform configuration. This approach prioritizes simplicity and service independence over DRY principles.

**Why this approach:**
- Each microservice is in its own GitHub repository
- Infrastructure lives with the service it describes
- No external dependencies on shared modules
- Easy to understand and modify per service
- Can evolve each service's infrastructure independently

**Future consideration:** If infrastructure patterns become highly repetitive, consider extracting to a shared module repository.

### Division of Responsibilities

#### Terraform Creates (Infrastructure Layer)
✅ Lambda function (empty shell)  
✅ API Gateway and routes  
✅ CodeBuild project  
✅ CodePipeline  
✅ IAM roles and policies  
✅ S3 bucket for terraform artifacts   
✅ Resource tags  

#### CodePipeline Deploys (Application Layer)
✅ Compiles TypeScript  
✅ Runs database migrations  
✅ Packages application code  
✅ Updates Lambda function code  
✅ Installs npm dependencies  

### Terraform File Structure

Company microservice repository contains:

```
wsapp-companies/                    # GitHub repo
├── src/                            # Application code
├── Dockerfile
├── buildspec.yml
├── deploy.js
├── package.json
└── terraform/                      # Infrastructure as code
    ├── main.tf                     # Lambda, API Gateway, CodeBuild, Pipeline
    ├── iam.tf                      # IAM roles and policies
    ├── variables.tf                # Input parameters (with defaults)
    ├── outputs.tf                  # Return values after deployment
    ├── backend.tf                  # Terraform state configuration
    ├── placeholder.zip             # Initial Lambda deployment package
    └── .gitignore                  # Ignore Terraform state files
```

### Key Terraform Concepts Used

**1. Self-Contained Resources**
All infrastructure defined directly in `main.tf` and `iam.tf` - no module references. This means:
```hcl
resource "aws_lambda_function" "main" {
  function_name = "wsapp-${var.service_name}"
  role          = aws_iam_role.lambda_execution.arn
  # ... all configuration inline
}
```

**2. Variables with Defaults**
Each service can customize via variables, but defaults make it simple:
```hcl
variable "service_name" {
  description = "Name of the microservice"
  type        = string
  default     = "companies"  # Default right in the code
}
```

**3. Outputs for Verification**
After deployment, Terraform displays useful information:
```hcl
output "api_gateway_url" {
  value = aws_apigatewayv2_stage.default.invoke_url
}
```

**4. Automatic Dependency Resolution**
Terraform reads all `.tf` files and determines creation order:
```hcl
resource "aws_lambda_function" "main" {
  role = aws_iam_role.lambda_execution.arn  # Terraform knows: create role first
}
```

**5. State Management**
```hcl
backend "s3" {
  bucket = "brandora-terraform-state"
  key    = "microservices/companies/terraform.tfstate"
}
```
State tracks what Terraform created so it can update/destroy later.

### Terraform Workflow

**Initial Setup (completed, not a local change):**
```bash
cd wsapp-companies/terraform
terraform init        # Download providers, initialize backend
terraform plan        # Preview changes
terraform apply       # Create resources
```

**Infrastructure Updates:**
```bash
terraform plan        # Preview changes
terraform apply       # Apply changes
```

**Teardown (if needed):**
```bash
terraform destroy     # Delete all resources
```

### What Terraform Doesn't Touch

❌ Application code (handled by git push)  
❌ Production Database migrations (handled by buildspec.yml)  
❌ npm dependencies (handled by CodeBuild)  
❌ Lambda function code versions (handled by deploy.js)  

### Secret Management

**Approach:** Secrets stored in ignored Terraform code.

**Options:**
1. **Manual Entry:** Create resources with Terraform, manually add secrets via AWS Console
2. **Environment Files:** Use `.tfvars` files (gitignored) for sensitive values

**Current Recommendation:**
- Create infrastructure with Terraform (companies only now)
- Add Terraform infrastructure for Users and Verify
- Manually add environment variables in AWS Lambda Console for Users and Verify Microservice:
  - `DATABASE_URL`
  - `JWT_SECRET`
  - `RECAPTCHA_SECRET_KEY`
- Enter ENVs in main.tf and variables.tf and manage secrets with terraform.tfvars

**Future Enhancement (cost money):**
```hcl
# Use Secrets Manager
data "aws_secretsmanager_secret_version" "db_url" {
  secret_id = "brandora/companies/database_url"
}

# Reference in Lambda
environment {
  variables = {
    DATABASE_URL = data.aws_secretsmanager_secret_version.db_url.secret_string
  }
}
```

---

## Deployment Pipeline

### Complete Deployment Flow

```
┌─────────────────────────────────────────────────────────┐
│ ONE-TIME SETUP (Terraform)                              │
│                                                          │
│ terraform apply                                         │
│   ↓                                                     │
│ Creates:                                                │
│   • Lambda function (empty placeholder)                 │
│   • API Gateway endpoints                               │
│   • CodeBuild project (references buildspec.yml)        │
│   • CodePipeline (monitors GitHub)                      │
│   • ElastiCache                                         │
│   • IAM roles                                           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ CONTINUOUS DEPLOYMENT (Automatic)                       │
│                                                          │
│ Developer: git push origin main                         │
│   ↓                                                     │
│ CodePipeline detects GitHub change                      │
│   ↓                                                     │
│ CodeBuild triggered, runs buildspec.yml:                │
│   1. npm ci                                             │
│   2. npm run migrate:up      ← Database migrations      │
│   3. npm run build           ← Compile TypeScript       │
│   4. npm ci --omit=dev       ← Production dependencies  │
│   5. node deploy.js          ← Update Lambda code       │
│   ↓                                                     │
│ Lambda function now has latest code                     │
└─────────────────────────────────────────────────────────┘
```

### buildspec.yml Structure

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: 18
  pre_build:
    commands:
      - npm ci
      - npm run migrate:up    # Run database migrations
  build:
    commands:
      - npm run build         # Compile TypeScript
      - npm ci --omit=dev     # Remove dev dependencies
  post_build:
    commands:
      - node deploy.js        # Update Lambda function
```

### deploy.js Logic

```javascript
// 1. Package code into zip
const zip = new AdmZip();
zip.addLocalFolder('./dist');
zip.addLocalFolder('./node_modules');

// 2. Update Lambda function
await lambda.updateFunctionCode({
  FunctionName: process.env.LAMBDA_FUNCTION_NAME,
  ZipFile: zip.toBuffer()
});

// 3. Publish new version (optional)
await lambda.publishVersion({
  FunctionName: process.env.LAMBDA_FUNCTION_NAME
});
```

---

## Resource Tagging

### Tagging Strategy

**Purpose:** Organize resources for cost tracking, access control, and automation.

**Standard Tags:**
```
Project = brandora
Service = {users|companies|verify}
Owner = jduffey
Environment = production
ManagedBy = terraform
```

### Current Tagging Issues

- **Problem:** Billing/Cost Explorer doesn't aggregate well
- **Solution:** Standardized tags across all resources
- **Tool:** AWS Resource Groups & Tag Editor

### Creating Resource Groups

```bash
# View all resources by service
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Project,Values=brandora Key=Service,Values=companies

# Create a resource group via AWS Console:
# Resource Groups → Create Group
# Tag-based: Project=brandora, Service=companies
```

### Retroactive Tagging Script

If resources were created before standardized tagging:

```bash
#!/bin/bash
# retag-resources.sh

SERVICE="companies"
RESOURCES=$(aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Project,Values=brandora \
  --query 'ResourceTagMappingList[].ResourceARN' --output text)

for arn in $RESOURCES; do
  aws resourcegroupstaggingapi tag-resources \
    --resource-arn-list "$arn" \
    --tags Service=$SERVICE,ManagedBy=terraform
done
```

---

## Microservice Patterns

### Standard Microservice Structure

Each microservice follows this pattern:

```
wsapp-{service}/
├── src/
│   ├── app.ts                  # Hapi server entry point
│   ├── routes/                 # API route handlers
│   ├── controllers/            # Business logic
│   ├── models/                 # TypeScript interfaces
│   ├── migrations/             # node-pg-migrate files
│   └── scripts/                # Seed scripts, utilities
├── dist/                       # Compiled JavaScript (gitignored)
├── Dockerfile                  # For local development
├── buildspec.yml               # CodeBuild instructions
├── deploy.js                   # Lambda deployment script
├── package.json
├── tsconfig.json
└── terraform/                  # Infrastructure as code
    └── (uses ../microservice module)
```

### Authentication Pattern

**JWT Strategy:**
- Shared `JWT_SECRET` across all microservices
- Token contains: amalgamation of various user fields (see source code)
- No inter-service communication needed for auth
- Client caches JWT in localStorage
- Protected routes use Hapi's JWT strategy

### Database Pattern

**Per-Service Databases:**
- Each microservice has its own PostgreSQL database
- Enables independent scaling and migration
- Avoids schema conflicts

**Local Development:**
```yaml
# docker-compose.yml
services:
  postgres-users:
    ports: ["5432:5432"]
  postgres-companies:
    ports: ["5433:5432"]  # Different host port
```

### Invitation Code Pattern (Companies Service)

**Use Case:** Users generate time-limited codes to invite others to their company.

**Implementation:**
```typescript
interface CompanyCode {
  id: string;
  companyId: string;
  code: string;           // Random 8-character code
  createdBy: string;      // User ID
  expiresAt: Date;        // 24 hours from creation
  usedBy?: string;        // Null until redeemed
  usedAt?: Date;
}

### Company Verification Pattern

**Multi-Signal Confidence Scoring:**

Rather than requiring a single "proof" (like tax ID), the verification service builds confidence from multiple signals:

1. **Domain Name** (primary anchor)
2. **LinkedIn Company Page** (cross-validation)
3. **Google Places** (location verification)
4. **Website Metadata** (JSON-LD, Open Graph)
5. **AI Enrichment** (only when structured data insufficient)

**Data Flow:**
```
User Input (URL/LinkedIn/Address)
  ↓
Web Scraping (Cheerio) or API calls
  ↓
Extract structured data (JSON-LD, Schema.org)
  ↓
Cross-reference signals
  ↓
Calculate confidence score
  ↓
AI enrichment (if needed) with strict validation
  ↓
Return company profile
```

**Key Principles:**
- Domain is the source of truth
- Multiple weak signals > single strong signal
- AI used strategically, not as primary source
- Strict validation to prevent hallucinations

---

## Future Enhancements

### Near Term
- [ ] Move secrets to AWS Secrets Manager
- [ ] Add custom domain with Route53
- [ ] Implement API Gateway request validation
- [ ] Add CloudWatch alarms for errors/latency

### Medium Term
- [ ] Multi-environment setup (dev/staging/prod)
- [ ] Terraform remote state with DynamoDB locking
- [ ] VPC setup with private subnets
- [ ] ElastiCache connection pooling optimization

### Long Term
- [ ] Blue/green deployments
- [ ] Multi-region failover
- [ ] API Gateway usage plans and API keys
- [ ] Comprehensive monitoring with X-Ray

---

## Cost Optimization Tips

**Lambda:**
- Use ARM64 architecture (Graviton2) for 20% cost savings
- Optimize bcrypt rounds (reduced from 10→8 for performance)
- Implement response caching where appropriate

**API Gateway:**
- Use HTTP API (not REST API) - cheaper, faster
- Enable caching for frequently accessed endpoints

**CodePipeline:**
- Single pipeline per service (not per branch)
- S3 lifecycle policies to delete old artifacts after 30 days

**Data Transfer:**
- Keep services in same region
- Use VPC endpoints to avoid internet egress charges

---

## Quick Reference

### Common Commands

**AWS CLI:**
```bash
# View Lambda function
aws lambda get-function --function-name wsapp-companies

# Invoke Lambda manually
aws lambda invoke --function-name wsapp-companies \
  --payload '{"httpMethod":"GET","path":"/health"}' response.json

# View CodePipeline status
aws codepipeline get-pipeline-state --name wsapp-companies-pipe

# View logs
aws logs tail /aws/lambda/wsapp-companies --follow
```

**Database Migrations:**
```bash
npm run migrate:create my-migration-name
npm run migrate:up
npm run migrate:down
npm run migrate:reset
```

### Important URLs

- AWS Console: https://console.aws.amazon.com
- Terraform Docs: https://registry.terraform.io/providers/hashicorp/aws

---

## Troubleshooting

### Lambda won't update after push

**Check:**
1. CodePipeline status: Is it running?
2. CodeBuild logs: Did build fail?
3. Lambda version: Is it the latest?

**Solution:**
```bash
aws codepipeline get-pipeline-state --name wsapp-companies-pipe
aws codebuild batch-get-builds --ids <build-id>
```

### Database connection errors

**Common causes:**
- Lambda not in VPC
- Security group blocking PostgreSQL port
- Incorrect connection string
- SSL certificate issues

**Check:**
```bash
aws lambda get-function-configuration --function-name wsapp-companies \
  | jq '.VpcConfig'
```

### Terraform state conflicts

**Error:** "State locked"

**Solution:**
```bash
# Force unlock (use carefully!)
terraform force-unlock <lock-id>
```

### API Gateway 502 errors

**Common causes:**
- Lambda timeout
- Lambda runtime error
- IAM permissions issue

**Check:**
```bash
aws logs tail /aws/lambda/wsapp-companies --since 5m
```

---

**Document Maintained By:** Jordan Duffey  
**Project Repository:** github.com/jduffey1990/wsapp-companies  
**Contact:** foxdogdevelopment@gmail.com







```
├──brandora-verify/      # has it's own github repo and its own AWS pathway (CodePipeline, API Gateway->Lambda).  No DB.  "Is this you?" logic
  ├── src/
  │   ├── lib/                  # enrichment logic and helpers
  │   ├── handlers/               # entry for different versions of the algorithm
  ├── dist/                       # Compiled JavaScript (gitignored)
  ├── Dockerfile                  # For local development
  ├── buildspec.yml               # CodeBuild instructions
  ├── deploy.js                   # Lambda deployment script
  ├── package.json
  ├── tsconfig.json
├──wsapp-companies/                # Most of the business logic, has it's own github repo and its own AWS pathway (CodePipeline, S3 for terraform builds, API Gateway->Lambda) and Neon Postgres DB
  ├── src/
  │   ├── app.ts                  # Hapi server entry point
  │   ├── routes/                 # API route handlers
  │   ├── controllers/            # Business logic
  │   ├── models/                 # TypeScript interfaces
  │   ├── migrations/             # node-pg-migrate files
  │   └── scripts/                # Seed scripts, utilities
  ├── dist/                       # Compiled JavaScript (gitignored)
  ├── Dockerfile                  # For local development
  ├── buildspec.yml               # CodeBuild instructions
  ├── deploy.js                   # Lambda deployment script
  ├── package.json
  ├── tsconfig.json
  └── terraform/                  # Infrastructure as code
      └── (uses ../microservice module)
├──wsapp-users/                   # user and auth logic, has it's own github repo and its own AWS pathway (CodePipeline, API Gateway->Lambda) and Neon Postgres DB
  ├── src/
  │   ├── app.ts                  # Hapi server entry point
  │   ├── routes/                 # API route handlers
  │   ├── controllers/            # Business logic
  │   ├── models/                 # TypeScript interfaces
  │   ├── migrations/             # node-pg-migrate files
  │   └── scripts/                # Seed scripts, utilities
  ├── dist/                       # Compiled JavaScript (gitignored)
  ├── Dockerfile                  # For local development
  ├── buildspec.yml               # CodeBuild instructions
  ├── deploy.js                   # Lambda deployment script
  ├── package.json
  ├── tsconfig.json
├──docker-compose.yml
```