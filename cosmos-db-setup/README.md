# Azure Cosmos DB SQL API — ARM Template Setup

Deploy a fully parameterized Azure Cosmos DB account (SQL / NoSQL API) using the two files in this repository:

| File | Purpose |
| ---- | ------- |
| `azuredeploy.json` | ARM template — resource definitions |
| `azuredeploy.parameters.json` | Parameter values for the deployment |

## Architecture Overview

```
Azure Resource Group
└── Cosmos DB Account  (SQL / NoSQL API)
    └── SQL Database  (AppDatabase)
        └── Container  (Items)
            ├── Partition Key: /partitionKey
            ├── Indexing: consistent (all paths included)
            └── TTL: enabled, no default expiry
```

## Prerequisites

- Azure CLI ≥ 2.50 or Azure PowerShell ≥ 10.0
- An active Azure subscription
- Contributor role (or higher) on the target Resource Group

Verify your CLI setup:

```bash
az --version
az account show
```

## Parameters Reference

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `accountName` | string | *(required)* | Globally unique Cosmos DB account name (lowercase, hyphens only) |
| `location` | string | RG location | Primary Azure region |
| `secondaryLocation` | string | `""` | Secondary region for geo-replication (leave empty to skip) |
| `databaseName` | string | `AppDatabase` | SQL database name |
| `containerName` | string | `Items` | Container name |
| `partitionKeyPath` | string | `/partitionKey` | Partition key path (must start with `/`) |
| `defaultConsistencyLevel` | string | `Session` | Consistency level: `Eventual`, `ConsistentPrefix`, `Session`, `BoundedStaleness`, `Strong` |
| `maxStalenessPrefix` | int | `100000` | Stale requests allowed (BoundedStaleness only) |
| `maxIntervalInSeconds` | int | `300` | Staleness lag in seconds (BoundedStaleness only) |
| `enableAutomaticFailover` | bool | `false` | Auto-failover for multi-region accounts |
| `enableMultipleWriteLocations` | bool | `false` | Multi-region writes (active-active) |
| `enableServerless` | bool | `false` | Serverless capacity mode (ignores throughput settings) |
| `throughputType` | string | `manual` | `manual` or `autoscale` |
| `manualThroughput` | int | `400` | Provisioned RU/s (manual mode only) |
| `autoscaleMaxThroughput` | int | `4000` | Max RU/s ceiling (autoscale mode only) |
| `enableFreeTier` | bool | `false` | Free tier discount (one account per subscription) |
| `enablePublicNetworkAccess` | bool | `true` | Allow public internet access |
| `enableAnalyticalStorage` | bool | `false` | Synapse Link / analytical store |
| `defaultTtlSeconds` | int | `-1` | Item TTL: `-1` = TTL on/no default, `0` = TTL off, `N` = expire after N seconds |
| `tags` | object | `{environment, project, managedBy}` | Resource tags |

## Deployment

### Step 1 — Edit parameters

Open `azuredeploy.parameters.json` and update at minimum:

```json
"accountName": { "value": "YOUR-UNIQUE-ACCOUNT-NAME" },
"location":    { "value": "eastus" }
```

Account name rules: 3–44 characters, lowercase letters and hyphens only, globally unique.

### Step 2 — Create a Resource Group

```bash
az group create --name rg-cosmosdb-dev --location eastus
```

### Step 3 — Validate the template (recommended)

```bash
# ensure you are in the cosmos-db-setup directory
az deployment group validate \
  --resource-group rg-cosmosdb-dev \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json
```

### Step 4 — Deploy

```bash
az deployment group create \
  --resource-group rg-cosmosdb-dev \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json \
  --name cosmos-deploy-$(date +%Y%m%d%H%M%S)
```

Deployment typically completes in **3–5 minutes**.

### Step 5 — Review outputs

```bash
# ensure to change the timestamp to the one you used in step 4
az deployment group show \
  --resource-group rg-cosmosdb-dev \
  --name cosmos-deploy-<timestamp> \
  --query properties.outputs
```

