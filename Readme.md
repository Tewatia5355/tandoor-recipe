# Tandoor Deployment Guide for Azure with Automation

## Prerequisites

- Azure subscription with appropriate permissions
- Azure CLI installed locally
- Docker Desktop (for testing locally)
- Git installed
- Basic understanding of containers and Azure services

## Phase 1: Initial Azure Setup

### Step 1: Install and Configure Azure CLI

```bash
# Install Azure CLI (if not already installed)
# For Windows: Download from https://aka.ms/installazurecliwindows
# For macOS: brew install azure-cli
# For Linux: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login to Azure
az login

# Set your subscription (replace with your subscription ID)
az account set --subscription "your-subscription-id"
```

### Step 2: Create Resource Group and Basic Resources

```powershell
# Set variables (customize these values)
$RESOURCE_GROUP = "tandoor-rg"
$LOCATION = "northeurope"
$CONTAINER_APP_ENV = "tandoor-env"
$POSTGRES_SERVER = "tandoor-postgres-$(Get-Date -Format 'yyyyMMddHHmmss')"
$POSTGRES_DB = "tandoor"
$POSTGRES_USER = "tandooradmin"
$POSTGRES_PASSWORD = "********"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION
```

### Step 3: Register PostgreSQL Resource Provider

```powershell
# Register the PostgreSQL resource provider (required if not already registered)
az provider register --namespace Microsoft.DBforPostgreSQL
```

### Step 4: Create PostgreSQL Database

```powershell
# Create PostgreSQL server (using latest recommended version 16)
az postgres flexible-server create `
  --resource-group $RESOURCE_GROUP `
  --name $POSTGRES_SERVER `
  --location $LOCATION `
  --admin-user $POSTGRES_USER `
  --admin-password $POSTGRES_PASSWORD `
  --sku-name Standard_B1ms `
  --tier Burstable `
  --version 16 `
  --storage-size 32 `
  --public-access 0.0.0.0

# Create database
az postgres flexible-server db create `
  --resource-group $RESOURCE_GROUP `
  --server-name $POSTGRES_SERVER `
  --database-name $POSTGRES_DB

# Configure firewall to allow Azure services
az postgres flexible-server firewall-rule create `
  --resource-group $RESOURCE_GROUP `
  --name $POSTGRES_SERVER `
  --rule-name "AllowAzureServices" `
  --start-ip-address 0.0.0.0 `
  --end-ip-address 0.0.0.0
```

## Phase 2: Container Apps Setup

### Step 5: Create Container Apps Environment

```powershell
# Install Container Apps extension with latest features
az extension add --name containerapp --upgrade --allow-preview true

# Register required providers
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# Create Container Apps environment (simplified - Log Analytics workspace is created automatically)
az containerapp env create `
  --name $CONTAINER_APP_ENV `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION
```

### Step 6: Deploy Tandoor Container App

```powershell
# Generate a secure secret key
$SECRET_KEY = [System.Web.Security.Membership]::GeneratePassword(50, 10)

# Create the Tandoor container app
az containerapp create `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --environment $CONTAINER_APP_ENV `
  --image "vabene1111/recipes:latest" `
  --target-port 8080 `
  --ingress external `
  --min-replicas 1 `
  --max-replicas 3 `
  --cpu 1.0 `
  --memory 2.0Gi `
  --env-vars `
    SECRET_KEY="$SECRET_KEY" `
    DB_ENGINE="django.db.backends.postgresql" `
    POSTGRES_HOST="$POSTGRES_SERVER.postgres.database.azure.com" `
    POSTGRES_PORT="5432" `
    POSTGRES_USER="$POSTGRES_USER" `
    POSTGRES_PASSWORD="$POSTGRES_PASSWORD" `
    POSTGRES_DB="$POSTGRES_DB" `
    DEBUG="0" `
    ALLOWED_HOSTS="*" `
    TZ="Europe/Dublin" `
    ENABLE_SIGNUP="1" `
    GUNICORN_MEDIA="0"
```

## Phase 3: Storage and Media Files

### Step 7: Create Azure Storage for Media Files

```powershell
# Create storage account
$STORAGE_ACCOUNT = "tandoorstoragemedia"
az storage account create `
  --name $STORAGE_ACCOUNT `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --sku Standard_LRS

# Enable public blob access on the storage account
az storage account update `
  --name $STORAGE_ACCOUNT `
  --resource-group $RESOURCE_GROUP `
  --allow-blob-public-access true

# Create container for media files
az storage container create `
  --name "media" `
  --account-name $STORAGE_ACCOUNT `
  --public-access blob `
  --auth-mode login

# Get storage connection string
$STORAGE_CONNECTION_STRING = az storage account show-connection-string `
  --name $STORAGE_ACCOUNT `
  --resource-group $RESOURCE_GROUP `
  --output tsv
