# MCP Agent Setup Guide

คู่มือการเชื่อมต่อ Azure SQL MCP Server กับ AI Agents

---

## Azure AI Agent Builder Setup

### ขั้นตอนการเชื่อมต่อ:

#### 1. เปิด Azure AI Agent Builder

ไปที่: https://ai.azure.com หรือ Azure AI Studio

#### 2. เพิ่ม MCP Tool

1. คลิก **Add Model Context Protocol tool**
2. กรอกข้อมูลดังนี้:

### ⚙️ Configuration

**Name:**
```
Azure-SQL-MCP-Server
```

**Remote MCP Server endpoint:**
```
https://sql-mcp-appservice.azurewebsites.net/mcp
```

**Authentication:**
- เลือก `Key-based`

**Credential (Key-Value Pair):**

**สำหรับ Development/Testing:**
```
Key: X-API-Key
Value: development
```

**หรือ:**
```
Key: Authorization
Value: Bearer anonymous
```

> **หมายเหตุ:** เนื่องจาก MCP Server ตั้งค่าเป็น `development mode` กับ `anonymous permissions` ค่า credential เหล่านี้จะไม่ถูกตรวจสอบ แต่ Azure AI Agent Builder อาจบังคับให้ใส่

#### 3. คลิก Connect

รอสักครู่ให้ระบบเชื่อมต่อ

#### 4. ตรวจสอบการเชื่อมต่อ

หลังจาก Connect สำเร็จ คุณจะเห็น Tools ที่มีจาก MCP Server:
- **UserInfo queries** - สำหรับ query ข้อมูล user information
- **License queries** - สำหรับ query ข้อมูล license

---

## Claude Desktop Setup

### ขั้นตอนการเชื่อมต่อกับ Claude Desktop:

#### 1. หาไฟล์ Config

**macOS:**
```
~/Library/Application Support/Claude/claude_desktop_config.json
```

**Windows:**
```
%APPDATA%\Claude\claude_desktop_config.json
```

**Linux:**
```
~/.config/Claude/claude_desktop_config.json
```

#### 2. แก้ไขไฟล์ Config

เพิ่ม MCP Server configuration:

```json
{
  "mcpServers": {
    "azure-sql-server": {
      "url": "https://sql-mcp-appservice.azurewebsites.net/mcp",
      "transport": "sse"
    }
  }
}
```

**หรือถ้าใช้ Local DAB Server:**

```json
{
  "mcpServers": {
    "azure-sql-server": {
      "command": "/opt/homebrew/opt/dotnet@8/bin/dotnet",
      "args": [
        "dab",
        "start",
        "--config",
        "/path/to/dab-config.json",
        "--no-https-redirect"
      ]
    }
  }
}
```

#### 3. Restart Claude Desktop

ปิดและเปิด Claude Desktop ใหม่

#### 4. ตรวจสอบการเชื่อมต่อ

ใน Claude Desktop จะมีไอคอน 🔌 แสดงว่ามี MCP Server เชื่อมต่ออยู่

---

## Other AI Platforms

### OpenAI GPTs / Custom Actions

**Action Schema:**

```yaml
openapi: 3.0.0
info:
  title: Azure SQL MCP Server
  version: 1.0.0
servers:
  - url: https://sql-mcp-appservice.azurewebsites.net
paths:
  /graphql:
    post:
      operationId: queryUserInfo
      summary: Query user information including licenses
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query:
                  type: string
                  description: GraphQL query
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
```

### Microsoft Copilot Studio

1. ไปที่ **Topics** → **+ New topic**
2. เลือก **Call an action**
3. เพิ่ม **Custom endpoint**
   - URL: `https://sql-mcp-appservice.azurewebsites.net/graphql`
   - Method: `POST`
   - Headers:
     - `Content-Type: application/json`
   - Body:
     ```json
     {
       "query": "{ userInfos(first: 10) { items { DisplayName Mail MainLicense } } }"
     }
     ```

---

## ตัวอย่างการใช้งาน

### คำถามที่สามารถถามได้:

**เกี่ยวกับ License:**
- "มีพนักงานกี่คนที่ใช้ E3 license?"
- "รายชื่อคนที่มี license เสริม (LicenseAddin1)"
- "คนไหนบ้างที่ไม่มี Current License?"
- "แสดงสรุป license ทั้งหมดแยกตามประเภท"

