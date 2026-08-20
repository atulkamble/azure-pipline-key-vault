# Azure Key Vault + Azure DevOps Pipeline

## 1. Cryptography Basics

Cryptography is used to protect information so that unauthorized users cannot understand or access it.

### Encryption

Encryption converts readable information into an unreadable format.

```text
Plain Text
    ↓
Encryption
    ↓
Cipher Text
```

Example:

```text
Password123
    ↓ Encryption
X7@92#AB...
```

### Decryption

Decryption converts encrypted information back into its original readable form.

```text
Cipher Text
    ↓
Decryption
    ↓
Plain Text
```

### Simple Flow

```text
Plain Text → Encryption → Cipher Text
Cipher Text → Decryption → Plain Text
```

---

# 2. Why Secrets Need Protection

Applications frequently require sensitive information such as:

* Database usernames
* Database passwords
* API keys
* Access tokens
* Connection strings
* Certificates
* SSH keys
* Encryption keys

These values should **not be stored directly inside source code**.

Bad example:

```python
db_password = "Pass@123"
```

If this code is pushed to Git:

```text
Code
 ↓
Git
 ↓
GitHub / Azure Repos
```

the password can become exposed through version-control history.

---

# 3. `.env` Files

Developers sometimes store configuration in a `.env` file.

Example:

```env
DB_USERNAME=admin
DB_PASSWORD=example-password
```

The application can then load these values at runtime.

However, `.env` files containing secrets should normally be excluded from Git.

Example `.gitignore`:

```gitignore
.env
```

For production environments, a managed secret store such as **Azure Key Vault** is preferable.

---

# 4. Website Certificates

Websites use certificates to establish encrypted HTTPS communication.

Example:

```text
Browser
   ↓
HTTPS / TLS Certificate
   ↓
Web Server
```

Certificates can also be stored and managed in Azure Key Vault.

Azure Key Vault can manage:

```text
Secrets
Keys
Certificates
```

---

# 5. What is Azure Key Vault?

Azure Key Vault is a Microsoft Azure service used to securely store and manage sensitive information.

Typical examples include:

```text
Credentials
Username / Password
API Keys
Tokens
Database Passwords
Connection Strings
Certificates
Encryption Keys
```

Instead of putting a password directly in application code:

```text
Application
    ↓
Azure Key Vault
    ↓
Secret
```

The application or pipeline authenticates with Azure and retrieves the secret securely.

---

# 6. Azure Key Vault Components

| Component          | Purpose                                                |
| ------------------ | ------------------------------------------------------ |
| Secrets            | Passwords, tokens, connection strings                  |
| Keys               | Cryptographic keys                                     |
| Certificates       | SSL/TLS certificates                                   |
| IAM                | Controls access through Azure RBAC                     |
| Microsoft Entra ID | Provides identities for users, groups and applications |

---

# 7. Key Vault Lab Architecture

For this lab:

```text
Microsoft Entra ID
        │
        ├── User
        │
        ├── Group
        │
        └── Service Principal
                │
                ▼
        Azure RBAC
                │
                ▼
       Azure Key Vault
       myvalut68698
                │
                ▼
          Secret: dbpass
                │
                ▼
        Azure DevOps Pipeline
```

---

# 8. Key Vault Creation

## Step 1 — Configure Microsoft Entra ID

Open:

```text
Azure Portal
↓
Microsoft Entra ID
```

Create or use a group such as:

```text
admin
```

Add the required user to the group.

Conceptually:

```text
User
 ↓
Admin Group
 ↓
Azure Resources
```

---

# 9. Create the Key Vault

Create a new Azure Key Vault.

For this lab:

```text
Key Vault Name: myvalut68698
Resource Group: myrg
```

Example Azure CLI command:

```bash
az keyvault create \
  --name myvalut68698 \
  --resource-group myrg \
  --location eastus \
  --enable-rbac-authorization true
```

---

# 10. Assign Key Vault Administrator Permissions

Open:

```text
Azure Portal
↓
Key Vault
↓
myvalut68698
↓
Access Control (IAM)
↓
Add Role Assignment
```

Select:

```text
Key Vault Secrets Officer
```

Then:

```text
Select User
↓
Review + Assign
```

### Key Vault Secrets Officer

This role allows a user to manage Key Vault secrets.

It can typically:

```text
Create secrets
Read secrets
Update secrets
Delete secrets
```

---

# 11. Create a Secret

