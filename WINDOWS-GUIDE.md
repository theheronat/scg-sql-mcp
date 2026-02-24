# Windows Quick Start Guide

Complete guide for deploying Azure SQL MCP Server on Windows using PowerShell.

## Table of Contents

- [Prerequisites Installation](#prerequisites-installation)
- [Local Setup](#local-setup)
- [Azure Deployment](#azure-deployment)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites Installation

### Method 1: winget (Recommended for Windows 10/11)

Open **PowerShell as Administrator**:

```powershell
# Install Azure CLI
winget install Microsoft.AzureCLI

# Install .NET SDK 8
winget install Microsoft.DotNet.SDK.8

# Install Git
winget install Git.Git

# Close and reopen PowerShell

# Verify installations
az --version
dotnet --version
git --version

# Login to Azure
az login
```

### Method 2: Chocolatey

```powershell
# Install Chocolatey first
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install tools
choco install azure-cli dotnet-8.0-sdk git -y

# Verify
az --version
dotnet --version
az login
```

### Method 3: Manual Installation

Download and install:
1. **Azure CLI**: https://aka.ms/installazurecliwindows
2. **.NET 8 SDK**: https://dotnet.microsoft.com/download/dotnet/8.0
3. **Git**: https://git-scm.com/download/win

---

## Local Setup

### 1. Clone Repository

```powershell
# Clone the repository
git clone https://github.com/theheronat/scg-sql-mcp.git
cd scg-sql-mcp

# Restore .NET tools
dotnet tool restore
```

### 2. Create .env File

**Using PowerShell:**
```powershell
@"
MSSQL_CONNECTION_STRING=Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
"@ | Out-File -FilePath .env -Encoding utf8
```

**Or create manually in Notepad:**
- Create file named `.env` (no extension)
- Save as UTF-8 encoding
- Add connection string

### 3. Start Local Server

```powershell
# Start DAB server
dotnet dab start --config dab-config.json
```

Server runs at: `http://localhost:5000`

### 4. Test Locally

```powershell
# Test GraphQL
$body = @{
    query = "{ userInfos(first: 5) { items { DisplayName Mail } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:5000/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

---

## Azure Deployment

### Step 1: Set Variables

```powershell
# Configuration
$RESOURCE_GROUP = "your-resource-group"
$LOCATION = "southeastasia"
$PLAN_NAME = "sql-mcp-plan"
$WEBAPP_NAME = "sql-mcp-appservice"
$REGISTRY_NAME = "yourregistryname"  # Must be globally unique
$CONNECTION_STRING = "Server=tcp:YOUR_SERVER.database.windows.net,1433;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USERNAME;Password=YOUR_PASSWORD;Encrypt=True;"
```

### Step 2: Create Azure Resources

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

# Create App Service Plan
az appservice plan create `
  --name $PLAN_NAME `
  --resource-group $RESOURCE_GROUP `
  --is-linux `
  --sku B1 `
  --location $LOCATION
```

### Step 3: Build Container

```powershell
# Build on Azure
az acr build `
  --registry $REGISTRY_NAME `
  --image sql-mcp-server:latest `
  --file Dockerfile .
```

### Step 4: Deploy Web App

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

# Configure container
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

# Restart app
az webapp restart `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP
```

### Step 5: Test Deployment

```powershell
# Display URL
Write-Host "🚀 Your MCP Server: https://$WEBAPP_NAME.azurewebsites.net"

# Test
$body = @{ query = "{ __typename }" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://$WEBAPP_NAME.azurewebsites.net/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

---

## Testing

### GraphQL Queries

```powershell
# Get users
$body = @{
    query = "{ userInfos(first: 5) { items { DisplayName Mail MainLicense } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://YOUR-URL/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body

# Filter by license
$body = @{
    query = "{ userInfos(filter: { MainLicense: { eq: \`"E3\`" } }) { items { DisplayName Mail } } }"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://YOUR-URL/graphql" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

### REST API

```powershell
# Get all users
Invoke-RestMethod -Uri "https://YOUR-URL/api/UserInfo"

# Filter by license
Invoke-RestMethod -Uri "https://YOUR-URL/api/UserInfo?`$filter=MainLicense eq 'E3'"

# Select fields
Invoke-RestMethod -Uri "https://YOUR-URL/api/UserInfo?`$select=DisplayName,Mail,MainLicense"
```

### MCP Protocol

```powershell
# List tools
$body = @{ method = "tools/list" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://YOUR-URL/mcp" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

---

## Troubleshooting

### 1. Execution Policy Error

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### 2. Certificate/SSL Issues

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

### 3. .env File Encoding Issues

```powershell
# Ensure UTF-8 without BOM
Get-Content .env | Set-Content -Encoding UTF8 .env
```

### 4. Port 5000 Already in Use

```powershell
# Find process
netstat -ano | findstr :5000

# Kill process (replace PID)
taskkill /PID <PID> /F
```

### 5. dotnet Not Found

```powershell
# Add to PATH permanently
[Environment]::SetEnvironmentVariable(
    "Path",
    "$env:Path;C:\Program Files\dotnet",
    [EnvironmentVariableTarget]::User
)

# Refresh current session
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
```

### 6. curl Not Found in PowerShell

Use `Invoke-RestMethod` instead (as shown in examples above), or install curl:

```powershell
choco install curl -y
```

### 7. Line Ending Issues with Git

```powershell
git config --global core.autocrlf true
```

### 8. Connection String Special Characters

If connection string has special characters in PowerShell:

```powershell
# Use single quotes
$CONNECTION_STRING = 'Server=tcp:...'

# Or escape with backtick
$PASSWORD = "P@``ssw0rd"
```

---

## Common Commands Reference

### Azure CLI

```powershell
# Login
az login

# List subscriptions
az account list --output table

# Set subscription
az account set --subscription "Subscription Name"

# List resource groups
az group list --output table

# List web apps
az webapp list --output table
```

### Check Status

```powershell
# Check web app status
az webapp show --name $WEBAPP_NAME --resource-group $RESOURCE_GROUP

# View logs
az webapp log tail --name $WEBAPP_NAME --resource-group $RESOURCE_GROUP

# List app settings
az webapp config appsettings list --name $WEBAPP_NAME --resource-group $RESOURCE_GROUP
```

### Update Deployment

```powershell
# Rebuild image
az acr build `
  --registry $REGISTRY_NAME `
  --image sql-mcp-server:latest `
  --file Dockerfile .

# Restart app
az webapp restart `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP
```

---

## PowerShell Tips

### Line Continuation

Use backtick (`) for multi-line commands:

```powershell
az webapp create `
  --name $WEBAPP_NAME `
  --resource-group $RESOURCE_GROUP
```

### String Escaping

```powershell
# Escape double quotes
"He said \`"Hello\`""

# Or use single quotes
'He said "Hello"'
```

### Environment Variables

```powershell
# Set for current session
$env:VARIABLE_NAME = "value"

# Set permanently
[Environment]::SetEnvironmentVariable("VARIABLE_NAME", "value", [EnvironmentVariableTarget]::User)
```

---

## Next Steps

1. **Connect to AI Agent**: See [MCP-AGENT-SETUP.md](MCP-AGENT-SETUP.md)
2. **Add Tables**: See [ADD-NEW-TABLE.md](ADD-NEW-TABLE.md)
3. **Copilot Studio**: See [COPILOT-STUDIO-GUIDE.md](COPILOT-STUDIO-GUIDE.md)

---

## Support

- **Main README**: [README.md](README.md)
- **GitHub Issues**: https://github.com/theheronat/scg-sql-mcp/issues
- **Azure Docs**: https://learn.microsoft.com/azure

---

**Note**: Replace `YOUR_SERVER`, `YOUR_DATABASE`, `YOUR_USERNAME`, `YOUR_PASSWORD`, `YOUR-URL` with actual values.
