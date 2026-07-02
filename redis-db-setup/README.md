# Azure Cache for Redis — ARM Template Setup

Deploy a fully parameterized Azure Cache for Redis using the two files in this repository:

| File | Purpose |
| ---- | ------- |
| `azuredeploy.json` | ARM template — resource definitions |
| `azuredeploy.parameters.json` | Parameter values for the deployment |

## Architecture Overview

```text
Azure Resource Group
└── Azure Cache for Redis
    ├── SKU: Basic, Standard, or Premium
    ├── Capacity: 0, 1, 2, ...
    └── TLS Configuration: 1.2 minimum
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
| `redisCacheName` | string | *(required)* | Globally unique Redis Cache name |
| `location` | string | RG location | Primary Azure region |
| `skuName` | string | `Basic` | `Basic`, `Standard`, or `Premium` |
| `skuFamily` | string | `C` | `C` (Basic/Standard) or `P` (Premium) |
| `capacity` | int | `0` | Pricing tier capacity (0-6) |
| `enableNonSslPort` | bool | `false` | Enable unencrypted access on port 6379 |
| `minimumTlsVersion` | string | `1.2` | Minimum TLS version required |
| `tags` | object | `{environment, project, managedBy}` | Resource tags |

## Deployment

### Step 1 — Edit parameters

Open `azuredeploy.parameters.json` and update at minimum:

```json
"redisCacheName": { "value": "YOUR-UNIQUE-REDIS-NAME" }
```

### Step 2 — Create a Resource Group (if needed)

```bash
az group create --name rg-redis-dev --location eastus
```

### Step 3 — Validate the template (recommended)

```bash
# ensure you are in the redis-db-setup directory
az deployment group validate \
  --resource-group rg-redis-dev \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json
```

### Step 4 — Deploy

```bash
az deployment group create \
  --resource-group rg-redis-dev \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json \
  --name redis-deploy-$(date +%Y%m%d%H%M%S)
```

Deployment typically completes in **15–20 minutes** depending on the SKU.

### Step 5 — Review outputs

```bash
# ensure to change the timestamp to the one you used in step 4
az deployment group show \
  --resource-group rg-redis-dev \
  --name redis-deploy-<timestamp> \
  --query properties.outputs
```

**Outputs provided:**

| Output key | Description |
| ---------- | ----------- |
| `redisCacheName` | Deployed cache name |
| `redisCacheId` | Full Azure Resource ID |
| `hostName` | Redis connection endpoint |
| `sslPort` | SSL port (usually 6380) |
| `primaryKey` | Primary access key |

> **Security note:** Passing the primary key directly to outputs is convenient for testing, but in production, fetch it dynamically from **Azure Key Vault**.

## Testing with Data

### Option A1 — Redis CLI (Terminal)

If you have `redis-cli` installed and want to connect over SSL/TLS, you might need to use `stunnel` (since native redis-cli doesn't always support TLS easily).

Alternatively, if you temporally set `enableNonSslPort` to `true` (not recommended for production):
```bash
redis-cli -h myapp-redis-dev-001.redis.cache.windows.net -p 6379 -a <your-primary-key>
```

```text
> PING
PONG
> SET session:101 "{'user':'john','role':'admin'}" EX 3600
OK
> GET session:101
"{'user':'john','role':'admin'}"
```

### Option A2 — Redis Console on Azure Portal

```bash
# go to the azure portal
# search for your redis cache
# click on "Console" in the left-hand menu
# use the commands below
```

```text
> PING
PONG
> SET session:101 "{'user':'john','role':'admin'}" EX 3600
OK
> GET session:101
"{'user':'john','role':'admin'}"
```

### Option B — Python Testing (Programmatic)

Install the required library:

```bash
pip install redis
```

Run a simple test script (using SSL):

```python
import redis

HOST = "myapp-redis-dev-001.redis.cache.windows.net"
PORT = 6380
PASSWORD = "<your-primary-key>"

try:
    # Connect to Redis
    r = redis.Redis(host=HOST, port=PORT, password=PASSWORD, ssl=True)
    
    print(f"Connected to Redis cache: {HOST}")

    # Set a simple key-value
    r.set("greeting", "Hello from Azure Cache for Redis!")
    print("Set 'greeting' key.")

    # Retrieve the value
    result = r.get("greeting")
    if result:
        print(f"Value retrieved: {result.decode('utf-8')}")

    # Set an expiring key (e.g., for caching/sessions)
    r.setex("session:201", 60, "active_user_data")
    print("Set expiring session key.")

except Exception as e:
    print(f"Error connecting to Redis: {e}")
```

## Clean Up

Remove all deployed resources when finished:

```bash
az group delete --name rg-redis-dev --yes --no-wait
```

## Troubleshooting

| Error | Cause | Fix |
| ----- | ----- | --- |
| `NameAlreadyExists` | Cache name is taken globally | Change `redisCacheName` to something unique |
| `ConnectionRefused` | Connecting to non-SSL port | Use SSL port `6380` or enable non-SSL port `6379` |
| `Unauthorized` | Wrong access key | Verify `primaryKey` from portal or CLI |

## Security Best Practices

- Always use the **SSL/TLS endpoint** (Port 6380) for data-in-transit encryption.
- Disable the non-SSL port (`enableNonSslPort: false`).
- Rotate keys periodically via `az redis force-reboot` / regenerate keys.
- For production, use **Azure Virtual Network integration** (Premium SKU only) or **Private Link** (Basic/Standard/Premium).

## References

- [Azure Cache for Redis documentation](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/)
- [ARM template reference — Microsoft.Cache](https://learn.microsoft.com/en-us/azure/templates/microsoft.cache/redis)
- [Connect to Redis using Python](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-python-get-started)