**เกี่ยวกับ User Information:**
- "หา user ที่ชื่อ Thatchakorn"
- "แสดง email ของทุกคนใน company SCG"
- "รายชื่อ user ที่ account ถูก disable"
- "หา user ที่สร้างในเดือนนี้"

**เกี่ยวกับ Cost Center และ Business Unit:**
- "รายชื่อพนักงานใน cost center 12345"
- "แสดงข้อมูล user ทั้งหมดใน BU Digital"
- "สรุปจำนวน license แยกตาม BU"

---

## Troubleshooting

### ❌ Error: Cannot connect to MCP server

**แก้ไข:**
1. ตรวจสอบว่า URL ถูกต้อง: `https://sql-mcp-appservice.azurewebsites.net/mcp`
2. ลองเปิด URL ใน browser ดูว่า server ทำงานปกติ
3. ตรวจสอบ App Service status: `az webapp show --name sql-mcp-appservice --resource-group BMG-CUST-SAT-RG --query state`

### ❌ Error: Authentication failed

**แก้ไข:**
1. ลองเปลี่ยน Authentication เป็น `None` (ถ้ามี option)
2. หรือใช้ dummy credential:
   - Key: `X-API-Key`
   - Value: `development`

### ❌ Error: No tools found

**แก้ไข:**
1. ตรวจสอบว่า MCP endpoint `/mcp` ทำงานได้
2. ตรวจสอบ `dab-config.json` ว่ามี `"mcp": { "enabled": true }`
3. Restart App Service: `az webapp restart --name sql-mcp-appservice --resource-group BMG-CUST-SAT-RG`

### ❌ Error: Timeout

**แก้ไข:**
1. App Service อาจจะ sleep (cold start) - รอ 30 วินาที แล้วลองใหม่
2. เช็ค logs: `az webapp log tail --name sql-mcp-appservice --resource-group BMG-CUST-SAT-RG`

---

## Security Best Practices

### สำหรับ Production:

#### 1. เพิ่ม API Key Validation

แก้ไข `dab-config.json`:

```json
{
  "runtime": {
    "host": {
      "authentication": {
        "provider": "StaticWebApps"
      },
      "mode": "production"
    }
  }
}
```

**หมายเหตุ:** การเปลี่ยนเป็น production mode กับ StaticWebApps provider บน App Service จะต้องมี proper authentication setup

#### 2. จำกัด CORS Origins

แก้ไข `dab-config.json`:

```json
{
  "runtime": {
    "host": {
      "cors": {
        "origins": [
          "https://ai.azure.com",
          "https://claude.ai",
          "https://your-allowed-domain.com"
        ],
        "allow-credentials": true
      }
    }
  }
}
```

#### 3. ใช้ Azure AD Authentication

สำหรับ enterprise security:

```bash
# Enable Azure AD authentication on App Service
az webapp auth update \
  --name sql-mcp-appservice \
  --resource-group BMG-CUST-SAT-RG \
  --enabled true \
  --action LoginWithAzureActiveDirectory
```

#### 4. Network Restrictions

จำกัด IP ที่สามารถเข้าถึงได้:

```bash
# Add IP restriction
az webapp config access-restriction add \
  --name sql-mcp-appservice \
  --resource-group BMG-CUST-SAT-RG \
  --rule-name allow-azure-ai \
  --action Allow \
  --ip-address <Azure_AI_IP>/32 \
  --priority 100
```

---

## Testing

### Test MCP Endpoint with curl

```bash
# Test GraphQL through MCP
curl -X POST https://sql-mcp-appservice.azurewebsites.net/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ userInfos(first: 2) { items { DisplayName Mail } } }"}'
```

### Test with Postman

Import [postman-collection.json](postman-collection.json) และทดสอบ endpoints ต่างๆ

---

## Additional Resources

- **MCP Specification:** https://modelcontextprotocol.io/
- **Data API Builder Docs:** https://github.com/Azure/data-api-builder
- **Azure AI Studio:** https://ai.azure.com
- **Claude Desktop:** https://claude.ai/download

---

## Support

สำหรับปัญหาหรือคำถาม:
- **GitHub Issues:** [Your repo]
- **Email:** your-email@example.com
- **Documentation:** [Your docs]

---

**Last Updated:** 2026-02-09
**MCP Server Version:** 1.7.83-rc
**Protocol Version:** 2024-11-05
