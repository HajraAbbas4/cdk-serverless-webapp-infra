# AWS CDK Infrastructure Deployment

An intermediate-level Infrastructure as Code (IaC) project built with **AWS CDK v2** and **Python 3.12**, deploying a secure, scalable, serverless web application environment on AWS.

## What This Project Deploys

- **Amazon VPC** with public and private (egress) subnets across 2 Availability Zones
- **Internet Gateway** and **NAT Gateway** for outbound connectivity
- **Security Groups** following least-privilege principles (no unnecessary inbound access)
- **Amazon S3 bucket** with versioning, server-side encryption, and blocked public access
- **Amazon DynamoDB table** with Pay-Per-Request billing and point-in-time recovery
- **AWS Lambda function** (Python 3.12) running inside the VPC's private subnets
- **Amazon API Gateway** REST API with a Lambda proxy integration
- **IAM Roles and Policies** scoped to least privilege (no wildcard permissions)
- **CloudWatch Log Groups** with explicit retention for both Lambda and API Gateway
- **CloudWatch Dashboard** for at-a-glance monitoring of Lambda, API Gateway, and DynamoDB

See [`docs/architecture.md`](docs/architecture.md) for a full architecture diagram and stack-by-stack breakdown.

## Project Structure
infrastructure-as-code/
│
├── app.py                     # CDK app entry point — wires all stacks together
├── requirements.txt           # Python dependencies (aws-cdk-lib, constructs, pytest)
├── cdk.json                   # CDK Toolkit configuration
├── README.md                  # This file
├── .gitignore
├── LICENSE
│
├── stacks/
│   ├── init.py
│   ├── network_stack.py       # VPC, subnets, IGW, NAT Gateway, security groups
│   ├── storage_stack.py       # S3 bucket + DynamoDB table
│   ├── compute_stack.py       # Lambda function, IAM role, CloudWatch log group
│   ├── api_stack.py           # API Gateway REST API + Lambda integration
│   └── monitoring_stack.py    # CloudWatch Dashboard
│
├── lambda/
│   └── handler.py             # Lambda function source code
│
├── tests/
│   ├── init.py
│   └── test_stacks.py         # pytest + CDK assertions tests
│
├── conftest.py                # Ensures stacks package is importable by pytest
│
└── docs/
└── architecture.md        # Architecture overview and diagram

## Prerequisites

- **Python 3.12** installed locally
- **Node.js** (required by the AWS CDK Toolkit CLI) — v18 or later recommended
- An **AWS account** with credentials configured (e.g. via `aws configure`)
- The **AWS CDK Toolkit CLI** installed globally:

```bash
  npm install -g aws-cdk
```

## Setup

1. **Clone or extract this project**, then move into its directory:

```bash
   cd infrastructure-as-code
```

2. **Create and activate a Python virtual environment:**

```bash
   python3.12 -m venv .venv
   source .venv/bin/activate      # On Windows: .venv\Scripts\activate
```

3. **Install dependencies:**

```bash
   pip install -r requirements.txt
```

4. **Configure environment variables** (optional — sensible defaults are
   built in, but you should set these for a real deployment):

```bash
   export CDK_DEFAULT_ACCOUNT=123456789012      # your AWS account ID
   export CDK_DEFAULT_REGION=us-east-1          # target AWS region
   export PROJECT_NAME=iac-webapp               # used as a naming prefix
   export DEPLOY_ENV=dev                        # dev | stage | prod
   export NAT_GATEWAY_COUNT=1                   # NAT gateways to provision
   export LOG_RETENTION_DAYS=14                 # CloudWatch log retention
```

   > On `prod`, `StorageStack` automatically switches to `RemovalPolicy.RETAIN`
   > for the S3 bucket and DynamoDB table so `cdk destroy` will not delete
   > production data.

## Running Tests

The project includes CDK assertion tests using `pytest`. These synthesize
each stack in memory and assert that key resources/properties exist,
without needing real AWS credentials or making any API calls.

```bash
pytest -v
```

## Deployment Workflow

All commands below are run from the project root (where `app.py` and
`cdk.json` live), with your virtual environment activated.

### 1. Bootstrap your AWS environment (one-time per account/region)

```bash
cdk bootstrap aws://ACCOUNT-ID/REGION
```

Example:

```bash
cdk bootstrap aws://123456789012/us-east-1
```

### 2. Synthesize the CloudFormation templates

Validates the app and generates CloudFormation templates under `cdk.out/`:

```bash
cdk synth
```

### 3. Preview changes before deploying

Shows a diff between the currently deployed stacks and what would change:

```bash
cdk diff
```

### 4. Deploy the stacks

Deploy all stacks (you'll be prompted to approve any IAM/security-related
changes):

```bash
cdk deploy --all
```

Or deploy a single stack by name, e.g.:

```bash
cdk deploy iac-webapp-dev-NetworkStack
```

After a successful deployment, the API Gateway base URL is printed as a
CloudFormation output (`ApiEndpoint`).

### 5. Test the deployed API

```bash
curl -X POST https://<api-id>.execute-api.<region>.amazonaws.com/dev/items \
  -H "Content-Type: application/json" \
  -d '{"name": "example item"}'

curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/items
```

### 6. Clean up (destroy all resources)

To avoid ongoing AWS charges, destroy the stacks when you're done:

```bash
cdk destroy --all
```

> In `dev`/`stage` environments the S3 bucket and DynamoDB table are
> configured to auto-delete on stack destruction. In `prod`, they are
> **retained** by design — you'll need to delete them manually if you
> really want to remove that data.

## Stack Deployment Order

The stacks declare explicit dependencies in `app.py`, so `cdk deploy --all`
will always deploy (and `cdk destroy --all` will always tear down) in the
correct order:
NetworkStack → StorageStack → ComputeStack → ApiStack → MonitoringStack

## Security & Best Practices Applied

- **Least-privilege IAM**: the Lambda execution role only has the specific
  permissions it needs (via CDK's `grant_*` helpers), not broad managed
  policies with wildcard resources.
- **Least-privilege networking**: the Lambda security group has no inbound
  rules and restricts outbound traffic to HTTPS only.
- **Encryption everywhere**: S3 (SSE-S3) and DynamoDB (AWS-managed keys)
  are encrypted at rest; API Gateway and S3 enforce HTTPS/SSL.
- **No public S3 access**: all four S3 "block public access" settings are
  enabled.
- **Explicit log retention**: CloudWatch Log Groups are created with a
  defined retention period instead of the default "never expire" behavior.
- **Resource tagging**: every stack tags its resources with `Project`,
  `Environment`, and `Stack` for cost allocation and ownership tracking.
- **Defensive coding**: the Lambda handler wraps all logic in
  exception handling and returns clean, logged error responses instead of
  crashing.

## License

This project is licensed under the MIT License — see [`LICENSE`](LICENSE).