**Outputs provided:**

| Output key | Description |
| ---------- | ----------- |
| `cosmosDbAccountName` | Deployed account name |
| `cosmosDbAccountId` | Full Azure Resource ID |
| `documentEndpoint` | Cosmos DB HTTPS endpoint |
| `primaryConnectionString` | Full connection string (store securely) |
| `primaryReadonlyKey` | Read-only key |
| `databaseName` | Database name |
| `containerName` | Container name |

> **Security note:** `primaryConnectionString` and `primaryReadonlyKey` are returned as ARM outputs for convenience. In production, retrieve them directly from Key Vault or via Azure CLI to avoid logging sensitive values.

## Deployment Scenarios

### Serverless (dev/test, spiky workloads)

```json
"enableServerless": { "value": true }
```

Throughput settings are automatically ignored.

### Autoscale (variable production workloads)

```json
"enableServerless":       { "value": false },
"throughputType":         { "value": "autoscale" },
"autoscaleMaxThroughput": { "value": 4000 }
```

Cosmos DB scales between 10% and 100% of `autoscaleMaxThroughput`.

### Multi-region geo-redundancy

```json
"secondaryLocation":        { "value": "westus" },
"enableAutomaticFailover":  { "value": true }
```

### Multi-region writes (active-active)

```json
"secondaryLocation":            { "value": "westus" },
"enableAutomaticFailover":      { "value": true },
"enableMultipleWriteLocations": { "value": true }
```

### BoundedStaleness consistency

```json
"defaultConsistencyLevel": { "value": "BoundedStaleness" },
"maxStalenessPrefix":      { "value": 100000 },
"maxIntervalInSeconds":    { "value": 300 }
```

### Item expiry via TTL (e.g., 7-day session data)

```json
"defaultTtlSeconds": { "value": 604800 }
```

## Testing with Data

### Option A — Azure Portal Data Explorer