Open:

```text
Key Vault
↓
Objects
↓
Secrets
↓
Generate / Import
```

Example:

```text
Secret Name: dbpass
Secret Value: <your-demo-password>
```

Do not put a real production password in documentation or source control.

CLI equivalent:

```bash
az keyvault secret set \
  --vault-name myvalut68698 \
  --name dbpass \
  --value "<your-demo-password>"
```

---

# 12. Verify the Secret

List secrets:

```bash
az keyvault secret list \
  --vault-name myvalut68698 \
  -o table
```

Retrieve the secret:

```bash
az keyvault secret show \
  --vault-name myvalut68698 \
  --name dbpass
```

Retrieve only the value:

```bash
az keyvault secret show \
  --vault-name myvalut68698 \
  --name dbpass \
  --query value \
  -o tsv
```

Be careful when running the last command because the secret value is printed directly in the terminal.

---

# 13. Azure DevOps Pipeline Architecture

The goal is to allow an Azure DevOps pipeline to securely retrieve a secret from Azure Key Vault.

Architecture:

```text
GitHub Repository
       │
       ▼
azure-pipelines.yml
       │
       ▼
Azure DevOps Pipeline
       │
       ▼
Azure Service Connection
       │
       ▼
Service Principal
       │
       ▼
Azure RBAC
       │
       ▼
Azure Key Vault
       │
       ▼
dbpass
```

---

# 14. Create GitHub Repository

Repository:

```text
https://github.com/atulkamble/azure-pipline-key-vault
```

Recommended project structure:

```text
azure-pipline-key-vault/
│
├── azure-pipelines.yml
├── README.md
└── .gitignore
```

Clone the repository:

```bash
git clone https://github.com/atulkamble/azure-pipline-key-vault.git
```

Enter the repository:

```bash
cd azure-pipline-key-vault
```

Open it in VS Code:

```bash
code .
```

---

# 15. Create `azure-pipelines.yml`

Create:

```text
azure-pipelines.yml
```

Example pipeline:

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  keyvault: 'myvalut68698'
  secretName: 'dbpass'

steps:

- task: AzureKeyVault@2
  displayName: 'Fetch Azure Key Vault Secrets'
  inputs:
    azureSubscription: 'keyvaultconnection'
    KeyVaultName: '$(keyvault)'
    SecretsFilter: '$(secretName)'
    RunAsPreJob: false

- script: |
    echo "Secret successfully retrieved from Azure Key Vault."
  displayName: 'Verify Secret Retrieval'
```

Azure DevOps automatically exposes the retrieved secret using its secret name.

For example:

```text
$(dbpass)
```

should be treated as a secret pipeline variable.

Do not intentionally print its actual value into pipeline logs.

---

# 16. Create Azure Service Connection

In Azure DevOps:

```text
Project
↓
Project Settings
↓
Service Connections
↓
New Service Connection
↓
Azure Resource Manager
```

Create a connection such as:

```text
keyvaultconnection
```

The service connection normally uses a Microsoft Entra identity such as a:

```text
Service Principal
```

The service principal then needs permission to read Key Vault secrets.

---

# 17. Important Information Required

Before assigning Key Vault permissions, identify:

```text
Key Vault Name
Subscription ID
Resource Group
Key Vault Resource ID
Service Principal Object ID
Service Principal Client ID
```

For this lab:

```text
Key Vault Name:
myvalut68698

Resource Group:
myrg

Subscription ID:
08b7b8d4-af42-4972-9517-11ea256ea068
```

---

# 18. Print Current Subscription ID

Run:

```bash
az account show \
  --query id \
  -o tsv
```

Or:

```bash
az account show \
  --query "{Subscription:name,SubscriptionID:id}" \
  -o table
```

---

# 19. Print Key Vault Information

Run:

```bash
az keyvault show \
  --name myvalut68698 \
  --resource-group myrg \
  -o table
```

---

# 20. Print Key Vault Resource ID

Use:

```bash
az keyvault show \
  --name myvalut68698 \
  --resource-group myrg \
  --query id \
  -o tsv
