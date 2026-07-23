# AIOps Assistant — Kira

An AI-powered SRE assistant built on AWS Bedrock Agent. Kira diagnoses production incidents by querying CloudWatch Logs, CloudWatch Metrics (via Prometheus), and EKS cluster health — then responds with root cause, evidence, and fix recommendations.

---

## Architecture

```
Streamlit UI (app.py)
      │
      ▼
Bedrock Agent (Kira)
      │
      ├── fetch_logs         → CloudWatch Logs
      ├── fetch_metrics      → Prometheus (ELB endpoint)
      └── fetch_service_health → EKS cluster + node groups
```

---

## Prerequisites

- AWS account with access to Bedrock (model access enabled for your chosen model)
- EKS cluster running with Prometheus exposed via a LoadBalancer service
- AWS CLI configured (`aws configure`)
- Python 3.10+

---

## Step 1: Set Up IAM Roles

Run the provided script to create both required IAM roles:

```bash
chmod +x setup-iam.sh
./setup-iam.sh
```

This creates:

| Role | Used By | Permissions |
|------|---------|-------------|
| `aiops-lambda-role` | All 3 Lambda functions | CloudWatch Logs read, EKS describe, Lambda basic execution |
| `aiops-bedrock-agent-role` | Bedrock Agent | Invoke the 3 Lambda functions, invoke Bedrock models |

---

## Step 2: Create the Lambda Functions

Create the following 3 Lambda functions in the AWS Console (or via CLI). Use the code from the `lambda/` directory.

| Function Name | Code File | Execution Role |
|---------------|-----------|----------------|
| `aiops-fetch-logs` | `lambda/fetch_logs/lambda_function.py` | `aiops-lambda-role` |
| `aiops-fetch-metrics` | `lambda/fetch_metrics/lambda_function.py` | `aiops-lambda-role` |
| `aiops-fetch-health` | `lambda/fetch_health/lambda_function.py` | `aiops-lambda-role` |

Runtime: **Python 3.12** | Timeout: **30 seconds**

---

## Step 3: Update the Prometheus URL

Both `fetch_metrics` and `fetch_health` lambdas query Prometheus directly. Update the `PROMETHEUS_URL` placeholder in each file before uploading the code.

In `lambda/fetch_metrics/lambda_function.py`:
```python
PROMETHEUS_URL = "http://<YOUR_PROMETHEUS_ELB_URL>:9090"
```

In `lambda/fetch_health/lambda_function.py`:
```python
PROMETHEUS_URL = "http://<YOUR_PROMETHEUS_ELB_URL>:9090"
```

To get the Prometheus ELB URL, expose Prometheus as a LoadBalancer service:

```bash
kubectl patch svc kube-prometheus-stack-prometheus -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc kube-prometheus-stack-prometheus -n monitoring
# Copy the EXTERNAL-IP value — that is your ELB URL
```

---

## Step 4: Deploy the Bedrock Agent

Run the deploy script. It will:
- Verify the Lambda functions and IAM role exist
- Set Lambda timeouts to 30s and add Bedrock invoke permissions
- Create the Bedrock Agent (`aiops-assistant`) with the Kira system prompt
- Attach all 3 action groups with their OpenAPI schemas
- Prepare the agent

```bash
chmod +x deploy.sh
./deploy.sh
```

At the end, the script prints your **Agent ID** — keep it for the next step.

---

## Step 5: (Optional) Generate Sample Data

Populate CloudWatch Logs with realistic error scenarios to test Kira:

```bash
python3 scripts/generate_sample_data.py --region us-east-1
```

This writes 100 realistic log events (503 errors, OOM kills, connection pool exhaustion, etc.) to `/app/production`.

---

## Step 6: Run the Streamlit UI

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```env
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<YOUR_AGENT_ID>
BEDROCK_AGENT_ALIAS_ID=TSTALIASID

# Optional — omit to use your AWS CLI profile / SSO / IAM role:
# AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY>
# AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY>
# AWS_SESSION_TOKEN=<YOUR_SESSION_TOKEN>
```

Install dependencies and start the UI:

```bash
pip install -r requirements.txt
streamlit run app.py
```

Open **http://localhost:8501** in your browser.

---

## Project Structure

```
aiops-assistant/
├── app.py                  # Streamlit chat UI
├── deploy.sh               # Bedrock Agent deployment script
├── setup-iam.sh            # IAM roles and policies setup
├── requirements.txt        # Python dependencies
├── .env.example            # Environment variable template
├── lambda/
│   ├── fetch_logs/         # CloudWatch Logs query
│   ├── fetch_metrics/      # Prometheus metrics query
│   └── fetch_health/       # EKS cluster health check
├── schemas/
│   ├── fetch_logs.json     # OpenAPI schema for fetch_logs
│   ├── fetch_metrics.json  # OpenAPI schema for fetch_metrics
│   └── fetch_health.json   # OpenAPI schema for fetch_health
└── scripts/
    └── generate_sample_data.py  # Seed CloudWatch with test errors
```

---

## Sample Questions to Ask Kira

