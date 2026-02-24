# Azure SQL MCP Server

A Model Context Protocol (MCP) server for Azure SQL Database using Data API Builder (DAB). Exposes your SQL database through REST API, GraphQL, and MCP protocol for AI agents like Microsoft Copilot Studio, Claude, and more.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Azure](https://img.shields.io/badge/Azure-Deployed-blue)](https://azure.microsoft.com)

## 🚀 Features

- **Multiple Protocols**: REST API, GraphQL, and MCP (Model Context Protocol)
- **Zero Code**: Configure your database access with JSON configuration
- **AI Agent Ready**: Direct integration with Copilot Studio, Claude Desktop, and other AI platforms
- **Azure Native**: Deploys seamlessly to Azure App Service or Container Apps
- **Production Ready**: Built on Microsoft's Data API Builder with enterprise features

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Local Development](#local-development)
- [Azure Deployment](#azure-deployment)
  - [Option 1: Azure App Service](#option-1-azure-app-service-recommended)
  - [Option 2: Azure Container Apps](#option-2-azure-container-apps)
- [Testing](#testing)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

## 🏗️ Architecture

```
┌─────────────────┐
│   AI Agents     │  (Copilot Studio, Claude, etc.)
│  - Copilot      │
│  - Claude       │
└────────┬────────┘
         │ MCP Protocol
         ▼
┌─────────────────┐
│   DAB Server    │  (This Project)
│  - REST API     │  Port 5000
│  - GraphQL      │
│  - MCP Protocol │
└────────┬────────┘
         │ SQL Queries
         ▼
┌─────────────────┐
│ Azure SQL DB    │
│  - UserInfo     │
│  - Other Tables │
└─────────────────┘
```

## Prerequisites

### Required
- **Azure Subscription** with permissions to create resources
- **Azure SQL Database** with connection string
- **Azure CLI** installed ([Install guide](https://docs.microsoft.com/cli/azure/install-azure-cli))
- **.NET SDK 8 or 9** ([Download](https://dotnet.microsoft.com/download))

### Optional
- **Docker Desktop** (for local container testing)
- **Postman** (for API testing)
- **Git** (for version control)

### Install Prerequisites (macOS)

```bash
# Install Homebrew (if not installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install required tools
brew install azure-cli dotnet@8

# Add .NET to PATH
echo 'export PATH="/opt/homebrew/opt/dotnet@8/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Verify installations
az --version
dotnet --version

# Login to Azure
az login
```

### Install Prerequisites (Windows)

#### Option 1: Using winget (Windows 10/11)

```powershell
# Open PowerShell as Administrator

# Install Azure CLI
winget install Microsoft.AzureCLI

# Install .NET SDK 8
winget install Microsoft.DotNet.SDK.8

# Install Git (if not installed)
winget install Git.Git

# Close and reopen PowerShell to refresh PATH

# Verify installations
az --version
dotnet --version
git --version

# Login to Azure
az login
```

#### Option 2: Manual Installation

1. **Azure CLI**: Download from [aka.ms/installazurecliwindows](https://aka.ms/installazurecliwindows)
2. **.NET SDK 8**: Download from [dotnet.microsoft.com/download/dotnet/8.0](https://dotnet.microsoft.com/download/dotnet/8.0)
3. **Git**: Download from [git-scm.com](https://git-scm.com/download/win)

After installation, restart PowerShell and verify:

```powershell
az --version
dotnet --version
git --version
az login
```

#### Option 3: Using Chocolatey

```powershell
# Install Chocolatey (if not installed)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install tools
choco install azure-cli dotnet-8.0-sdk git -y

# Verify
az --version
dotnet --version
git --version
az login
```

## Quick Start

### 1. Clone and Setup

**macOS/Linux:**
```bash
# Clone the repository
git clone https://github.com/theheronat/scg-sql-mcp.git
cd scg-sql-mcp

# Restore .NET tools
dotnet tool restore
```

**Windows (PowerShell):**
```powershell
# Clone the repository
git clone https://github.com/theheronat/scg-sql-mcp.git
cd scg-sql-mcp

# Restore .NET tools
dotnet tool restore
```

### 2. Configure Database Connection

Create a `.env` file in the project root:

**macOS/Linux:**
```bash
cat > .env << 'EOF'
MSSQL_CONNECTION_STRING=Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
EOF
```

**Windows (PowerShell):**
```powershell
# Create .env file
@"
MSSQL_CONNECTION_STRING=Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
"@ | Out-File -FilePath .env -Encoding utf8
```

**Or manually create `.env` file with:**
```env
MSSQL_CONNECTION_STRING=Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
```

**Important:** Never commit `.env` to Git (already in `.gitignore`)

### 3. Test Locally

```bash
# Start the DAB server (same for Windows/Mac/Linux)
dotnet dab start --config dab-config.json
```

Server runs at `http://localhost:5000`

**Endpoints:**
- REST API: `http://localhost:5000/api`
- GraphQL: `http://localhost:5000/graphql`
- MCP: `http://localhost:5000/mcp`

### 4. Quick Test

**macOS/Linux:**
```bash
# Test GraphQL endpoint
curl -X POST http://localhost:5000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ userInfos(first: 5) { items { DisplayName Mail } } }"}'
```

**Windows (PowerShell):**
```powershell
# Test GraphQL endpoint
$body = @{
    query = "{ userInfos(first: 5) { items { DisplayName Mail } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:5000/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

**Windows (curl - if installed):**
```powershell
curl -X POST http://localhost:5000/graphql `
  -H "Content-Type: application/json" `
  -d '{\"query\":\"{ userInfos(first: 5) { items { DisplayName Mail } } }\"}'
```

## Local Development

### Run with Live Reload

```bash
dotnet watch dab start --config dab-config.json
```

### Test Different Endpoints

```bash
# GraphQL Query
curl -X POST http://localhost:5000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ userInfos { items { DisplayName Mail MainLicense } } }"}'

# REST API (OData)
curl "http://localhost:5000/api/UserInfo"

# MCP Tools List
curl -X POST http://localhost:5000/mcp \
  -H "Content-Type: application/json" \
  -d '{"method":"tools/list"}'
```

## Azure Deployment

### Option 1: Azure App Service (Recommended)

Best for production workloads with predictable traffic.

#### Step 1: Set Variables

**macOS/Linux (Bash):**
```bash
# Configuration
export RESOURCE_GROUP="your-resource-group"
export LOCATION="southeastasia"
export PLAN_NAME="sql-mcp-plan"
export WEBAPP_NAME="sql-mcp-appservice"
export REGISTRY_NAME="yourregistryname"  # Must be globally unique
export CONNECTION_STRING="Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;"
```

**Windows (PowerShell):**
```powershell
# Configuration
$RESOURCE_GROUP = "your-resource-group"
$LOCATION = "southeastasia"
$PLAN_NAME = "sql-mcp-plan"
$WEBAPP_NAME = "sql-mcp-appservice"
$REGISTRY_NAME = "yourregistryname"  # Must be globally unique
$CONNECTION_STRING = "Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;"
```

#### Step 2: Create Resources

**macOS/Linux (Bash):**
```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Create Container Registry
az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku Basic \
  --admin-enabled true

# Create App Service Plan (Linux)
az appservice plan create \
  --name $PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --is-linux \
  --sku B1 \
  --location $LOCATION
```

**Windows (PowerShell):**
```powershell
# Create resource group
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION

# Create Container Registry
az acr create `
  --name $REGISTRY_NAME `
  --resource-group $RESOURCE_GROUP `
  --sku Basic `
  --admin-enabled true

# Create App Service Plan (Linux)
az appservice plan create `
  --name $PLAN_NAME `
  --resource-group $RESOURCE_GROUP `
  --is-linux `
  --sku B1 `
  --location $LOCATION
```

**SKU Options:**
- `B1` - Basic ($13/month) - Development/Testing
- `P1v2` - Premium ($73/month) - Production
- `P2v2` - Premium ($146/month) - High Performance

#### Step 3: Build and Push Container

**macOS/Linux (Bash):**
```bash
# Build on Azure (no local Docker needed)
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .
```

**Windows (PowerShell):**
```powershell
# Build on Azure (no local Docker needed)
az acr build `
  --registry $REGISTRY_NAME `
  --image sql-mcp-server:latest `
  --file Dockerfile .
```

#### Step 4: Create Web App

**macOS/Linux (Bash):**
```bash
# Create web app
az webapp create \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan $PLAN_NAME \
  --deployment-container-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest

# Get ACR credentials
export ACR_USERNAME=$(az acr credential show --name $REGISTRY_NAME --query username -o tsv)
export ACR_PASSWORD=$(az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv)

# Configure container registry
az webapp config container set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-custom-image-name $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest \
  --docker-registry-server-url https://$REGISTRY_NAME.azurecr.io \
  --docker-registry-server-user $ACR_USERNAME \
  --docker-registry-server-password "$ACR_PASSWORD"

# Configure environment
az webapp config appsettings set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    WEBSITES_PORT=5000 \
    MSSQL_CONNECTION_STRING="$CONNECTION_STRING"

# Restart to apply changes
az webapp restart \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP
```

**Windows (PowerShell):**
```powershell
# Create web app
az webapp create `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP `
  --plan $PLAN_NAME `
  --deployment-container-image-name "$REGISTRY_NAME.azurecr.io/sql-mcp-server:latest"

# Get ACR credentials
$ACR_USERNAME = az acr credential show --name $REGISTRY_NAME --query username -o tsv
$ACR_PASSWORD = az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv

# Configure container registry
az webapp config container set `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP `
  --docker-custom-image-name "$REGISTRY_NAME.azurecr.io/sql-mcp-server:latest" `
  --docker-registry-server-url "https://$REGISTRY_NAME.azurecr.io" `
  --docker-registry-server-user $ACR_USERNAME `
  --docker-registry-server-password $ACR_PASSWORD

# Configure environment
az webapp config appsettings set `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP `
  --settings `
    WEBSITES_PORT=5000 `
    MSSQL_CONNECTION_STRING="$CONNECTION_STRING"

# Restart to apply changes
az webapp restart `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP
```

#### Step 5: Get URL and Test

**macOS/Linux (Bash):**
```bash
# Display URL
echo "🚀 Your MCP Server: https://$WEBAPP_NAME.azurewebsites.net"

# Test deployment
curl -X POST https://$WEBAPP_NAME.azurewebsites.net/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __typename }"}'
```

**Windows (PowerShell):**
```powershell
# Display URL
Write-Host "🚀 Your MCP Server: https://$WEBAPP_NAME.azurewebsites.net"

# Test deployment
$body = @{ query = "{ __typename }" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://$WEBAPP_NAME.azurewebsites.net/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

### Option 2: Azure Container Apps

Best for microservices and cost optimization (scale to zero).

#### Step 1: Set Variables

```bash
export RESOURCE_GROUP="your-resource-group"
export LOCATION="southeastasia"
export REGISTRY_NAME="yourregistryname"
export ENV_NAME="sql-mcp-env"
export CONTAINERAPP_NAME="sql-mcp-server"
export CONNECTION_STRING="Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;"
```

#### Step 2: Create Resources

```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Create Container Registry
az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku Basic \
  --admin-enabled true

# Build image
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .

# Create Container Apps Environment
az containerapp env create \
  --name $ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

#### Step 3: Deploy Container App

```bash
# Get ACR credentials
export ACR_USERNAME=$(az acr credential show --name $REGISTRY_NAME --query username -o tsv)
export ACR_PASSWORD=$(az acr credential show --name $REGISTRY_NAME --query "passwords[0].value" -o tsv)

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

#### Step 4: Get URL and Test

```bash
# Get FQDN
export MCP_URL=$(az containerapp show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.configuration.ingress.fqdn" -o tsv)

echo "🚀 Your MCP Server: https://$MCP_URL"

# Test
curl -X POST https://$MCP_URL/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __typename }"}'
```

## Testing

### GraphQL Queries

**macOS/Linux (Bash):**
```bash
# Get first 5 users
curl -X POST https://YOUR-URL/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ userInfos(first: 5) { items { DisplayName Mail MainLicense Company } } }"
  }'

# Filter by license type
curl -X POST https://YOUR-URL/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ userInfos(filter: { MainLicense: { eq: \"E3\" } }) { items { DisplayName Mail } } }"
  }'

# Search by name
curl -X POST https://YOUR-URL/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ userInfos(filter: { DisplayName: { contains: \"John\" } }) { items { DisplayName Mail } } }"
  }'
```

**Windows (PowerShell):**
```powershell
# Get first 5 users
$body = @{
    query = "{ userInfos(first: 5) { items { DisplayName Mail MainLicense Company } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://YOUR-URL/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body

# Filter by license type
$body = @{
    query = "{ userInfos(filter: { MainLicense: { eq: \`"E3\`" } }) { items { DisplayName Mail } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://YOUR-URL/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body

# Search by name
$body = @{
    query = "{ userInfos(filter: { DisplayName: { contains: \`"John\`" } }) { items { DisplayName Mail } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://YOUR-URL/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

### REST API Queries

**macOS/Linux (Bash):**
```bash
# Get all users
curl "https://YOUR-URL/api/UserInfo"

# Filter by license
curl "https://YOUR-URL/api/UserInfo?\$filter=MainLicense eq 'E3'"

# Select specific fields
curl "https://YOUR-URL/api/UserInfo?\$select=DisplayName,Mail,MainLicense"
```

**Windows (PowerShell):**
```powershell
# Get all users
Invoke-RestMethod -Uri "https://YOUR-URL/api/UserInfo"

# Filter by license
Invoke-RestMethod -Uri "https://YOUR-URL/api/UserInfo?`$filter=MainLicense eq 'E3'"

# Select specific fields
Invoke-RestMethod -Uri "https://YOUR-URL/api/UserInfo?`$select=DisplayName,Mail,MainLicense"
```

### MCP Protocol

**macOS/Linux (Bash):**
```bash
# List available tools
curl -X POST https://YOUR-URL/mcp \
  -H "Content-Type: application/json" \
  -d '{"method":"tools/list"}'

# Describe entities
curl -X POST https://YOUR-URL/mcp \
  -H "Content-Type: application/json" \
  -d '{"method":"describe_entities"}'
```

**Windows (PowerShell):**
```powershell
# List available tools
$body = @{ method = "tools/list" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://YOUR-URL/mcp" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body

# Describe entities
$body = @{ method = "describe_entities" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://YOUR-URL/mcp" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

### Using Postman

Import the provided collection:

```bash
# Collection file is included in repository
postman-collection.json
```

## Configuration

### dab-config.json

Main configuration file for Data API Builder:

```json
{
  "data-source": {
    "database-type": "mssql",
    "connection-string": "@env('MSSQL_CONNECTION_STRING')"
  },
  "runtime": {
    "rest": { "enabled": true, "path": "/api" },
    "graphql": { "enabled": true, "path": "/graphql" },
    "mcp": { "enabled": true, "path": "/mcp" },
    "host": {
      "mode": "development",
      "authentication": { "provider": "StaticWebApps" }
    }
  },
  "entities": {
    "UserInfo": {
      "source": { "object": "dbo.UserInfo", "type": "table" },
      "permissions": [
        { "role": "anonymous", "actions": ["read"] }
      ]
    }
  }
}
```

### Adding New Tables

See [ADD-NEW-TABLE.md](ADD-NEW-TABLE.md) for detailed instructions.

Quick example:

```json
"entities": {
  "NewTable": {
    "source": {
      "object": "dbo.NewTable",
      "type": "table"
    },
    "permissions": [
      {
        "role": "anonymous",
        "actions": ["read", "create", "update", "delete"]
      }
    ]
  }
}
```

## Troubleshooting

### Common Issues

**1. Connection Timeout**

```bash
# Check SQL firewall rules
# Add Azure service access in Azure Portal
# Or add your IP address
```

**2. Authentication Failed**

```bash
# Verify connection string credentials
az webapp config appsettings list \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[?name=='MSSQL_CONNECTION_STRING']"
```

**3. Container Won't Start**

```bash
# Check logs (App Service)
az webapp log tail \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP

# Check logs (Container Apps)
az containerapp logs show \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --tail 50
```

**4. 503 Service Unavailable**

- Container is still starting (wait 1-2 minutes)
- Check if always-on is enabled (App Service only)
- Verify WEBSITES_PORT=5000 is set

**5. GraphQL Schema Not Loading**

- Verify database connection string
- Check database permissions
- Ensure table exists and has correct schema

### Enable Detailed Logging

```bash
# App Service
az webapp log config \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-container-logging filesystem \
  --level verbose
```

### Update Deployment

**macOS/Linux (Bash):**
```bash
# Rebuild image
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .

# Restart service (App Service)
az webapp restart \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP

# Update service (Container Apps)
az containerapp update \
  --name $CONTAINERAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --image $REGISTRY_NAME.azurecr.io/sql-mcp-server:latest
```

**Windows (PowerShell):**
```powershell
# Rebuild image
az acr build `
  --registry $REGISTRY_NAME `
  --image sql-mcp-server:latest `
  --file Dockerfile .

# Restart service (App Service)
az webapp restart `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP

# Update service (Container Apps)
az containerapp update `
  --name $CONTAINERAPP_NAME `
  --resource-group $RESOURCE_GROUP `
  --image "$REGISTRY_NAME.azurecr.io/sql-mcp-server:latest"
```

### Windows-Specific Troubleshooting

**1. PowerShell Execution Policy Error**

```powershell
# If you get "execution policy" error
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**2. Certificate/SSL Issues**

```powershell
# If you get SSL/TLS errors
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

**3. .env File Encoding Issues**

```powershell
# Ensure .env file is UTF-8 without BOM
Get-Content .env | Set-Content -Encoding UTF8 .env
```

**4. Port Already in Use**

```powershell
# Find process using port 5000
netstat -ano | findstr :5000

# Kill process (replace PID with actual process ID)
taskkill /PID <PID> /F
```

**5. Path Issues with dotnet**

```powershell
# Add dotnet to PATH permanently
[Environment]::SetEnvironmentVariable(
    "Path",
    "$env:Path;C:\Program Files\dotnet",
    [EnvironmentVariableTarget]::User
)

# Refresh PATH in current session
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
```

**6. curl Not Found**

```powershell
# Use Invoke-RestMethod instead
$body = @{ query = "{ __typename }" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:5000/graphql" -Method Post -ContentType "application/json" -Body $body

# Or install curl via chocolatey
choco install curl -y
```

**7. Git Line Ending Issues**

```powershell
# Configure git for Windows
git config --global core.autocrlf true
```

## Resources

### Documentation
- [Data API Builder GitHub](https://github.com/Azure/data-api-builder)
- [MCP Protocol Specification](https://modelcontextprotocol.io)
- [Azure App Service Docs](https://learn.microsoft.com/azure/app-service/)
- [Azure Container Apps Docs](https://learn.microsoft.com/azure/container-apps/)

### Related Guides
- [MCP Agent Setup Guide](MCP-AGENT-SETUP.md) - Connect to AI platforms
- [Copilot Studio Guide](COPILOT-STUDIO-GUIDE.md) - Microsoft Copilot integration
- [Deployment Guide](DEPLOYMENT.md) - Advanced deployment scenarios
- [Add New Table Guide](ADD-NEW-TABLE.md) - Configure additional entities

### Example Queries
See [postman-collection.json](postman-collection.json) for comprehensive examples.

## Cost Estimate

**Monthly costs (Southeast Asia region):**

| Service | Tier | Cost (USD) |
|---------|------|------------|
| App Service | B1 | ~$13 |
| App Service | P1v2 | ~$73 |
| Container Apps | Consumption | ~$5-20 (usage-based) |
| Container Registry | Basic | ~$5 |
| SQL Database | Basic | ~$5 |
| SQL Database | Standard S0 | ~$15 |

**Total estimate:** $23-$98/month depending on tier selection

## Security Notes

- Never commit `.env` files (already in `.gitignore`)
- Use Azure Key Vault for production secrets
- Enable Managed Identity for SQL authentication
- Restrict network access with firewall rules
- Use HTTPS only (enforced by default)
- Regularly update DAB image version

## License

MIT License - see [LICENSE](LICENSE) file for details

## Support

- **Issues**: [GitHub Issues](https://github.com/theheronat/scg-sql-mcp/issues)
- **Discussions**: [GitHub Discussions](https://github.com/theheronat/scg-sql-mcp/discussions)
- **Email**: your-email@example.com

## Contributing

Contributions welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

## Acknowledgments

- Built with [Data API Builder](https://github.com/Azure/data-api-builder) by Microsoft
- Implements [Model Context Protocol](https://modelcontextprotocol.io) by Anthropic
- Hosted on Microsoft Azure

---

**Version:** 1.0.0
**Last Updated:** February 2026
**Maintained by:** Your Team Name