```

Expected format:

```text
/subscriptions/<subscription-id>/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvalut68698
```

For this environment:

```text
/subscriptions/08b7b8d4-af42-4972-9517-11ea256ea068/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvalut68698
```

This value becomes the Azure RBAC `--scope`.

---

# 21. Key Vault Secret URL

A Key Vault secret URL follows this pattern:

```text
https://<keyvault-name>.vault.azure.net/secrets/<secret-name>/<version>
```

Example:

```text
https://myvalut68698.vault.azure.net/secrets/dbpass/<secret-version>
```

Avoid hardcoding a secret version unless a specific immutable version is required.

Applications typically use:

```text
https://myvalut68698.vault.azure.net/secrets/dbpass
```

to access the current version through the Key Vault API or SDK.

---

# 22. Find the Service Principal

A service principal has several identifiers.

Important distinction:

```text
Client ID / Application ID
        ≠
Object ID
```

The **Object ID** is normally used with:

```bash
--assignee-object-id
```

---

# 23. Print Service Principal Details

If you know the service principal identifier:

```bash
az ad sp show \
  --id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --query "{Name:displayName,ObjectID:id,ClientID:appId}" \
  -o table
```

Output format:

```text
Name                  ObjectID                                ClientID
--------------------  --------------------------------------  --------------------------------------
service-principal     <object-id>                             <client-id>
```

---

# 24. Print Only the Service Principal Object ID

Use:

```bash
az ad sp show \
  --id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --query id \
  -o tsv
```

---

# 25. Print Only the Client ID

Use:

```bash
az ad sp show \
  --id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --query appId \
  -o tsv
```

Remember:

```text
id    = Object ID
appId = Client ID
```

---

# 26. Assign Key Vault Permission to the Service Principal

The Azure DevOps service principal needs permission to retrieve secrets.

Recommended RBAC role:

```text
Key Vault Secrets User
```

This role allows the identity to read secret contents without giving it unnecessary secret-management permissions.

Run:

```bash
az role assignment create \
  --assignee-object-id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/08b7b8d4-af42-4972-9517-11ea256ea068/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvalut68698
```

---

# 27. RBAC Permission Flow

The permission flow is:

```text
Azure DevOps Pipeline
        │
        ▼
Service Connection
        │
        ▼
Service Principal
        │
        │ Key Vault Secrets User
        ▼
Azure Key Vault
        │
        ▼
dbpass
```

The service principal therefore receives permission only at the Key Vault scope.

---

# 28. Verify the Role Assignment

Run:

```bash
az role assignment list \
  --assignee-object-id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --scope /subscriptions/08b7b8d4-af42-4972-9517-11ea256ea068/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvalut68698 \
  -o table
```

You should see:

```text
Key Vault Secrets User
```

associated with the service principal.

---

# 29. Easier Way to Store the Key Vault Scope

Instead of repeatedly typing the entire resource ID:

```bash
KV_SCOPE=$(az keyvault show \
  --name myvalut68698 \
  --resource-group myrg \
  --query id \
  -o tsv)
```

Check it:

```bash
echo "$KV_SCOPE"
```

Then assign the role:

```bash
az role assignment create \
  --assignee-object-id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope "$KV_SCOPE"
```

Verify:

```bash
az role assignment list \
  --assignee-object-id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --scope "$KV_SCOPE" \
  -o table
```

---

# 30. Useful Command — Automatically Get Subscription ID

Instead of hardcoding:

```text
08b7b8d4-af42-4972-9517-11ea256ea068
```

use:

```bash
SUBSCRIPTION_ID=$(az account show \
  --query id \
  -o tsv)
```

Verify:

```bash
echo "$SUBSCRIPTION_ID"
```

---

# 31. Useful Command — Automatically Get Key Vault Scope

```bash
KV_SCOPE=$(az keyvault show \
  --name myvalut68698 \
  --resource-group myrg \
  --query id \
  -o tsv)
```

---

# 32. Useful Command — Automatically Get Service Principal Object ID

If you know the Client ID:

```bash
SP_OBJECT_ID=$(az ad sp show \
  --id <client-id> \
  --query id \
  -o tsv)
```

Verify:

```bash
echo "$SP_OBJECT_ID"
```

---

# 33. Complete Automated RBAC Example

```bash
KEYVAULT_NAME="myvalut68698"
RESOURCE_GROUP="myrg"
SP_CLIENT_ID="<service-principal-client-id>"

SUBSCRIPTION_ID=$(az account show \
  --query id \
  -o tsv)

KV_SCOPE=$(az keyvault show \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query id \
  -o tsv)

SP_OBJECT_ID=$(az ad sp show \
  --id "$SP_CLIENT_ID" \
  --query id \
  -o tsv)

