# Sophos Central EC2 Auto-Deregistration

Automatically removes Sophos Central endpoint licences when AWS EC2 instances
are terminated. No manual cleanup needed.
Please Note: Instances terminated before running the code do not automatically remove the Sophos Central endpoint.
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

a)Sophos Client ID : Paste the Sophos Central API key ID.

b)Sophos Client Secret: Paste the Sophos Central API client secret.

c)Secret Manager-Secret Name : Change the name as the default one most likely exists in Sophos NSG IAAS playground.

d)Lambda - Function Name:Provide suitable name/leave the default one.

e)EventBridge - Rule Name:Provide suitable name/leave the default one.

f)EventBridge - Rule State on Deploy : ENABLED.

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


