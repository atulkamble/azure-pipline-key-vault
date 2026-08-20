```
cryptography - 

plain text >> cipher text - encryption 
cipher text >> plain text >> decryption 

azure key vault 

- credentials 
- username, password 

.env 

website - certificates

code >> git (version control)

// Key Vault Creation

1. Microsoft Entra ID 
group - admin 
user >> admin 

2. Create key vault - myvalut68698 
3. Select key vault - IAM >> 
Add Role >> Key Vault Secrets Officer >> Select user >> Apply 

4. secrets >>
dbpass
Pass@123

// Create Azure Pipeline for Azure KeyVault Secrets 

1. create repo https://github.com/atulkamble/azure-pipline-key-vault
2. create azure-pipelines.yml
3. clone repo >> open VS Code 
4. create pipeline code >> Build >> Check output
5. note down keyvault name, subscription id, objectid 
6. print key vault


az keyvault show \
  --name myvalut68698 \
  --resource-group myrg \
  --query id \
  -o tsv

https://myvalut68698.vault.azure.net/secrets/dbpass/21d20c26d7864bb584c4e638ca415f17

paste subscription id - 08b7b8d4-af42-4972-9517-11ea256ea068
keyvault name - myvalut68698

az ad sp show \
  --id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --query "{Name:displayName,ObjectID:id,ClientID:appId}" \
  -o table

7. create role assignemnt*

az role assignment create \
  --assignee-object-id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/08b7b8d4-af42-4972-9517-11ea256ea068/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvalut68698

8. verify role assignemnt 

az role assignment list \
  --assignee-object-id bc15a766-7fbd-4b49-8969-95208bd8b5e9 \
  --scope /subscriptions/08b7b8d4-af42-4972-9517-11ea256ea068/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvalut68698 \
  -o table

9. build and run pipeline 