echo "Subscription ID : $SUBSCRIPTION_ID"
echo "Key Vault Scope : $KV_SCOPE"
echo "SP Object ID    : $SP_OBJECT_ID"

az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope "$KV_SCOPE"
```

This is easier and safer than manually copying long IDs.

---

# 34. Verify Role Assignment Automatically

```bash
az role assignment list \
  --assignee-object-id "$SP_OBJECT_ID" \
  --scope "$KV_SCOPE" \
  -o table
```

---

# 35. Build and Run the Azure Pipeline

After completing the Key Vault permissions:

```text
Azure DevOps
↓
Pipelines
↓
Create / Select Pipeline
↓
GitHub Repository
↓
azure-pipelines.yml
↓
Run Pipeline
```

The pipeline executes:

```text
Pipeline Starts
     ↓
AzureKeyVault@2
     ↓
Azure Service Connection
     ↓
Service Principal Authentication
     ↓
Azure RBAC Check
     ↓
Key Vault
     ↓
Fetch dbpass
     ↓
Secret available to pipeline
```

---

# 36. Example Final Pipeline

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  keyvault: 'myvalut68698'
  secretName: 'dbpass'

steps:

- task: AzureKeyVault@2
  displayName: 'Fetch Key Vault Secret'
  inputs:
    azureSubscription: 'keyvaultconnection'
    KeyVaultName: '$(keyvault)'
    SecretsFilter: '$(secretName)'
    RunAsPreJob: false

- script: |
    echo "Key Vault connection successful."
    echo "Secret has been retrieved securely."
  displayName: 'Verify Key Vault Access'
```

---

# 37. Using the Secret in an Application

Suppose the secret name is:

```text
dbpass
```

The Azure Key Vault task exposes it as:

```text
$(dbpass)
```

For example:

```yaml
- script: |
    python app.py
  displayName: 'Run Application'
  env:
    DB_PASSWORD: $(dbpass)
```

Inside Python:

```python
import os

password = os.getenv("DB_PASSWORD")

print("Database password was loaded.")
```

Do **not** use:

```python
print(password)
```

in production because that can expose the secret in logs.

---

# 38. Security Best Practices

Follow these practices:

| Practice                                       | Recommendation             |
| ---------------------------------------------- | -------------------------- |
| Password in code                               | ❌ Avoid                    |
| Password committed to Git                      | ❌ Avoid                    |
| Production password in `.env` committed to Git | ❌ Avoid                    |
| Password in Key Vault                          | ✅ Recommended              |
| RBAC                                           | ✅ Use least privilege      |
| Service Principal                              | ✅ Give only required roles |
| Print secrets in logs                          | ❌ Avoid                    |
| Secret rotation                                | ✅ Recommended              |
| Managed Identity                               | ✅ Prefer where supported   |
| HTTPS/TLS certificates                         | ✅ Securely manage them     |

---

# 39. Least Privilege Principle

Always provide only the permissions required for the task.

Example:

For the developer who manages secrets:

```text
Key Vault Secrets Officer
```

For the Azure DevOps pipeline that only reads secrets:

```text
Key Vault Secrets User
```

Avoid assigning:

```text
Owner
Contributor
```

unless those permissions are actually required.

---

# 40. Key Vault Roles

| Role                           | Purpose                                       |
| ------------------------------ | --------------------------------------------- |
| Key Vault Administrator        | Full Key Vault data-plane administration      |
| Key Vault Secrets Officer      | Manage secrets                                |
| Key Vault Secrets User         | Read secret values                            |
| Key Vault Crypto Officer       | Manage cryptographic keys                     |
| Key Vault Certificates Officer | Manage certificates                           |
| Key Vault Reader               | Read metadata but generally not secret values |

For this pipeline:

```text
Key Vault Secrets User
```

is usually sufficient.

---

# 41. Troubleshooting

## Error: Forbidden

Example:

```text
Caller is not authorized to perform action on resource
```

Check:

```bash
az role assignment list \
  --assignee-object-id "$SP_OBJECT_ID" \
  --scope "$KV_SCOPE" \
  -o table
```

Verify the service principal has:

```text
Key Vault Secrets User
```

---

## Error: Secret Not Found

Check available secrets:

```bash
az keyvault secret list \
  --vault-name myvalut68698 \
  -o table
```

Check the exact name:

```text
dbpass
```

Secret names are important because the Azure Pipeline task creates variables based on those names.

---