```

### Step 8: Update Container App with Storage

```powershell
# Get storage account key
$STORAGE_KEY = az storage account keys list --account-name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --query '[0].value' -o tsv

# Update the container app to include storage configuration
az containerapp update `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --set-env-vars `
    AZURE_ACCOUNT_NAME="$STORAGE_ACCOUNT" `
    AZURE_ACCOUNT_KEY="$STORAGE_KEY" `
    AZURE_CONTAINER="media"
```

## Phase 4: Automation Setup

### Step 9: Create GitHub Repository for Infrastructure

Create a new GitHub repository and add these files:

**`.github/workflows/deploy.yml`:**
```yaml
name: Deploy Tandoor to Azure

on:
  push:
    branches: [ main ]
  workflow_dispatch:

env:
  RESOURCE_GROUP: tandoor-rg
  CONTAINER_APP_NAME: tandoor-app

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Log in to Azure
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy to Container Apps
      run: |
        az containerapp update \
          --name $CONTAINER_APP_NAME \
          --resource-group $RESOURCE_GROUP \
          --image vabene1111/recipes:latest
```

**`docker-compose.override.yml`** (for local development):
```yaml
version: '3.8'
services:
  web_recipes:
    environment:
      - DEBUG=1
      - POSTGRES_HOST=db_recipes
    ports:
      - "8080:8080"
    volumes:
      - ./staticfiles:/opt/recipes/staticfiles
      - ./mediafiles:/opt/recipes/mediafiles
```

### Step 10: Set up GitHub Actions Secrets

1. Create Azure Service Principal:
```powershell
$subscriptionId = az account show --query id -o tsv
az ad sp create-for-rbac `
  --name "tandoor-github-actions" `
  --role contributor `
  --scopes "/subscriptions/$subscriptionId/resourceGroups/$RESOURCE_GROUP" `
  --sdk-auth
```

2. Copy the JSON output and add it as `AZURE_CREDENTIALS` secret in your GitHub repository settings.

### Step 11: Automated Backup Setup

```powershell
# Create backup storage account
$BACKUP_STORAGE = "tandoorstoragebackup"
az storage account create `
  --name $BACKUP_STORAGE `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --sku Standard_GRS
```

#### Logic App for Automated Backups

- Use Azure Portal's Logic App Designer for easiest setup.
- **Trigger:** Recurrence (e.g., daily)
- **Action 1:** List blobs in the source container (`media`) of `$STORAGE_ACCOUNT`.
- **Action 2:** For each blob:
    - Get blob content from source
    - Create blob with same name/content in `backupmedia` container of `$BACKUP_STORAGE`
- Establish two Azure Blob Storage connections (one for each storage account).
- Assign `Storage Blob Data Contributor` role to your user/service principal on both storage accounts, or use storage account keys for authentication.
- Test by uploading a file to the source container and running the Logic App.
- Monitor runs and set up alerts for failures.
- Consider enhancements: incremental backup, deletion sync, or backup versioning as needed.

> For infrastructure-as-code, you can export the Logic App definition as JSON or generate ARM/Bicep templates.

## Phase 5: Monitoring and Maintenance

### Step 12: Set up Application Insights

```powershell
# Create Application Insights
az monitor app-insights component create `
  --app "tandoor-insights" `
  --location $LOCATION `
  --resource-group $RESOURCE_GROUP `
  --application-type web `
  --kind web

# Get instrumentation key
$INSTRUMENTATION_KEY = az monitor app-insights component show `
  --app "tandoor-insights" `
  --resource-group $RESOURCE_GROUP `
  --query instrumentationKey `
  --output tsv

# Update container app with Application Insights
az containerapp update `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --set-env-vars `
    APPINSIGHTS_INSTRUMENTATIONKEY="$INSTRUMENTATION_KEY"
```

### Step 13: Configure Alerts

```powershell
# Get subscription ID
$subscriptionId = az account show --query id -o tsv

# Create alert rule for high CPU usage
az monitor metrics alert create `
  --name "tandoor-high-cpu" `
  --resource-group $RESOURCE_GROUP `
  --scopes "/subscriptions/$subscriptionId/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.App/containerApps/tandoor-app" `
  --condition "avg Percentage CPU > 80" `
  --description "Alert when CPU usage is over 80%" `
  --evaluation-frequency 5m `
  --window-size 15m `
  --severity 2

# Create alert rule for high memory usage
az monitor metrics alert create `
  --name "tandoor-high-memory" `
  --resource-group $RESOURCE_GROUP `
  --scopes "/subscriptions/$subscriptionId/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.App/containerApps/tandoor-app" `
  --condition "avg WorkingSetBytes > 1600000000" `
  --description "Alert when memory usage is over 1.6GB" `
  --evaluation-frequency 5m `
  --window-size 15m `
  --severity 2
```