- Why are we seeing 503 errors in the last hour?
- Is CPU usage high across the boutique services?
- Check database connections and latency
- Are all pods healthy? Any restarts?
- What are the most frequent errors in the last 2 hours?

---

## Potential Issues

### Bedrock model access not enabled
The deploy script will fail at agent creation if model access hasn't been requested. Go to **AWS Console → Bedrock → Model access** and enable access for the model used in `deploy.sh` before running the script.

### Prometheus URL unreachable from Lambda
`fetch_metrics` and `fetch_health` make outbound HTTP calls to the Prometheus ELB. If Lambda is deployed inside a VPC without a NAT gateway or internet gateway route, these calls will time out. Either:
- Keep Lambda outside a VPC (default), or
- Ensure the VPC has a route to the internet and the Prometheus ELB security group allows inbound on port 9090.

### Agent stuck in PREPARING state
After running `deploy.sh`, the agent status shows `PREPARING`. This is normal and takes 30–60 seconds. If it stays in this state, check the Bedrock console for validation errors — usually caused by a malformed OpenAPI schema or a Lambda ARN that doesn't exist.

### Streamlit shows "NOT CONFIGURED"
The app requires `BEDROCK_AGENT_ID` and `BEDROCK_AGENT_ALIAS_ID` to be set in `.env`. If you started Streamlit before populating `.env`, stop it and restart — `load_dotenv()` only reads the file at startup.

```bash
# Stop and restart
pkill -f "streamlit run app.py"
streamlit run app.py
```

### fetch_logs returns no results
The default log group is `/eks/boutique/pods`. This group is only created after Fluent Bit starts shipping logs. Make sure `aws-for-fluent-bit` is running:

```bash
kubectl get pods -n amazon-cloudwatch
```

If the log group doesn't exist yet, run the sample data generator first (Step 5) which creates `/app/production`.

### fetch_health uses wrong cluster name
The Lambda defaults to cluster name `eks-cluster`. If your cluster has a different name, update `DEFAULT_CLUSTER` in `lambda/fetch_health/lambda_function.py` before uploading the function code.

### Lambda execution role missing permissions
If `fetch_health` returns an access denied error on `eks:DescribeCluster`, the inline policy may not have propagated yet (IAM can take ~10–15 seconds). Wait and retry. If it persists, verify the inline policy is attached:

```bash
aws iam get-role-policy \
  --role-name aiops-lambda-role \
  --policy-name aiops-lambda-inline-policy
```

### AWS credentials not resolving in Streamlit
If `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are left blank in `.env`, boto3 falls back to the default credential chain (`~/.aws/credentials`, environment variables, IAM role). If none of those are configured, Bedrock calls will fail with an auth error. Either fill in the credentials in `.env` or ensure your terminal session has valid AWS credentials before starting Streamlit.

# My notes

## Important Concepts

### Foundational Model
Pre-trained large AI models ready to use
No need to train, just leverage them

### Model as an API
Access AI models via simple http calls
No need to host or manage infrastructure

### Inference
Model takes input and returns output
We give data > It generates results

### Prompt Engineering
How we ask = What we get
Better prompts lead to better asnwers

### System Prompt
Defines AI's role and behaviour
Set boundaries before interaction starts

### Agent
Not just answering -- it acts
Decides steps and uses tools autonomously

### Tool calling
AI can trgger external functions
Fetches data and performs actions dynamically

### Context Window
AI's short-term memory
Holds conversations, logs, and data context

### Serverless
Run code without managing servers
Pay only when your function executes

### Event-Driver Architecture
Nothing runs until triggered
Every action starts from an event

## Cloud services

### Bedrock
Bedrock is AWS Managed AI Platform
Instead of creating our own GPU cluster, hosting the model; bedrock gives us access to foundation models like claude, lama, quenn etc. through a simple api

### Bedrock Agent
Adds reasoning layer to the application, so AI can analyse problems and take actions

### Lambda
Serverless functions that execute tasks on demand for the agent
If agent is the brain, lambda functions are the hands
Each lambda function that we implement has its specific capability that agent can reach for, like need to check logs, call lambda 1, need to see if pods are crashing, call lambda 2 etc. 
These lambda functions talk to AWS APIs that we already have, so new infrastructure , we are only putting AI layer on top of the existing infrastructure

### Cloudwatch
Push logs from application to AWS Cloudwatch

### Streamlit
Builds a simple interactive UI to communicate with the AI agents, it is like frontend
It is a python UI framework which is perfect for creating interactive webapps using python scripts
Streamlit will help us deploy the UI and we will be interacting with the agent through it

## High level steps:

Step 1 : Export EKS logs to Cloudwatch

Step 2: Add cloudwatch, its logs, create loggroups etc permissions to eks role

Step 3: downlaod fluentbit via helm and install

Step 4: Make the kube-prometheus-stack service as type load balancer and update the value in functions

Step 5 Deploy three action groups, which are three lambda functions for us

Step 6: Create role with bedrock permissions to invoke the model and lambda fucntions

Step 7: Go to Bedrock > Model Catalog

Step 8: Run deployment script to deploy the agent via deploy.sh