## Error: Wrong Service Principal

Check:

```bash
az ad sp show \
  --id <client-id> \
  --query "{Name:displayName,ObjectID:id,ClientID:appId}" \
  -o table
```

Remember:

```text
Object ID → used for RBAC assignment
Client ID → identifies the application/service principal
```

---

## Error: Wrong Subscription

Check:

```bash
az account show -o table
```

List subscriptions:

```bash
az account list -o table
```

Select the required subscription:

```bash
az account set \
  --subscription 08b7b8d4-af42-4972-9517-11ea256ea068
```

---

# 42. Complete Lab Workflow

```text
1. Microsoft Entra ID
       ↓
2. Create/Admin User or Group
       ↓
3. Create Azure Key Vault
       ↓
4. Assign Key Vault Secrets Officer to User
       ↓
5. Create dbpass Secret
       ↓
6. Create GitHub Repository
       ↓
7. Create azure-pipelines.yml
       ↓
8. Create Azure DevOps Service Connection
       ↓
9. Identify Service Principal
       ↓
10. Find Service Principal Object ID
       ↓
11. Find Key Vault Resource ID
       ↓
12. Assign Key Vault Secrets User Role
       ↓
13. Verify RBAC Assignment
       ↓
14. Run Azure Pipeline
       ↓
15. AzureKeyVault@2 Retrieves Secret
       ↓
16. Application Uses Secret Securely
```

---

# 43. Important Commands Cheat Sheet

### Subscription ID

```bash
az account show \
  --query id \
  -o tsv
```

### Key Vault Details

```bash
az keyvault show \
  --name myvalut68698 \
  --resource-group myrg
```

### Key Vault Resource ID

```bash
az keyvault show \
  --name myvalut68698 \
  --resource-group myrg \
  --query id \
  -o tsv
```

### Service Principal Details

```bash
az ad sp show \
  --id <client-id-or-object-id> \
  --query "{Name:displayName,ObjectID:id,ClientID:appId}" \
  -o table
```

### Service Principal Object ID

```bash
az ad sp show \
  --id <client-id> \
  --query id \
  -o tsv
```

### Assign Key Vault Role

```bash
az role assignment create \
  --assignee-object-id <service-principal-object-id> \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope <key-vault-resource-id>
```

### Verify Role

```bash
az role assignment list \
  --assignee-object-id <service-principal-object-id> \
  --scope <key-vault-resource-id> \
  -o table
```

### List Secrets

```bash
az keyvault secret list \
  --vault-name myvalut68698 \
  -o table
```

---

# 44. Final Architecture

```text
                    Microsoft Entra ID
                           │
           ┌───────────────┴───────────────┐
           │                               │
          User                     Service Principal
           │                               │
Key Vault Secrets Officer        Key Vault Secrets User
           │                               │
           └───────────────┬───────────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │ Azure Key Vault   │
                 │ myvalut68698      │
                 │                   │
                 │ Secret: dbpass    │
                 └─────────┬─────────┘
                           ▲
                           │
                   AzureKeyVault@2
                           │
                           │
                 ┌─────────┴─────────┐
                 │ Azure DevOps      │
                 │ Pipeline          │
                 └─────────┬─────────┘
                           │
                           ▼
                  Application / Build
```

---

# 45. Key Learning Points

* **Plain text → encryption → cipher text**
* **Cipher text → decryption → plain text**
* Never hardcode credentials in source code.
* Do not commit `.env` files containing secrets to Git.
* Azure Key Vault securely stores secrets, keys, and certificates.
* Microsoft Entra ID provides identities.
* Azure RBAC determines what an identity can access.
* A developer who manages secrets may use **Key Vault Secrets Officer**.
* An Azure DevOps pipeline that only reads secrets should normally use **Key Vault Secrets User**.
* `id` from `az ad sp show` is the **Object ID**.
* `appId` is the **Client ID**.
* The Azure DevOps service connection authenticates the pipeline to Azure.
* `AzureKeyVault@2` retrieves secrets into the Azure Pipeline.
* Never print production secrets into pipeline logs.

## Final Concept

```text
Do not store secrets in code.

Code
  │
  ├── Git / GitHub
  │
  └── Azure DevOps Pipeline
             │
             ▼
       Service Connection
             │
             ▼
      Service Principal
             │
             ▼
          Azure RBAC
             │
             ▼
       Azure Key Vault
             │
             ▼
         Secret / Key
```
