
# Securely Building and Deploying Container Images in Azure with ACR and Key Vault using Bash Scripting

This project demonstrates how to automate the secure building and deployment of container images in Azure using Azure Container Registry (ACR) and Azure Key Vault, with the help of Bash scripting.
For a more detailed explanation of how this project was developed and deployed, you can refer to my [Medium blog post](https://medium.com/@venkatnarayanakuruva/automating-container-image-management-with-azure-container-registry-using-bash-scripting-and-key-dd1ce356ffdd).

## Table of Contents
- [Prerequisites](#prerequisites)
- [Steps](#steps)
  - [Deploy an Azure Container Registry](#deploy-an-azure-container-registry)
  - [Build Container Images](#build-container-images)
  - [Deploy Images from Azure Container Registry](#deploy-images-from-azure-container-registry)
  - [Clean Up Resources](#clean-up-resources)
- [Verification](#verification)
- [Accessing the Application](#accessing-the-application)
- [Repository](#repository)
- [Clean Up](#clean-up)

---

## Prerequisites
1. An active [Azure account](https://azure.microsoft.com/free/).
2. Azure CLI installed locally or use the Azure Cloud Shell.
3. Access to a [GitHub repository](https://github.com/Venkat-Narayan-07/acr-build-helloworld-node).

---

## Steps

### 1. Deploy an Azure Container Registry
1. **Clone the GitHub Repository:**
   ```bash
   git clone https://github.com/Venkat-Narayan-07/acr-build-helloworld-node.git
   ```
   ![image](https://github.com/user-attachments/assets/bc24ade0-0313-4d0c-a0e0-3f9565f88d41)

2. **Navigate to the Project Directory:**
   ```bash
   cd acr-build-helloworld-node
   ```
   ![image](https://github.com/user-attachments/assets/0e7645ce-d756-481c-b7fe-12931e9d94dd)

3. **Set Up Variables:**
   ```bash
   ACR_NAME=<acr-name>
   RES_GROUP=<resource-group-name>
   AKV_NAME=<azure-key-vault-name>
   ```
   ![image](https://github.com/user-attachments/assets/d5f21930-e35a-4b4c-9bd7-63d038eb33ae)

4. **Create a Resource Group and Azure Container Registry:**
   ```bash
   az group create --resource-group $RES_GROUP --location eastus
   az acr create --resource-group $RES_GROUP --name $ACR_NAME --sku Standard --location eastus
   ```
![image](https://github.com/user-attachments/assets/a3cca6e9-b5d3-4651-a09f-7ee948a3594d)

---

### 2. Build Container Images
Build a container image using Azure Container Registry tasks:
```bash
az acr build --registry $ACR_NAME --image helloacrtasks:v1 --file Dockerfile
```
![image](https://github.com/user-attachments/assets/76cdd242-8216-49dc-b716-fb7ef2b0bbe2)
![image](https://github.com/user-attachments/assets/4c9fd173-4c77-4b0a-be4f-e38839c93355)
Verify the image in Aaure portal
![image](https://github.com/user-attachments/assets/1f48a244-23c8-40ec-a422-0e2611c3d31e)

---

### 3. Deploy Images from Azure Container Registry

#### 3.1 create Azure Key vault
```bash
export RES_GROUP=<resource-group-name>
az keyvault create --resource-group $RES_GROUP --name $AKV_NAME
```
![image](https://github.com/user-attachments/assets/2eb37f97-88b1-4e7c-91bf-8b5520b89995)

#### 3.2 Registry Authentication
Azure Container Registry requires authentication:
- Use Microsoft Entra identities for role-based access.

#### 3.3 Generate Service Principals
1. Store the service principal ID in Azure Key Vault:
   ```bash
   az keyvault secret set --vault-name $AKV_NAME --name $ACR_NAME-pull-usr --value $(az sp list --display-name $ACR_NAME-pull --query [].appId --output tsv)
   ```
   ![image](https://github.com/user-attachments/assets/1cb46a65-f140-4390-838a-b0499d3c3237)

2. Store the service principal password:
   ```bash
   az keyvault secret set --vault-name $AKV_NAME --name $ACR_NAME-pull-pwd --value $(az ad sp create-for-rbac --name $ACR_NAME-pull --scopes $(az acr show --name $ACR_NAME --query id --output tsv) --query password --output tsv)
   ```
![image](https://github.com/user-attachments/assets/e69633c9-48b3-49c1-982d-bd386c19cc9e

#### 3.3 Deploy the Container
Deploy a container from the registry:
```bash
az container create \
  --resource-group $RES_GROUP \
  --name acr-tasks \
  --image $ACR_NAME.azureacr.io/helloacrtasks:v1 \
  --registry-login-server $ACR_NAME.azureacr.io \
  --registry-username $(az keyvault secret show --vault-name $AKV_NAME --name $ACR_NAME-pull-usr --query value -o tsv) \
  --registry-password $(az keyvault secret show --vault-name $AKV_NAME --name $ACR_NAME-pull-pwd --query value -o tsv) \
  --dns-name-label acr-tasks-$ACR_NAME \
  --query "{FQDN:ipAddress.fqdn}" --output table
```
![image](https://github.com/user-attachments/assets/acbaaafd-005a-4e9f-b08e-9e17720d9526)

---

## Verification
Verify the container's status:
```bash
az container attach --resource-group $RES_GROUP --name acr-tasks
```
![image](https://github.com/user-attachments/assets/033fdbe8-cbbf-4ae7-ab0c-ef86c42b88b5)

---

## Accessing the Application
Access the application by navigating to the Fully Qualified Domain Name (FQDN) of the container, as provided in the deployment output.
![image](https://github.com/user-attachments/assets/acf89213-e445-4cf8-8922-95b0e24a5523)

---

## Clean Up Resources
To delete all resources:
1. Delete the container:
   ```bash
   az container delete --resource-group $RES_GROUP --name acr-tasks
   ```
2. Optionally, remove the resource group:
   ```bash
   az group delete --name $RES_GROUP
   ```
   ![image](https://github.com/user-attachments/assets/6231a793-b856-4f9e-978c-4501350ac2a6)



---

## Repository
This project can be found in the GitHub repository:  
[https://github.com/Azure-Samples/acr-build-helloworld-node](https://github.com/Azure-Samples/acr-build-helloworld-node)

---
## Medium Blog Link
[Medium blog post](https://medium.com/@venkatnarayanakuruva/automating-container-image-management-with-azure-container-registry-using-bash-scripting-and-key-dd1ce356ffdd).
## Notes
- Replace placeholders (e.g., `<acr-name>`  `<azure-key-vault-name>` and `<resource-group-name>`) with your actual resource names.
  