## Phase 6: Access and Initial Setup

### Step 14: Get Application URL and Initial Configuration

```powershell
# Get the application URL
$TANDOOR_URL = az containerapp show `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --query properties.configuration.ingress.fqdn `
  --output tsv

Write-Host "Tandoor is available at: https://$TANDOOR_URL"
```

1. Navigate to your Tandoor URL
2. Create the first superuser account
3. Configure your recipe settings
4. Start importing your recipes!

## Maintenance Commands

### Update Tandoor to Latest Version:
```powershell
az containerapp update `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --image vabene1111/recipes:latest
```

### View Logs:
```powershell
# View recent logs
az containerapp logs show `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --tail 50

# Follow logs in real-time
az containerapp logs show `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --follow
```

### Scale Application:
```powershell
az containerapp update `
  --name "tandoor-app" `
  --resource-group $RESOURCE_GROUP `
  --min-replicas 2 `
  --max-replicas 5
```

## Cost Optimization Tips

1. **Use Burstable database tier** for small to medium workloads
2. **Set up auto-scaling** with appropriate min/max replicas
3. **Use Azure Storage lifecycle policies** for old backups
4. **Monitor costs** with Azure Cost Management
5. **Consider reserved instances** for predictable workloads

## Security Best Practices

1. **Enable HTTPS only** (automatically handled by Container Apps)
2. **Use Azure Key Vault** for secrets in production
3. **Configure network security groups** if needed
4. **Enable database SSL** connections
5. **Regular security updates** through automated deployments
6. **Use managed identities** where possible

## Troubleshooting

### Common Issues:
- **Database connection errors**: Check firewall rules and connection strings
- **Storage issues**: Verify storage account keys and container permissions
- **Memory issues**: Increase memory allocation or add more replicas
- **Slow performance**: Check database performance tier and consider scaling

### Useful Commands:
```powershell
# Check container app status and get URL
az containerapp show --name "tandoor-app" --resource-group $RESOURCE_GROUP --query "properties.configuration.ingress.fqdn" -o tsv

# Restart container app by updating with same image
az containerapp update --name "tandoor-app" --resource-group $RESOURCE_GROUP --image "vabene1111/recipes:latest"

# Check database metrics and status
az postgres flexible-server show --name $POSTGRES_SERVER --resource-group $RESOURCE_GROUP

# Check resource usage
az containerapp show --name "tandoor-app" --resource-group $RESOURCE_GROUP --query "properties.template.containers[0].resources"

# List all revisions
az containerapp revision list --name "tandoor-app" --resource-group $RESOURCE_GROUP -o table
```

## Additional Azure Setup and Troubleshooting (see Gpt.md for full details)

### Enable Required PostgreSQL Extensions

- Azure Postgres may require extensions like `pg_trgm` and `unaccent` for Tandoor.
- Enable them with:
  ```powershell
  az postgres flexible-server parameter set --resource-group $RESOURCE_GROUP --server-name $POSTGRES_SERVER --name azure.extensions --value "pg_trgm,unaccent"
  ```
- Then connect to the database using `psql` (replace values as needed):
  ```powershell
  psql "host=$POSTGRES_SERVER.postgres.database.azure.com port=5432 dbname=$POSTGRES_DB user=$POSTGRES_USER@$POSTGRES_SERVER sslmode=require"
  ```
- In the psql shell, run:
  ```sql
  CREATE EXTENSION pg_trgm;
  CREATE EXTENSION unaccent;
  ```

### Container App Shell Access

- For troubleshooting, connect to the running container shell:
  ```powershell
  az containerapp exec --name "tandoor-app" --resource-group $RESOURCE_GROUP
  ```

### Database Reset and Recovery

If migrations fail due to missing extensions or other issues, you may need to reset the Postgres database schema and re-run migrations. Here is the full process:

1. Connect to the running container shell:
   ```powershell
   az containerapp exec --name "tandoor-app" --resource-group $RESOURCE_GROUP
   ```
2. Open the database shell from inside the container:
   ```powershell
   python migrate.py dbshell
   ```
3. In the Postgres shell, run the following commands to drop and recreate the schema:
   ```sql
   DROP SCHEMA public CASCADE;
   CREATE SCHEMA public;
   GRANT ALL ON SCHEMA public TO postgres;
   GRANT ALL ON SCHEMA public TO public;
   \q
   ```
4. Re-run migrations and create the superuser:
   ```powershell
   python manage.py migrate
   python manage.py createsuperuser
   ```
5. Re-enable the required extensions (`pg_trgm` and `unaccent`) as described above before running migrations again.

This setup provides a production-ready Tandoor deployment with automated updates, monitoring, and backup capabilities.