# Azure Database for MySQL Flexible Server — ARM Template Setup

Deploy a fully parameterized Azure Database for MySQL (Flexible Server) using the two files in this repository:

| File | Purpose |
| ---- | ------- |
| `azuredeploy.json` | ARM template — resource definitions |
| `azuredeploy.parameters.json` | Parameter values for the deployment |

## Architecture Overview

```
Azure Resource Group
└── MySQL Flexible Server
    ├── Firewall Rule: AllowAll (0.0.0.0 - 255.255.255.255)
    └── MySQL Database (appdatabase)
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
| `serverName` | string | *(required)* | Globally unique MySQL server name |
| `administratorLogin` | string | `mysqladmin` | Server administrator login |
| `administratorLoginPassword` | securestring | *(required)* | Administrator password |
| `location` | string | RG location | Primary Azure region |
| `version` | string | `8.0.21` | MySQL version (`5.7`, `8.0.21`) |
| `skuName` | string | `Standard_B1ms` | VM SKU name |
| `skuTier` | string | `Burstable` | Compute tier (`Burstable`, `GeneralPurpose`, `MemoryOptimized`) |
| `storageSizeGB` | int | `20` | Storage size in GB |
| `storageIops` | int | `360` | Storage IOPS |
| `backupRetentionDays` | int | `7` | Days to keep backups (1-35) |
| `geoRedundantBackup` | string | `Disabled` | Geo-redundant backup mode |
| `highAvailabilityMode`| string | `Disabled` | High availability mode (`Disabled`, `ZoneRedundant`, `SameZone`) |
| `databaseName` | string | `appdatabase` | Name of the initial database |
| `allowAllIPs` | bool | `true` | Create firewall rule to allow all public IPs (for dev/test) |
| `tags` | object | `{environment, project, managedBy}` | Resource tags |

## Deployment

### Step 1 — Edit parameters

Open `azuredeploy.parameters.json` and update at minimum:

```json
"serverName": { "value": "YOUR-UNIQUE-SERVER-NAME" },
"administratorLoginPassword": { "value": "YOUR-SECURE-PASSWORD" }
```

### Step 2 — Create a Resource Group

```bash
az group create --name rg-mysqldb-dev --location eastus
```

### Step 3 — Validate the template (recommended)

```bash
# ensure you are in the mysql-db-setup directory
az deployment group validate \
  --resource-group rg-mysqldb-dev \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json
```

### Step 4 — Deploy

```bash
az deployment group create \
  --resource-group rg-mysqldb-dev \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json \
  --name mysql-deploy-$(date +%Y%m%d%H%M%S)
```

Deployment typically completes in **5–10 minutes**.

### Step 5 — Review outputs

```bash
# ensure to change the timestamp to the one you used in step 4
az deployment group show \
  --resource-group rg-mysqldb-dev \
  --name mysql-deploy-<timestamp> \
  --query properties.outputs
```

**Outputs provided:**

| Output key | Description |
| ---------- | ----------- |
| `mysqlServerName` | Deployed server name |
| `mysqlServerId` | Full Azure Resource ID |
| `mysqlServerEndpoint` | MySQL connection endpoint |
| `databaseName` | Database name |
| `administratorLogin` | Administrator login name |

> **Security note:** Passing the password directly in `azuredeploy.parameters.json` is okay for dev, but in production, fetch it from **Azure Key Vault**.

## Testing with Data

### Option A — MySQL CLI (Terminal)

Install MySQL client if not already installed.
Connect using the endpoint from the outputs.

```bash
mysql -h myapp-mysql-dev-001.mysql.database.azure.com -u mysqladmin -p
```
*(Enter the password when prompted)*

Once connected, verify the database and insert a record:

```sql
SHOW DATABASES;
USE appdatabase;

CREATE TABLE items (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  stock INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO items (name, price, stock) VALUES
  ('Wireless Headphones', 89.99, 150),
  ('USB-C Hub', 35.00, 320),
  ('Cloud Architecture Patterns', 49.99, 75);

SELECT * FROM items;
```

*to exit the mysql CLI, type `exit;` and press Enter*

### Option B — Python Testing (Programmatic)

Install the required library:

```bash
pip install mysql-connector-python
```

Run a simple test script:

```python
import mysql.connector
from mysql.connector import Error

HOST = "myapp-mysql-dev-001.mysql.database.azure.com"
USER = "mysqladmin"
PASSWORD = "<your-secure-password>"
DB = "appdatabase"

try:
    connection = mysql.connector.connect(
        host=HOST,
        user=USER,
        password=PASSWORD,
        database=DB,
        ssl_disabled=False
    )

    if connection.is_connected():
        print(f"Connected to MySQL server: {HOST}")
        cursor = connection.cursor()

        # Create table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS items (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                price DECIMAL(10,2) NOT NULL,
                stock INT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Insert data
        cursor.execute("""
            INSERT INTO items (name, price, stock)
            VALUES ('Wireless Mouse', 25.50, 100)
        """)
        connection.commit()
        print("Data inserted.")

        # Read data
        cursor.execute("SELECT * FROM items")
        records = cursor.fetchall()
        for row in records:
            print(row)

except Error as e:
    print(f"Error: {e}")
finally:
    if 'connection' in locals() and connection.is_connected():
        cursor.close()
        connection.close()
        print("MySQL connection closed.")
```

## Clean Up

Remove all deployed resources when finished:

```bash
az group delete --name rg-mysqldb-dev --yes --no-wait
```

## Troubleshooting

| Error | Cause | Fix |
| ----- | ----- | --- |
| `NameAlreadyExists` | Server name is taken globally | Change `serverName` to something unique |
| `ClientConnectionFailure` | Firewall blocking IP | Ensure `allowAllIPs` is true, or add your IP to the firewall rules |
| `AccessDenied` | Wrong password or user | Verify `administratorLogin` and `administratorLoginPassword` |
| `SkuNotAvailable` | Selected SKU not available in region | Try a different region or change `skuName` (e.g., to `Standard_D2ds_v4`) |

## Security Best Practices

- Do **NOT** store passwords in parameter files in production. Use **Azure Key Vault** references.
- Use **VNet integration** (private access) instead of public endpoints in production.
- Disable `allowAllIPs` and explicitly add required subnets or IP addresses to the firewall.
- Regularly rotate administrator passwords and use Entra ID (Azure AD) authentication when possible.

## References

- [Azure Database for MySQL Flexible Server documentation](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/)
- [ARM template reference — Microsoft.DBforMySQL](https://learn.microsoft.com/en-us/azure/templates/microsoft.dbformysql/flexibleservers)
- [Connect to Azure Database for MySQL Flexible Server using Python](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/connect-python)
