# Adding New Tables to MCP Server

คู่มือการเพิ่ม Table ใหม่เข้า MCP Server พร้อมการตั้งค่า Permissions แบบ Custom

---

## 📋 สารบัญ

1. [ขั้นตอนการเพิ่ม Table ใหม่](#ขั้นตอนการเพิ่ม-table-ใหม่)
2. [ตัวอย่าง Entity Configuration](#ตัวอย่าง-entity-configuration)
3. [Permission Configuration (แนะนำ)](#permission-configuration)
4. [Deploy การเปลี่ยนแปลง](#deploy-การเปลี่ยนแปลง)
5. [ทดสอบ Entity ใหม่](#ทดสอบ-entity-ใหม่)

---

## ขั้นตอนการเพิ่ม Table ใหม่

### 1. ตรวจสอบ Table Structure ใน SQL Database

```sql
-- ดู columns ของ table
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE,
    COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'YourTableName'
ORDER BY ORDINAL_POSITION;

-- ดู primary key
SELECT
    COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_NAME = 'YourTableName'
    AND CONSTRAINT_NAME LIKE 'PK%';
```

---

## ตัวอย่าง Entity Configuration

### ตัวอย่างที่ 1: Table ธรรมดา (Department)

สมมติมี table `dbo.Department`:

```sql
CREATE TABLE dbo.Department (
    DepartmentId INT PRIMARY KEY,
    DepartmentName NVARCHAR(100),
    ManagerId INT,
    Budget DECIMAL(18,2),
    CreatedDate DATETIME,
    IsActive BIT
);
```

**เพิ่มใน dab-config.json:**

```json
{
  "entities": {
    "Department": {
      "description": "Department information with budget and manager",
      "source": {
        "object": "dbo.Department",
        "type": "table"
      },
      "fields": [
        {
          "name": "DepartmentId",
          "description": "Unique department identifier",
          "primary-key": true
        },
        {
          "name": "DepartmentName",
          "description": "Department name",
          "primary-key": false
        },
        {
          "name": "ManagerId",
          "description": "Manager user ID",
          "primary-key": false
        },
        {
          "name": "Budget",
          "description": "Department budget",
          "primary-key": false
        },
        {
          "name": "CreatedDate",
          "description": "Creation date",
          "primary-key": false
        },
        {
          "name": "IsActive",
          "description": "Active status",
          "primary-key": false
        }
      ],
      "graphql": {
        "enabled": true,
        "type": {
          "singular": "Department",
          "plural": "Departments"
        }
      },
      "rest": {
        "enabled": true,
        "path": "/Department"
      },
      "permissions": [
        {
          "role": "anonymous",
          "actions": [
            {
              "action": "read"
            }
          ]
        }
      ]
    }
  }
}
```

### ตัวอย่างที่ 2: Table พร้อม Relationships (Employee)

```json
{
  "entities": {
    "Employee": {
      "description": "Employee information with department relationship",
      "source": {
        "object": "dbo.Employee",
        "type": "table"
      },
      "fields": [
        {
          "name": "EmployeeId",
          "primary-key": true
        },
        {
          "name": "FirstName",
          "primary-key": false
        },
        {
          "name": "LastName",
          "primary-key": false
        },
        {
          "name": "Email",
          "primary-key": false
        },
        {
          "name": "DepartmentId",
          "primary-key": false
        },
        {
          "name": "HireDate",
          "primary-key": false
        },
        {
          "name": "Salary",
          "primary-key": false
        }
      ],
      "relationships": {
        "department": {
          "cardinality": "many-to-one",
          "target.entity": "Department",
          "source.fields": ["DepartmentId"],
          "target.fields": ["DepartmentId"]
        }
      },
      "graphql": {
        "enabled": true,
        "type": {
          "singular": "Employee",
          "plural": "Employees"
        }
      },
      "rest": {
        "enabled": true,
        "path": "/Employee"
      },
      "permissions": [
        {
          "role": "anonymous",
          "actions": [
            {
              "action": "read",
              "fields": {
                "include": ["*"],
                "exclude": ["Salary"]
              }
            }
          ]
        }
      ]
    }
  }
}
```

---

## Permission Configuration

### 🔒 แนวคิดการตั้งค่า Permissions

DAB รองรับ permission system ที่ยืดหยุ่น:

1. **Role-based:** กำหนด permission ตาม role
2. **Action-based:** กำหนด actions ที่อนุญาต (read, create, update, delete, execute)
3. **Field-level:** จำกัดการเข้าถึง fields เฉพาะ
4. **Policy-based:** กำหนด business rules เพิ่มเติม

---

### Permission Level 1: Anonymous (Read-Only) - แนะนำสำหรับ MCP

**Use Case:** MCP Server ที่ใช้เพื่อ query ข้อมูลเท่านั้น ไม่มี authentication

```json
{
  "permissions": [
    {
      "role": "anonymous",
      "actions": [
        {
          "action": "read"
        }
      ]
    }
  ]
}
```

**GraphQL Query ที่ใช้ได้:**
```graphql
{
  userInfos { items { DisplayName Mail } }
}
```

**ไม่สามารถทำ:** Create, Update, Delete

---

### Permission Level 2: Anonymous with Field Restrictions

**Use Case:** ซ่อน sensitive fields เช่น Salary, SSN

```json
{
  "permissions": [
    {
      "role": "anonymous",
      "actions": [
        {
          "action": "read",
          "fields": {
            "include": ["*"],
            "exclude": ["Salary", "SSN", "BankAccount"]
          }
        }
      ]
    }
  ]
}
```

**หรือใช้ include เท่านั้น:**

```json
{
  "permissions": [
    {
      "role": "anonymous",
      "actions": [
        {
          "action": "read",
          "fields": {
            "include": ["Id", "FirstName", "LastName", "Email", "DepartmentName"]
          }
        }
      ]
    }
  ]
}
```

---

### Permission Level 3: Multiple Roles

**Use Case:** แยก permissions สำหรับ user ประเภทต่างๆ

```json
{
  "permissions": [
    {
      "role": "anonymous",
      "actions": [
        {
          "action": "read",
          "fields": {
            "include": ["Id", "Name", "Email"]
          }
        }
      ]
    },
    {
      "role": "authenticated",
      "actions": [
        {
          "action": "read"
        },
        {
          "action": "create"
        },
        {
          "action": "update",
          "fields": {
            "include": ["Name", "Email", "Phone"]
          }
        }
      ]
    },
    {
      "role": "admin",
      "actions": [
        {
          "action": "*"
        }
      ]
    }
  ]
}
```

**Role Hierarchy:**
- `anonymous` - อ่านบาง fields ได้
- `authenticated` - อ่านได้ทั้งหมด + สร้าง + แก้ไขบาง fields
- `admin` - ทำทุกอย่างได้

---

### Permission Level 4: Policy-Based (Database-level filtering)

**Use Case:** กรองข้อมูลตาม business rules

#### ตัวอย่างที่ 1: แสดงเฉพาะ Active Records

```json
{
  "permissions": [
    {
      "role": "anonymous",
      "actions": [
        {
          "action": "read",
          "policy": {
            "database": "@item.IsActive eq true"
          }
        }
      ]
    }
  ]
}
```

**Effect:** จะมองเห็นเฉพาะ records ที่ `IsActive = true` เท่านั้น

#### ตัวอย่างที่ 2: แสดงเฉพาะ Department ของตัวเอง

```json
{
  "permissions": [
    {
      "role": "authenticated",
      "actions": [
        {
          "action": "read",
          "policy": {
            "database": "@item.DepartmentId eq @claims.department_id"
          }
        }
      ]
    }
  ]
}
```

**Effect:** User จะเห็นเฉพาะข้อมูลใน department ของตัวเอง

#### ตัวอย่างที่ 3: Complex Policy

```json
{
  "permissions": [
    {
      "role": "manager",
      "actions": [
        {
          "action": "read",
          "policy": {
            "database": "@item.DepartmentId eq @claims.department_id and @item.IsActive eq true and @item.Salary le 100000"
          }
        },
        {
          "action": "update",
          "policy": {
            "database": "@item.DepartmentId eq @claims.department_id"
          },
          "fields": {
            "include": ["Status", "Notes"],
            "exclude": ["Salary", "BonusAmount"]
          }
        }
      ]
    }
  ]
}
```

---

### Permission Level 5: Full CRUD Operations

**Use Case:** API ที่ต้องการ Create, Update, Delete

```json
{
  "permissions": [
    {
      "role": "authenticated",
      "actions": [
        {
          "action": "create",
          "fields": {
            "include": ["Name", "Description", "StartDate", "Budget"]
          }
        },
        {
          "action": "read"
        },
        {
          "action": "update",
          "fields": {
            "include": ["Description", "Status", "EndDate"]
          }
        },
        {
          "action": "delete",
          "policy": {
            "database": "@item.CreatedBy eq @claims.user_id"
          }
        }
      ]
    }
  ]
}
```

**Explanation:**
- **create:** สร้างได้แต่ระบุได้เฉพาะบาง fields
- **read:** อ่านได้ทั้งหมด
- **update:** แก้ไขได้เฉพาะบาง fields
- **delete:** ลบได้เฉพาะที่ตัวเองสร้าง

---

### Permission Level 6: Stored Procedure Permissions

**Use Case:** Execute stored procedures

```json
{
  "entities": {
    "GetEmployeesByDepartment": {
      "source": {
        "object": "dbo.sp_GetEmployeesByDepartment",
        "type": "stored-procedure",
        "parameters": {
          "DepartmentId": "Int32",
          "IncludeInactive": "Boolean"
        }
      },
      "graphql": {
        "enabled": true,
        "operation": "query"
      },
      "rest": {
        "enabled": true,
        "methods": ["POST"]
      },
      "permissions": [
        {
          "role": "authenticated",
          "actions": [
            {
              "action": "execute"
            }
          ]
        },
        {
          "role": "admin",
          "actions": [
            {
              "action": "execute"
            }
          ]
        }
      ]
    }
  }
}
```

---

## การตั้งค่า Authentication Provider

### Development Mode (ไม่มี Authentication)

```json
{
  "runtime": {
    "host": {
      "authentication": {
        "provider": "StaticWebApps"
      },
      "mode": "development"
    }
  }
}
```

**Effect:** ทุก request ถือว่าเป็น `anonymous` role

---

### Production Mode with Azure AD

```json
{
  "runtime": {
    "host": {
      "authentication": {
        "provider": "AzureAD",
        "jwt": {
          "audience": "your-application-id",
          "issuer": "https://login.microsoftonline.com/your-tenant-id/v2.0"
        }
      },
      "mode": "production"
    }
  }
}
```

**Effect:** ต้องมี JWT token ที่ valid จาก Azure AD

---

### Production Mode with Custom JWT

```json
{
  "runtime": {
    "host": {
      "authentication": {
        "provider": "StaticWebApps",
        "jwt": {
          "audience": "your-api-audience",
          "issuer": "https://your-auth-server.com"
        }
      },
      "mode": "production"
    }
  }
}
```

---

## ตัวอย่างการใช้ Claims ใน Policy

### Claims ที่ใช้ได้:

```json
{
  "userId": "user123",
  "email": "user@example.com",
  "department_id": "DEPT001",
  "role": "manager",
  "permissions": ["read", "write"]
}
```

### ใช้ Claims ใน Policy:

```json
{
  "action": "read",
  "policy": {
    "database": "@item.OwnerId eq @claims.userId and @item.DepartmentId eq @claims.department_id"
  }
}
```

### Complex Claims Logic:

```json
{
  "action": "update",
  "policy": {
    "database": "(@item.OwnerId eq @claims.userId or @claims.role eq 'admin') and @item.Status ne 'Archived'"
  }
}
```

---

## Deploy การเปลี่ยนแปลง

### Quick Deploy Script

```bash
#!/bin/bash

set -e

REGISTRY_NAME="bmgcontainer"
WEBAPP_NAME="sql-mcp-appservice"
RESOURCE_GROUP="BMG-CUST-SAT-RG"

echo "🚀 Starting deployment..."

# Build
echo "📦 Building Docker image..."
az acr build \
  --registry $REGISTRY_NAME \
  --image sql-mcp-server:latest \
  --file Dockerfile .

# Restart
echo "♻️  Restarting App Service..."
az webapp restart \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP

# Wait
echo "⏳ Waiting 30s for deployment..."
sleep 30

# Test
echo "✅ Testing endpoint..."
curl -s https://$WEBAPP_NAME.azurewebsites.net/graphql \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __typename }"}' | jq .

echo "🎉 Deployment complete!"
```

---

## ทดสอบ Entity ใหม่

### GraphQL Introspection

```graphql
# ดู schema ของ entity ใหม่
{
  __type(name: "Department") {
    name
    description
    fields {
      name
      description
      type {
        name
        kind
      }
    }
  }
}
```

### GraphQL Queries

```graphql
# Basic query
{
  departments {
    items {
      DepartmentId
      DepartmentName
      Budget
    }
  }
}

# With filter
{
  departments(filter: { IsActive: { eq: true } }) {
    items {
      DepartmentId
      DepartmentName
      Budget
    }
  }
}

# With relationship
{
  employees {
    items {
      EmployeeId
      FirstName
      LastName
      department {
        DepartmentName
        Budget
      }
    }
  }
}
```

### REST API

```bash
# Get all
curl "https://sql-mcp-appservice.azurewebsites.net/api/Department"

# Filter
curl "https://sql-mcp-appservice.azurewebsites.net/api/Department?\$filter=IsActive eq true"

# Select specific fields
curl "https://sql-mcp-appservice.azurewebsites.net/api/Department?\$select=DepartmentName,Budget"
```

---

## Testing Permission Changes

### Test Read Permission

```bash
# Should work
curl -X POST https://sql-mcp-appservice.azurewebsites.net/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ departments { items { DepartmentName } } }"}'
```

### Test Excluded Fields

```bash
# If Salary is excluded, this should return error or null
curl -X POST https://sql-mcp-appservice.azurewebsites.net/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ employees { items { Salary } } }"}'
```

### Test with Authentication Token

```bash
# With JWT token
curl -X POST https://sql-mcp-appservice.azurewebsites.net/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{"query":"{ departments { items { DepartmentName Budget } } }"}'
```

---

## Best Practices

### ✅ DO:

1. **Always define primary key**
   ```json
   { "name": "Id", "primary-key": true }
   ```

2. **Use field-level permissions for sensitive data**
   ```json
   { "exclude": ["Salary", "SSN", "CreditCard"] }
   ```

3. **Test permissions after deployment**
   ```bash
   curl -X POST .../graphql -d '{"query":"..."}'
   ```

4. **Use database policies for row-level security**
   ```json
   { "database": "@item.IsActive eq true" }
   ```

5. **Document your entities**
   ```json
   { "description": "Clear description of what this entity represents" }
   ```

### ❌ DON'T:

1. **Don't expose all fields without review**
   - Review sensitive fields first

2. **Don't use anonymous + write permissions in production**
   ```json
   // ❌ Bad for production
   { "role": "anonymous", "actions": [{ "action": "delete" }] }
   ```

3. **Don't forget to test relationship queries**
   ```graphql
   { employees { department { name } } }
   ```

4. **Don't deploy without backing up config**
   ```bash
   cp dab-config.json dab-config.json.backup
   ```

---

## Troubleshooting

### ❌ Error: Primary key not found

**Solution:**
```json
{ "fields": [{ "name": "YourIdField", "primary-key": true }] }
```

### ❌ Error: Unauthorized access

**Solution:** Check permissions config:
```json
{ "role": "anonymous", "actions": [{ "action": "read" }] }
```

### ❌ Error: Field not accessible

**Solution:** Check field permissions:
```json
{ "fields": { "include": ["*"], "exclude": [] } }
```

### ❌ Error: Policy evaluation failed

**Solution:** Check policy syntax:
```json
{ "database": "@item.FieldName eq 'Value'" }
```

---

## Quick Reference Templates

### Minimal Entity (Development)

```json
{
  "EntityName": {
    "source": { "object": "dbo.TableName", "type": "table" },
    "graphql": { "enabled": true },
    "rest": { "enabled": true },
    "permissions": [
      { "role": "anonymous", "actions": [{ "action": "read" }] }
    ]
  }
}
```

### Production Entity with Security

```json
{
  "EntityName": {
    "source": { "object": "dbo.TableName", "type": "table" },
    "fields": [
      { "name": "Id", "primary-key": true }
    ],
    "graphql": { "enabled": true },
    "rest": { "enabled": true },
    "permissions": [
      {
        "role": "authenticated",
        "actions": [
          {
            "action": "read",
            "fields": { "exclude": ["SensitiveField"] },
            "policy": { "database": "@item.IsActive eq true" }
          }
        ]
      }
    ]
  }
}
```

---

## Resources

- **DAB Docs:** https://github.com/Azure/data-api-builder
- **GraphQL:** https://graphql.org/learn/
- **OData:** https://www.odata.org/documentation/

---

**Last Updated:** 2026-02-09
**DAB Version:** 1.7.83-rc
