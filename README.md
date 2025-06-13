# Cost Optimization for Azure Cosmos DB Billing Records

## 1. Overview

This document outlines a cost optimization solution for a read-heavy Azure Cosmos DB storing billing records in a serverless architecture.

- **Total Records:** 2 million  
- **Average Record Size:** 300 KB  
- **Total Data Size:** ~600 GB  
- **Optimization:** Archive records older than 3 months to Azure Blob Storage (Cool tier)

The solution ensures:
- ✅ Simplicity
- ✅ No data loss
- ✅ No downtime
- ✅ No API contract changes
- ✅ Access latency in seconds for old records

---

## 2. Architecture

The architecture leverages Azure services for seamless data management:

- **Client APIs:** Send read/write requests via Azure API Management  
- **Azure Functions (API Layer):** Handles requests from API Management  
- **Azure Cosmos DB (Serverless):** Stores records less than 3 months old  
- **Azure Blob Storage (Cool Tier):** Archives records older than 3 months  
- **Azure Functions (Archival):** Processes Cosmos DB change feed to archive records

![Cost Optimization Architecture](e9674aaa-206a-4845-a1c2-3897638d78b9.png)

---

## 3. Implementation

### 3.1 Cosmos DB Configuration

```json
{
  "indexingPolicy": {
    "includedPaths": [
      {"path": "/recordId/?"},
      {"path": "/timestamp/?"},
      {"path": "/customerId/?"}
    ],
    "excludedPaths": [{"path": "/*"}]
  },
  "defaultTtl": 7776000
}
```

### 3.2 Blob Storage Setup

```bash
az storage container create --name billing-records-archive \
  --account-name <storage-account-name> \
  --auth-mode login \
  --default-encryption-scope default
```

Lifecycle policy:

```json
{
  "rules": [
    {
      "name": "archiveOldRecords",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {"blobTypes": ["blockBlob"]},
        "actions": {
          "baseBlob": {
            "tierToCool": {"daysAfterModificationGreaterThan": 90}
          }
        }
      }
    }
  ]
}
```

### 3.3 Archival Function

> Azure Function triggered by Cosmos DB change feed  
> *(Code omitted for brevity)*

### 3.4 API Function

> Unified API layer for read/write  
> *(Code omitted for brevity)*

### 3.5 Deployment Commands

```bash
# Archival Function
func init BillingArchival --python
cd BillingArchival
func new --name ArchiveRecords --template "Azure Cosmos DB Trigger"
func azure functionapp publish <function-app-name> --python

# API Function
func init BillingAPI --python
cd BillingAPI
func new --name GetRecord --template "HTTP Trigger"
func azure functionapp publish <function-app-name> --python

# API Management
az apim api import \
  --resource-group <resource-group> \
  --service-name <apim-name> \
  --api-id billing-api \
  --path /records \
  --service-url https://<function-app-name>.azurewebsites.net/api \
  --specification-format OpenApi
```

---

## 4. Cost Optimization

| Cost Component            | Unit Cost         | Usage               | Monthly Cost |
|--------------------------|-------------------|---------------------|--------------|
| **Before Optimization**  |                   |                     |              |
| Cosmos DB Storage        | $0.025 / GB       | 600 GB              | $15.00       |
| RU Consumption           | ~100,000 RUs/day  | ~3M RUs/month       | $7.50        |
| **Total (Before)**       |                   |                     | **$22.50**   |
|                          |                   |                     |              |
| **After Optimization**   |                   |                     |              |
| Cosmos DB Storage        | $0.025 / GB       | 150 GB              | $3.75        |
| Cosmos DB RU Charges     | Reduced by 70%    | ~0.9M RUs/month     | $2.25        |
| Blob Storage (Cool Tier) | $0.015 / GB       | 450 GB              | $6.75        |
| **Total (After)**        |                   |                     | **$12.75**   |

### 💰 Savings

| Description    | Amount   |
|----------------|----------|
| Before Total   | $22.50   |
| After Total    | $12.75   |
| **Monthly Savings** | **$9.75 (~43%)** |

---

## 5. Requirements

- ✅ **Simplicity:** Managed services, minimal code  
- ✅ **No Data Loss:** Change feed archives before deletion  
- ✅ **No Downtime:** Asynchronous archival  
- ✅ **No API Changes:** API Management maintains contracts  
- ✅ **Performance:** Cosmos DB <10ms, Blob Storage <1s

---

## 6. Notes

- 🔁 **Rehydration Logic:** Optionally move hot records back to Cosmos DB  
- 📈 **Scalability:** Serverless and managed services scale automatically  
- ⚡ **Alternative:** Use Blob Hot Tier ($0.018/GB/month) for lower latency
