# Azure SQL MCP Server

Data API Builder (DAB) MCP Server สำหรับเข้าถึงข้อมูล Azure SQL Database ผ่าน REST API, GraphQL และ MCP Protocol

## 📋 สารบัญ

- [Prerequisites](#prerequisites)
- [โครงสร้างโปรเจค](#โครงสร้างโปรเจค)
- [การตั้งค่าเบื้องต้น](#การตั้งค่าเบื้องต้น)
- [การรันแบบ Local](#การรันแบบ-local)
- [การ Deploy ไปยัง Azure](#การ-deploy-ไปยัง-azure)
  - [Deploy to Container Apps](#1-deploy-to-azure-container-apps)
  - [Deploy to App Service](#2-deploy-to-azure-app-service)
- [การทดสอบ](#การทดสอบ)
- [ตัวอย่าง Query](#ตัวอย่าง-query)

---

## Prerequisites

### ติดตั้ง .NET SDK

```bash
# macOS (Homebrew)
brew install dotnet@8
brew install dotnet@9

# เพิ่ม .NET ใน PATH
echo 'export PATH="/opt/homebrew/opt/dotnet@8/bin:$PATH"' >> ~/.zshrc
echo 'export PATH="/opt/homebrew/opt/dotnet@9/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### ติดตั้ง Azure CLI

```bash
# macOS
brew install azure-cli

# เข้าสู่ระบบ Azure
az login
```

### ติดตั้ง Docker Desktop (Optional)

ดาวน์โหลดจาก: https://www.docker.com/products/docker-desktop/

---

## โครงสร้างโปรเจค

```
az-sql-mcp-server/
├── dab-config.json         # Data API Builder configuration
├── Dockerfile              # Docker image definition
├── .env                    # Environment variables (ห้าม commit!)
├── .config/
│   └── dotnet-tools.json   # .NET local tools manifest
└── README.md               # เอกสารนี้
```

---

## การตั้งค่าเบื้องต้น

### 1. Clone/Setup โปรเจค

```bash
cd az-sql-mcp-server
```

### 2. สร้าง .NET Tool Manifest

```bash
dotnet new tool-manifest
```

### 3. ติดตั้ง Data API Builder

```bash
dotnet tool restore
```

### 4. สร้างไฟล์ .env

สร้างไฟล์ `.env` ในโฟลเดอร์ root:

```env
MSSQL_CONNECTION_STRING=Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;Persist Security Info=False;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
```

**⚠️ สำคัญ:** อย่าลืมเพิ่ม `.env` เข้าไปใน `.gitignore`

---

## การรันแบบ Local

### เริ่ม DAB Server

```bash
dotnet dab start --config dab-config.json
```

Server จะทำงานที่ `http://localhost:5000`

### Endpoints ที่ใช้ได้

- **REST API:** http://localhost:5000/api
- **GraphQL:** http://localhost:5000/graphql
- **MCP:** http://localhost:5000/mcp

### ทดสอบ Local

```bash
# ทดสอบ GraphQL
curl -X POST http://localhost:5000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ userInfos(first: 5) { items { DisplayName Mail } } }"}'
```

---

## การ Deploy ไปยัง Azure

### 1. Deploy to Azure Container Apps

#### 1.1 สร้าง Container Registry (ถ้ายังไม่มี)

```bash
RESOURCE_GROUP="BMG-OpenAI"
REGISTRY_NAME="bmgcontainer"

az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku Basic \
  --location "Southeast Asia"
```

#### 1.2 Build และ Push Image

```bash
# Build image บน Azure (ไม่ต้องใช้ Docker local)
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .
```

#### 1.3 สร้าง Container Apps Environment (ถ้ายังไม่มี)

```bash
ENV_NAME="sql-mcp-env"

az containerapp env create \
  --name $ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location "Southeast Asia"
```

#### 1.4 สร้าง Container App

```bash
CONTAINERAPP_NAME="sql-mcp-server"
CONNECTION_STRING="Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"

# Get ACR credentials
ACR_USERNAME=$(az acr credential show --name $REGISTRY_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv)

# Create Container App
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
  --memory 1.0Gi
```

#### 1.5 ดู URL ของ Container App

```bash
az containerapp show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.configuration.ingress.fqdn" \
  --output tsv
```

#### 1.6 Update เมื่อมีการเปลี่ยนแปลง

```bash
# Build image ใหม่
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .

# Update Container App
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --image $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest
```

---

### 2. Deploy to Azure App Service

#### 2.1 สร้าง App Service Plan (ถ้ายังไม่มี)

```bash
RESOURCE_GROUP="BMG-CUST-SAT-RG"
PLAN_NAME="BMG-CUST-SAT-SERVICEPLAN"

az appservice plan create \
  --name $PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --is-linux \
  --sku P2v2 \
  --location "Southeast Asia"
```

#### 2.2 Build และ Push Image

```bash
REGISTRY_NAME="bmgcontainer"

az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .
```

#### 2.3 สร้าง Web App

```bash
WEBAPP_NAME="sql-mcp-appservice"

az webapp create \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan $PLAN_NAME \
  --deployment-container-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest
```

#### 2.4 ตั้งค่า Container Registry

```bash
# Get ACR credentials
ACR_USERNAME=$(az acr credential show --name $REGISTRY_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv)

# Configure container
az webapp config container set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-custom-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest \
  --docker-registry-server-url https://$REGISTRY_NAME.azurecr.io \
  --docker-registry-server-user $ACR_USERNAME \
  --docker-registry-server-password "$ACR_PASSWORD"
```

#### 2.5 ตั้งค่า Environment Variables

```bash
CONNECTION_STRING="Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"

az webapp config appsettings set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    WEBSITES_PORT=5000 \
    MSSQL_CONNECTION_STRING="$CONNECTION_STRING"
```

#### 2.6 Restart Web App

```bash
az webapp restart \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP
```

#### 2.7 ดู URL

```bash
echo "https://$WEBAPP_NAME.azurewebsites.net"
```

---

## การทดสอบ

### ทดสอบด้วย curl

```bash
# Replace with your actual URL
BASE_URL="https://sql-mcp-appservice.azurewebsites.net"

# Test GraphQL
curl -X POST $BASE_URL/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ userInfos(first: 2) { items { DisplayName Mail } } }"}'

# Test REST API (OData)
curl "$BASE_URL/api/UserInfo?\$top=5"

# Test MCP
curl -X POST $BASE_URL/mcp \
  -H "Content-Type: application/json" \
  -d '{"method":"tools/list"}'
```

### ทดสอบด้วย Postman

Import คำสั่งต่อไปนี้ใน Postman:

#### GraphQL Request

- **Method:** POST
- **URL:** `https://YOUR-URL/graphql`
- **Headers:** `Content-Type: application/json`
- **Body (raw JSON):**

```json
{
  "query": "{ userInfos { items { DisplayName Mail Company } } }"
}
```

---

## ตัวอย่าง Query

### GraphQL Queries

#### 1. ดูข้อมูล License ทั้งหมด

```graphql
{
  userInfos {
    items {
      DisplayName
      Mail
      Licenses
      MainLicense
      LicenseAddin1
      LicenseAddin2
      CurrentLicense
      Company
    }
  }
}
```

#### 2. หาพนักงานที่มี E3 License

```graphql
{
  userInfos(filter: { MainLicense: { eq: "E3" } }) {
    items {
      DisplayName
      Mail
      MainLicense
      CurrentLicense
      Company
    }
  }
}
```

#### 3. หาคนที่มี License Addin

```graphql
{
  userInfos(filter: { LicenseAddin1: { neq: null } }) {
    items {
      DisplayName
      Mail
      LicenseAddin1
      LicenseAddin2
      MainLicense
    }
  }
}
```

#### 4. หาคนที่ไม่มี Current License

```graphql
{
  userInfos(filter: { CurrentLicense: { isNull: true } }) {
    items {
      DisplayName
      Mail
      AccountEnabled
      WhenCreated
    }
  }
}
```

#### 5. ค้นหาตามชื่อ (ใช้ contains)

```graphql
{
  userInfos(filter: { DisplayName: { contains: "Thatch" } }) {
    items {
      DisplayName
      Mail
      Company
      MainLicense
    }
  }
}
```

#### 6. Pagination

```graphql
{
  userInfos(first: 10, after: "cursor_value") {
    items {
      DisplayName
      Mail
    }
    endCursor
    hasNextPage
  }
}
```

### REST API (OData) Queries

**⚠️ Note:** DAB version 1.7.83-rc ไม่รองรับ `$top` และ `$skip` parameters ใน development mode. สำหรับ pagination แนะนำให้ใช้ GraphQL แทน

```bash
BASE_URL="https://YOUR-URL"

# ดูข้อมูลทั้งหมด (ไม่ใช้ $top)
curl "$BASE_URL/api/UserInfo"

# Filter ตาม MainLicense
curl "$BASE_URL/api/UserInfo?\$filter=MainLicense eq 'E3'"

# Select เฉพาะ fields ที่ต้องการ
curl "$BASE_URL/api/UserInfo?\$select=DisplayName,Mail,MainLicense"

# Sorting
curl "$BASE_URL/api/UserInfo?\$orderby=DisplayName asc"

# Combine multiple parameters
curl "$BASE_URL/api/UserInfo?\$filter=MainLicense eq 'E3'&\$select=DisplayName,Mail&\$top=5"
```

---

## Configuration Files

### dab-config.json

ไฟล์หลักสำหรับตั้งค่า Data API Builder:

```json
{
  "$schema": "https://github.com/Azure/data-api-builder/releases/download/v1.7.86/dab.draft.schema.json",
  "data-source": {
    "database-type": "mssql",
    "connection-string": "@env('MSSQL_CONNECTION_STRING')",
    "options": {
      "set-session-context": false
    }
  },
  "runtime": {
    "rest": {
      "enabled": true,
      "path": "/api"
    },
    "graphql": {
      "enabled": true,
      "path": "/graphql",
      "allow-introspection": true
    },
    "mcp": {
      "enabled": true,
      "path": "/mcp"
    },
    "host": {
      "cors": {
        "origins": ["*"],
        "allow-credentials": false
      },
      "authentication": {
        "provider": "StaticWebApps"
      },
      "mode": "development"
    }
  },
  "entities": {
    "UserInfo": {
      "source": {
        "object": "dbo.UserInfo",
        "type": "table"
      },
      "graphql": {
        "enabled": true,
        "type": {
          "singular": "UserInfo",
          "plural": "UserInfos"
        }
      },
      "rest": {
        "enabled": true
      },
      "permissions": [
        {
          "role": "anonymous",
          "actions": ["read"]
        }
      ]
    }
  }
}
```

### Dockerfile

```dockerfile
FROM mcr.microsoft.com/azure-databases/data-api-builder:1.7.83-rc
COPY dab-config.json /App/dab-config.json
```

---

## Troubleshooting

### Container App Failed

```bash
# ดู logs
az containerapp logs show \
  --name sql-mcp-server \
  --resource-group BMG-OpenAI \
  --tail 50

# ดู revision status
az containerapp revision list \
  --name sql-mcp-server \
  --resource-group BMG-OpenAI \
  --output table
```

### App Service Failed

```bash
# ดู logs
az webapp log tail \
  --name sql-mcp-appservice \
  --resource-group BMG-CUST-SAT-RG

# Restart
az webapp restart \
  --name sql-mcp-appservice \
  --resource-group BMG-CUST-SAT-RG
```

### Common Issues

1. **Connection String ไม่ถูกต้อง**
   - ตรวจสอบ username, password, server name
   - ตรวจสอบว่า Azure SQL Firewall อนุญาต IP ของ Azure service

2. **Mode: production ไม่ทำงาน**
   - ใช้ `mode: "development"` สำหรับ Container Apps/App Service
   - `mode: "production"` ใช้ได้เฉพาะ Azure App Service แบบ native เท่านั้น

3. **CORS Error**
   - เพิ่ม origin ที่ต้องการใน `cors.origins` ใน dab-config.json
   - หรือใช้ `["*"]` เพื่ออนุญาตทุก origin (development only)

---

## Resources

- [Data API Builder Documentation](https://github.com/Azure/data-api-builder)
- [Azure Container Apps Documentation](https://learn.microsoft.com/azure/container-apps/)
- [Azure App Service Documentation](https://learn.microsoft.com/azure/app-service/)
- [GraphQL Documentation](https://graphql.org/learn/)

---

## License

This project is licensed under the MIT License.

---

## Contributors

- **Your Team Name**
- Contact: your-email@example.com

---

## Version History

- **v1.0.0** (2026-02-09): Initial release
  - REST API support
  - GraphQL support
  - MCP support
  - Azure deployment support
