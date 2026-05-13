# Sophos Central EC2 Auto-Deregistration

Automatically removes Sophos Central endpoint licences when AWS EC2 instances
are terminated. No manual cleanup needed.

## How it works

```
EC2 terminated
      |
      v
EventBridge rule fires  (state = terminated)
      |
      v
Lambda searches Sophos  (?cloud=instance-id)
      |
      v
Endpoint deleted - Licence seat released
```

---

Pre-requisite
Create API key from Sophos Central using the doc https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/FirewallManagement/AWSAutoscaling/ConfigureAWS/CreateAPICredentials/index.html 
save the client ID and Client secret 

## Deploy to AWS
1) Download the cloudformation template from the GitHub repo named sophos-deregister.yaml.
2) Navigate to your AWS account.
3) Search for Cloudformation >Create Stack>Upload a template file 
Download the sophos-deregister.yaml from GitHub and upload the file here
4) Fill in the parameters as below

a)Sophos Client ID : Paste the Sophos Central API key ID
b)Sophos Client Secret: Paste the Sophos Central API client secret
c)Secret Manager-Secret Name : Change the name as the default one most likely exists in Sophos NSG IAAS playground
d)Lambda - Function Name:Provide suitable name/leave the default one  
e)EventBridge - Rule Name:Provide suitable name/leave the default one
f)EventBridge - Rule State on Deploy : ENABLED

Create the stack 




---

## What gets created

| Resource | Default name | AWS Console navigation |
|----------|-------------|------------------------|
| Secrets Manager secret | `sophos/central-api` | AWS Console > Secrets Manager > Secrets |
| IAM Role | `sophos-deregister-execution-role` | AWS Console > IAM > Roles |
| Lambda function | `sophos-deregister` | AWS Console > Lambda > Functions |
| EventBridge rule | `sophos-ec2-termination` | AWS Console > EventBridge > Rules |
| Lambda permission | `allow-eventbridge-sophos` | AWS Console > Lambda > [fn] > Configuration > Permissions |

---

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `SophosClientId` | Yes | - | OAuth2 Client ID from Sophos Central |
| `SophosClientSecret` | Yes | - | OAuth2 Client Secret from Sophos Central |
| `SophosSecretName` | No | `sophos/central-api` | Secrets Manager secret name |
| `DeploymentRegion` | No | `ap-south-1` | AWS region (19 regions available in dropdown) |
| `LambdaFunctionName` | No | `sophos-deregister` | Lambda function name |
| `EventBridgeRuleName` | No | `sophos-ec2-termination` | EventBridge rule name |
| `EventBridgeRuleState` | No | `ENABLED` | ENABLED fires immediately, DISABLED requires manual activation |

---

## Setup

### What you need before starting

| Item | Where to get it |
|------|----------------|
| Sophos Client ID | Sophos Central > Global Settings > API Credentials Management > Add Credential |
| Sophos Client Secret | Same place - shown only once, copy before closing |
| AWS IAM user credentials | AWS Console > IAM > Users > [user] > Security credentials > Create access key |
| S3 bucket name | You create this in Step 1 below (must be globally unique) |

---

### Step 1 - Create the S3 bucket

The template must be in a public S3 bucket so CloudFormation can fetch it
when someone clicks the Deploy to AWS button from this private repo.

```bash
# Create the bucket (replace values with your own)
aws s3 mb s3://YOUR_BUCKET_NAME --region YOUR_REGION

# Enable versioning to keep a history of template versions
aws s3api put-bucket-versioning \
  --bucket YOUR_BUCKET_NAME \
  --versioning-configuration Status=Enabled
```

---

### Step 2 - Add GitHub Actions secrets

Navigation: **GitHub repo > Settings > Secrets and variables > Actions > New repository secret**

Add all six secrets:

| Secret name | Value | Where to find it |
|-------------|-------|-----------------|
| `AWS_ACCESS_KEY_ID` | e.g. `AKIAIOSFODNN7EXAMPLE` | AWS Console > IAM > Users > [user] > Security credentials |
| `AWS_SECRET_ACCESS_KEY` | e.g. `wJalrXUtnFEMI/K7MDENG` | Same page - shown once at creation |
| `AWS_REGION` | e.g. `ap-south-1` | The region where your EC2 instances run |
| `S3_BUCKET_NAME` | e.g. `my-sophos-templates` | The bucket you created in Step 1 |
| `SOPHOS_CLIENT_ID` | Your Sophos Client ID | Sophos Central > Global Settings > API Credentials Management |
| `SOPHOS_CLIENT_SECRET` | Your Sophos Client Secret | Same place - copy before closing the dialog |

---

### Step 3 - Update the Deploy to AWS button URLs

In this README, replace every occurrence of `YOUR_BUCKET_NAME` with your
actual S3 bucket name from Step 1.

```bash
# Quick find and replace (run from repo root)
sed -i 's/YOUR_BUCKET_NAME/my-actual-bucket-name/g' README.md
```

---

### Step 4 - Push to main

```bash
git add .
git commit -m "Configure deployment"
git push origin main
```

GitHub Actions automatically:
1. Validates the CloudFormation template
2. Uploads `sophos-deregister.yaml` to your S3 bucket (publicly readable)
3. Deploys or updates the CloudFormation stack in your configured region

---

### Step 5 - Click Deploy to AWS

Click the button for your region above, fill in:
- `SophosClientId`
- `SophosClientSecret`

Leave all other parameters as defaults or customise as needed.
Tick **"I acknowledge that AWS CloudFormation might create IAM resources with custom names"**
then click **Create stack**.

---

## GitHub Actions workflows

| Workflow | File | Trigger | What it does |
|----------|------|---------|-------------|
| Deploy | `deploy.yml` | Push to `main` or manual | Validates template, uploads to S3, deploys/updates stack |
| Validate | `validate.yml` | Pull request to `main` | Validates template and checks for invalid characters |
| Destroy | `destroy.yml` | Manual only (must type DESTROY) | Deletes the stack and all its resources |

---

## Manual CLI deploy

```bash
aws cloudformation deploy \
  --template-file sophos-deregister.yaml \
  --stack-name sophos-deregister \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-south-1 \
  --parameter-overrides \
      SophosClientId=YOUR_CLIENT_ID \
      SophosClientSecret=YOUR_CLIENT_SECRET
```

---

## Tear down

```bash
aws cloudformation delete-stack \
  --stack-name sophos-deregister \
  --region YOUR_REGION
```

Or use the **Destroy** workflow in GitHub Actions (repo > Actions > Destroy Sophos Deregistration Stack > Run workflow).

---

## Repository structure

```
.
|-- README.md                         This file
|-- sophos-deregister.yaml            CloudFormation template (Python 3.14 Lambda embedded)
`-- .github/
    `-- workflows/
        |-- deploy.yml                Auto-deploy on push to main
        |-- validate.yml              Template validation on pull requests
        `-- destroy.yml               Manual stack teardown
```
