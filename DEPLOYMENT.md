# Deployment Guide - Azure SQL MCP Server

คู่มือการ Deploy อย่างละเอียดสำหรับ Azure SQL MCP Server

---

## 📋 Table of Contents

1. [Quick Deploy](#quick-deploy)
2. [Container Apps Deployment](#container-apps-deployment)
3. [App Service Deployment](#app-service-deployment)
4. [CI/CD Setup](#cicd-setup)
5. [Monitoring & Logging](#monitoring--logging)

---

## Quick Deploy

### Prerequisites Checklist

- [ ] ติดตั้ง Azure CLI แล้ว
- [ ] Login Azure สำเร็จ (`az login`)
- [ ] มี Azure Subscription ที่มีสิทธิ์สร้าง resources
- [ ] มี Connection String ของ SQL Database
- [ ] ติดตั้ง .NET 8 SDK แล้ว

---

## Container Apps Deployment

### Step 1: ตั้งค่า Environment Variables

```bash
# ตั้งค่า variables
export RESOURCE_GROUP="BMG-OpenAI"
export LOCATION="southeastasia"
export REGISTRY_NAME="bmgcontainer"
export CONTAINERAPP_NAME="sql-mcp-server"
export ENV_NAME="sql-mcp-env"

# Connection String (แก้ไขตามของคุณ)
export CONNECTION_STRING="Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
```

### Step 2: สร้าง Resource Group (ถ้ายังไม่มี)

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 3: สร้าง Container Registry

```bash
az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku Basic \
  --location $LOCATION \
  --admin-enabled true
```

### Step 4: Build และ Push Image

```bash
# Navigate to project directory
cd /path/to/az-sql-mcp-server

# Build image บน Azure (ไม่ต้องใช้ Docker local)
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .
```

### Step 5: สร้าง Container Apps Environment

```bash
az containerapp env create \
  --name $ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 6: Get ACR Credentials

```bash
export ACR_USERNAME=$(az acr credential show --name $REGISTRY_NAME --query username -o tsv)
export ACR_PASSWORD=$(az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv)
```

### Step 7: สร้าง Container App

```bash
az containerapp create \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $ENV_NAME \
  --image $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest \
  --registry-server $REGISTRY_NAME.azurecr.io \
  --registry-username $ACR_USERNAME \
  --registry-password $ACR_PASSWORD \
  --secrets mssql-connection-string="$CONNECTION_STRING" \
  --env-vars MSSQL_CONNECTION_STRING=secretref:mssql-connection-string \
  --target-port 5000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 3 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --tags environment=production project=mcp-server
```

### Step 8: ดู URL และทดสอบ

```bash
# ดู URL
export MCP_URL=$(az containerapp show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.configuration.ingress.fqdn" \
  --output tsv)

echo "Your MCP Server URL: https://$MCP_URL"

# ทดสอบ
curl -X POST https://$MCP_URL/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __typename }"}'
```

### Update Deployment (เมื่อมีการเปลี่ยนแปลง)

```bash
# 1. Build image ใหม่
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .

# 2. Update Container App
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --image $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest

# 3. ตรวจสอบสถานะ
az containerapp revision list \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[?properties.active==\`true\`].{Name:name, Status:properties.runningState, Health:properties.healthState}" \
  --output table
```

### Update Secret (เมื่อต้องการเปลี่ยน Connection String)

```bash
# Update secret
az containerapp secret set \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --secrets "mssql-connection-string=$NEW_CONNECTION_STRING"

# Restart to apply changes
az containerapp revision restart \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --revision $(az containerapp revision list --name $CONTAINERAPP_NAME --resource-group $RESOURCE_GROUP --query "[0].name" -o tsv)
```

---

## App Service Deployment

### Step 1: ตั้งค่า Environment Variables

```bash
# ตั้งค่า variables
export RESOURCE_GROUP="BMG-CUST-SAT-RG"
export LOCATION="southeastasia"
export PLAN_NAME="BMG-CUST-SAT-SERVICEPLAN"
export WEBAPP_NAME="sql-mcp-appservice"
export REGISTRY_NAME="bmgcontainer"

# Connection String
export CONNECTION_STRING="Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
```

### Step 2: สร้าง Resource Group (ถ้ายังไม่มี)

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 3: สร้าง App Service Plan

```bash
az appservice plan create \
  --name $PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --is-linux \
  --sku P2v2 \
  --location $LOCATION
```

**SKU Options:**
- `B1` - Basic (ราคาถูก สำหรับ dev/test)
- `P1v2` - Premium V2 (production)
- `P2v2` - Premium V2 (production, more resources)
- `S1` - Standard (balanced)

### Step 4: สร้าง Container Registry (ถ้ายังไม่มี)

```bash
az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku Basic \
  --location $LOCATION \
  --admin-enabled true
```

### Step 5: Build และ Push Image

```bash
cd /path/to/az-sql-mcp-server

az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .
```

### Step 6: สร้าง Web App

```bash
az webapp create \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan $PLAN_NAME \
  --deployment-container-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest
```

### Step 7: ตั้งค่า Container Registry

```bash
# Get ACR credentials
export ACR_USERNAME=$(az acr credential show --name $REGISTRY_NAME --query username -o tsv)
export ACR_PASSWORD=$(az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv)

# Configure container
az webapp config container set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-custom-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest \
  --docker-registry-server-url https://$REGISTRY_NAME.azurecr.io \
  --docker-registry-server-user $ACR_USERNAME \
  --docker-registry-server-password "$ACR_PASSWORD"
```

### Step 8: ตั้งค่า Environment Variables

```bash
az webapp config appsettings set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    WEBSITES_PORT=5000 \
    MSSQL_CONNECTION_STRING="$CONNECTION_STRING"
```

### Step 9: Enable Always On และ HTTP 2.0 (แนะนำสำหรับ production)

```bash
az webapp config set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --always-on true \
  --http20-enabled true
```

### Step 10: Restart และทดสอบ

```bash
# Restart
az webapp restart \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP

# Get URL
echo "Your App Service URL: https://$WEBAPP_NAME.azurewebsites.net"

# ทดสอบ
curl -X POST https://$WEBAPP_NAME.azurewebsites.net/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __typename }"}'
```

### Update Deployment (App Service)

```bash
# 1. Build image ใหม่
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .

# 2. Trigger webhook to update (หรือ manual restart)
az webapp restart \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP

# 3. ดู logs
az webapp log tail \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP
```

---

## CI/CD Setup

### GitHub Actions (Container Apps)

สร้างไฟล์ `.github/workflows/deploy-containerapp.yml`:

```yaml
name: Deploy to Azure Container Apps

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY_NAME: bmgcontainer
  CONTAINERAPP_NAME: sql-mcp-server
  RESOURCE_GROUP: BMG-OpenAI

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Build and push image to ACR
      run: |
        az acr build \
          --registry ${{ env.REGISTRY_NAME }} \
          --image sql-mcp-server:${{ github.sha }} \
          --image sql-mcp-server:latest \
          --file Dockerfile .

    - name: Deploy to Container App
      run: |
        az containerapp update \
          --name ${{ env.CONTAINERAPP_NAME }} \
          --resource-group ${{ env.RESOURCE_GROUP }} \
          --image ${{ env.REGISTRY_NAME }}.azurecr.io/sql-mcp-server:${{ github.sha }}
```

### GitHub Actions (App Service)

สร้างไฟล์ `.github/workflows/deploy-appservice.yml`:

```yaml
name: Deploy to Azure App Service

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY_NAME: bmgcontainer
  WEBAPP_NAME: sql-mcp-appservice
  RESOURCE_GROUP: BMG-CUST-SAT-RG

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Build and push image
      run: |
        az acr build \
          --registry ${{ env.REGISTRY_NAME }} \
          --image sql-mcp-server:${{ github.sha }} \
          --image sql-mcp-server:latest \
          --file Dockerfile .

    - name: Deploy to App Service
      run: |
        az webapp config container set \
          --name ${{ env.WEBAPP_NAME }} \
          --resource-group ${{ env.RESOURCE_GROUP }} \
          --docker-custom-image-name ${{ env.REGISTRY_NAME }}.azurecr.io/sql-mcp-server:${{ github.sha }}

        az webapp restart \
          --name ${{ env.WEBAPP_NAME }} \
          --resource-group ${{ env.RESOURCE_GROUP }}
```

### Setup GitHub Secrets

1. สร้าง Service Principal:

```bash
az ad sp create-for-rbac \
  --name "github-actions-mcp" \
  --role contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP \
  --sdk-auth
```

2. Copy output และเพิ่มเป็น secret ชื่อ `AZURE_CREDENTIALS` ใน GitHub

---

## Monitoring & Logging

### Container Apps Logs

```bash
# Real-time logs
az containerapp logs show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --follow

# Last 100 lines
az containerapp logs show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --tail 100

# Filter by timestamp
az containerapp logs show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --since 1h
```

### App Service Logs

```bash
# Enable logging
az webapp log config \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-container-logging filesystem

# Stream logs
az webapp log tail \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP

# Download logs
az webapp log download \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --log-file logs.zip
```

### Application Insights (แนะนำสำหรับ production)

```bash
# Create Application Insights
INSIGHTS_NAME="sql-mcp-insights"

az monitor app-insights component create \
  --app $INSIGHTS_NAME \
  --location $LOCATION \
  --resource-group $RESOURCE_GROUP

# Get Instrumentation Key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app $INSIGHTS_NAME \
  --resource-group $RESOURCE_GROUP \
  --query instrumentationKey -o tsv)

# Add to Container App
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=$INSTRUMENTATION_KEY"
```

### Health Check Endpoint

เพิ่ม health check ใน Container Apps:

```bash
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --health-probe-path "/graphql" \
  --health-probe-interval 30 \
  --health-probe-timeout 10
```

---

## Scaling Configuration

### Container Apps Auto-scaling

```bash
# Scale based on HTTP requests
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --min-replicas 1 \
  --max-replicas 10 \
  --scale-rule-name http-scaling \
  --scale-rule-type http \
  --scale-rule-http-concurrency 10
```

### App Service Auto-scaling

```bash
# Enable autoscale
az monitor autoscale create \
  --resource-group $RESOURCE_GROUP \
  --resource $WEBAPP_NAME \
  --resource-type Microsoft.Web/serverfarms \
  --name autoscale-$WEBAPP_NAME \
  --min-count 1 \
  --max-count 5 \
  --count 1

# Add CPU-based rule
az monitor autoscale rule create \
  --resource-group $RESOURCE_GROUP \
  --autoscale-name autoscale-$WEBAPP_NAME \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1
```

---

## Security Best Practices

### 1. Use Managed Identity (แนะนำสำหรับ production)

```bash
# Enable managed identity on Container App
az containerapp identity assign \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --system-assigned

# Get managed identity principal ID
PRINCIPAL_ID=$(az containerapp identity show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Grant SQL Database access (ต้องทำใน SQL Database)
# CREATE USER [sql-mcp-server] FROM EXTERNAL PROVIDER;
# ALTER ROLE db_datareader ADD MEMBER [sql-mcp-server];
```

### 2. Secure Secrets with Key Vault

```bash
# Create Key Vault
KEYVAULT_NAME="sql-mcp-kv"

az keyvault create \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Add secret
az keyvault secret set \
  --vault-name $KEYVAULT_NAME \
  --name mssql-connection-string \
  --value "$CONNECTION_STRING"

# Reference in Container App
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --secrets mssql-connection-string="keyvaultref:https://$KEYVAULT_NAME.vault.azure.net/secrets/mssql-connection-string,identityref:/subscriptions/YOUR_SUB/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/YOUR_IDENTITY"
```

### 3. Network Security

```bash
# Restrict Container App ingress to specific IPs
az containerapp ingress access-restriction set \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --rule-name allow-office \
  --ip-address 1.2.3.4/32 \
  --action Allow
```

---

## Rollback Strategy

### Container Apps Rollback

```bash
# List revisions
az containerapp revision list \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table

# Activate previous revision
az containerapp revision activate \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --revision PREVIOUS_REVISION_NAME
```

### App Service Deployment Slots (สำหรับ zero-downtime deployment)

```bash
# Create staging slot
az webapp deployment slot create \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging

# Deploy to staging
az webapp config container set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging \
  --docker-custom-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest

# Test staging
curl https://$WEBAPP_NAME-staging.azurewebsites.net/graphql

# Swap to production
az webapp deployment slot swap \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging
```

---

## Cost Optimization

### Container Apps

- **Consumption Plan**: Pay only for what you use
- **Scale to zero**: Set `--min-replicas 0` for dev/test environments
- **Right-sizing**: Monitor CPU/Memory usage and adjust

```bash
# Scale to zero for non-production
az containerapp update \
  --name $CONTAINERAPP_NAME-dev \
  --resource-group $RESOURCE_GROUP \
  --min-replicas 0 \
  --max-replicas 1
```

### App Service

- **Use App Service Plan efficiently**: Share plan across multiple apps
- **Auto-shutdown for dev/test**:

```bash
# Stop app during non-business hours
az webapp stop --name $WEBAPP_NAME-dev --resource-group $RESOURCE_GROUP
```

---

## Troubleshooting

### Common Deployment Issues

**1. Image Pull Failed**
```bash
# Check ACR credentials
az acr credential show --name $REGISTRY_NAME

# Update credentials
az containerapp registry set \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --server $REGISTRY_NAME.azurecr.io \
  --username $ACR_USERNAME \
  --password $ACR_PASSWORD
```

**2. Connection String Error**
```bash
# Verify secret
az containerapp secret list \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP

# Update secret
az containerapp secret set \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --secrets mssql-connection-string="NEW_CONNECTION_STRING"
```

**3. Port Configuration**
```bash
# Verify target port
az containerapp show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.configuration.ingress.targetPort"
```

---

## Quick Reference Commands

```bash
# Container Apps
az containerapp list --resource-group $RESOURCE_GROUP --output table
az containerapp show --name $CONTAINERAPP_NAME --resource-group $RESOURCE_GROUP
az containerapp logs show --name $CONTAINERAPP_NAME --resource-group $RESOURCE_GROUP --tail 50
az containerapp revision list --name $CONTAINERAPP_NAME --resource-group $RESOURCE_GROUP

# App Service
az webapp list --resource-group $RESOURCE_GROUP --output table
az webapp show --name $WEBAPP_NAME --resource-group $RESOURCE_GROUP
az webapp log tail --name $WEBAPP_NAME --resource-group $RESOURCE_GROUP
az webapp config show --name $WEBAPP_NAME --resource-group $RESOURCE_GROUP

# ACR
az acr repository list --name $REGISTRY_NAME --output table
az acr repository show-tags --name $REGISTRY_NAME --repository sql-mcp-server
```

---

## Support

สำหรับปัญหาหรือคำถาม:
- GitHub Issues: [Your repo URL]
- Email: your-email@example.com
- Documentation: [Your docs URL]

---

**Last Updated:** 2026-02-09