1. Navigate to your Cosmos DB account in the [Azure Portal](https://portal.azure.com)
2. Open **Data Explorer** → `AppDatabase` → `Items` → **New Item**
3. Paste the sample JSON below and click **Save**

### Option B — Azure CLI

Retrieve your endpoint and key:

```bash
ACCOUNT_NAME="myapp-cosmos-dev-001"
RESOURCE_GROUP="rg-cosmosdb-dev"

ENDPOINT=$(az cosmosdb show \
  --name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query documentEndpoint -o tsv)

KEY=$(az cosmosdb keys list \
  --name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query primaryMasterKey -o tsv)
```

Insert a document using the REST API:

```bash
BODY='{
  "id": "item-001",
  "partitionKey": "category-electronics",
  "name": "Wireless Headphones",
  "price": 89.99,
  "stock": 150,
  "tags": ["audio", "wireless", "bluetooth"],
  "createdAt": "2025-01-01T00:00:00Z"
}'

DATE=$(date -u +"%a, %d %b %Y %H:%M:%S GMT")

curl -s -X POST "${ENDPOINT}dbs/AppDatabase/colls/Items/docs" \
  -H "Authorization: type=master&ver=1.0&sig=${KEY}" \
  -H "x-ms-date: ${DATE}" \
  -H "x-ms-version: 2018-12-31" \
  -H "x-ms-documentdb-partitionkey: [\"category-electronics\"]" \
  -H "Content-Type: application/json" \
  -d "$BODY"
```

### Option C — Python SDK (recommended for programmatic testing)

Install the SDK:

```bash
pip install azure-cosmos
```

#### Insert sample documents

```python
from azure.cosmos import CosmosClient, PartitionKey

ENDPOINT = "https://myapp-cosmos-dev-001.documents.azure.com:443/"
KEY      = "<your-primary-key>"
DB_NAME  = "AppDatabase"
CON_NAME = "Items"

client    = CosmosClient(ENDPOINT, credential=KEY)
database  = client.get_database_client(DB_NAME)
container = database.get_container_client(CON_NAME)

sample_items = [
    {
        "id": "item-001",
        "partitionKey": "category-electronics",
        "name": "Wireless Headphones",
        "price": 89.99,
        "stock": 150,
        "tags": ["audio", "wireless", "bluetooth"],
        "createdAt": "2025-01-01T00:00:00Z"
    },
    {
        "id": "item-002",
        "partitionKey": "category-electronics",
        "name": "USB-C Hub",
        "price": 35.00,
        "stock": 320,
        "tags": ["accessories", "usb", "hub"],
        "createdAt": "2025-01-02T00:00:00Z"
    },
    {
        "id": "item-003",
        "partitionKey": "category-books",
        "name": "Cloud Architecture Patterns",
        "price": 49.99,
        "stock": 75,
        "tags": ["cloud", "architecture", "devops"],
        "createdAt": "2025-01-03T00:00:00Z"
    }
]

for item in sample_items:
    result = container.upsert_item(item)
    print(f"Upserted: {result['id']} (partition: {result['partitionKey']})")
```

#### Query documents

```python
# All items in a partition
query = "SELECT * FROM c WHERE c.partitionKey = 'category-electronics'"
items = list(container.query_items(query=query, enable_cross_partition_query=False))
for item in items:
    print(f"{item['id']} — {item['name']} — ${item['price']}")

# Cross-partition query with filter
query = "SELECT c.id, c.name, c.price FROM c WHERE c.price < 50.0"
items = list(container.query_items(query=query, enable_cross_partition_query=True))
for item in items:
    print(item)

# Count items
query = "SELECT VALUE COUNT(1) FROM c"
count = list(container.query_items(query=query, enable_cross_partition_query=True))
print(f"Total items: {count[0]}")
```

#### Point-read a specific item

```python
# Most efficient read — uses id + partition key directly (no index scan)
item = container.read_item(item="item-001", partition_key="category-electronics")
print(item)
```

#### Update an item

```python
item = container.read_item(item="item-001", partition_key="category-electronics")
item["stock"] = 120
item["price"] = 84.99
container.replace_item(item=item, body=item)
print("Item updated.")
```

#### Delete an item

```python
container.delete_item(item="item-001", partition_key="category-electronics")
print("Item deleted.")
```

## Clean Up

Remove all deployed resources when finished:

```bash
az group delete --name rg-cosmosdb-dev --yes --no-wait
```

## Troubleshooting

| Error | Cause | Fix |
| ----- | ----- | --- |
| `AccountNameAlreadyExists` | Account name is taken globally | Change `accountName` to something unique |
| `InvalidResourceType` | API version mismatch | Ensure ARM template API version is current |
| `RequestRateTooLarge (429)` | Throughput exhausted | Increase `manualThroughput` or switch to `autoscale` |
| `PartitionKeyMismatch` | Wrong partition key in query/insert | Ensure `partitionKey` field matches `partitionKeyPath` |
| `ForbiddenError` | Missing RBAC or wrong key | Verify your key; enable `enablePublicNetworkAccess` for dev |
| `FreeTierAlreadyEnabled` | Another account uses free tier | Set `enableFreeTier: false` |

## Security Best Practices

- Store `primaryConnectionString` and keys in **Azure Key Vault**, not in code or parameter files
- Use **Azure Managed Identity** and Cosmos DB RBAC instead of primary keys in production
- Set `enablePublicNetworkAccess: false` and use **Private Endpoints** for production workloads
- Enable **Microsoft Defender for Cosmos DB** for threat detection
- Rotate keys periodically via `az cosmosdb keys regenerate`

## References

- [Azure Cosmos DB SQL API documentation](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/)
- [ARM template reference — Microsoft.DocumentDB](https://learn.microsoft.com/en-us/azure/templates/microsoft.documentdb/databaseaccounts)
- [Azure Cosmos DB Python SDK](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/sdk-python)
- [Cosmos DB consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [Partitioning best practices](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview)
