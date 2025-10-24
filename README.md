# Azure Container Apps Demo

This project demonstrates deploying a multi-container Azure Container App with:
- **Flask Backend**: A simple Python Flask application
- **Nginx Sidecar**: Nginx reverse proxy with HTTP Basic Authentication

## Architecture

```
Internet → Nginx (Port 8080) → Flask App (Port 5000)
           [Basic Auth]
```

The nginx container acts as a reverse proxy and adds basic authentication before requests reach the Flask application.

## Prerequisites

- Azure subscription
- Azure Container Registry (ACR)
- Azure Container Apps Environment
- GitHub repository with Actions enabled

## Project Structure

```
.
├── flask-app/
│   ├── app.py              # Flask application
│   ├── requirements.txt    # Python dependencies
│   └── Dockerfile          # Flask container image
├── nginx-sidecar/
│   ├── nginx.conf          # Nginx configuration
│   ├── .htpasswd           # Basic auth credentials
│   └── Dockerfile          # Nginx container image
├── containerapp.yaml       # Azure Container App descriptor
└── .github/
    └── workflows/
        └── deploy.yml      # GitHub Actions workflow
```

## Setup Instructions

### 1. Azure Resources

Create the required Azure resources:

```bash
# Set variables
RESOURCE_GROUP="aca-demo-rg"
LOCATION="eastus"
ACR_NAME="your-acr-name"  # Must be globally unique
CONTAINERAPPS_ENV="aca-demo-env"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create container registry (if not exists)
az acr create --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic \
  --admin-enabled true

# Create Container Apps environment
az containerapp env create \
  --name $CONTAINERAPPS_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

### 2. Enable Managed Identity (Recommended)

Enable managed identity for your Container App to pull images from ACR:

```bash
# Enable managed identity
az containerapp identity assign \
  --name <your-app-name> \
  --resource-group $RESOURCE_GROUP \
  --system-assigned

# Get the principal ID
PRINCIPAL_ID=$(az containerapp identity show \
  --name <your-app-name> \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Get ACR resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

# Assign AcrPull role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role AcrPull \
  --scope $ACR_ID

# Configure the container app to use managed identity
az containerapp registry set \
  --name <your-app-name> \
  --resource-group $RESOURCE_GROUP \
  --server $ACR_NAME.azurecr.io \
  --identity system
```

### 3. Configure GitHub Secrets

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

| Secret Name | Description | Example |
|-------------|-------------|---------||
| `AZURE_CONTAINER_REGISTRY` | ACR name (without .azurecr.io) | `myacrname` |
| `AZURE_RESOURCE_GROUP` | Azure resource group name | `aca-demo-rg` |
| `CONTAINER_APP_NAME` | Container app name | `flask-demo-app` |
| `AZURE_CREDENTIALS` | Service principal credentials (JSON) | See below |

#### Create Azure Service Principal

```bash
az ad sp create-for-rbac \
  --name "github-actions-aca-demo" \
  --role contributor \
  --scopes /subscriptions/{subscription-id}/resourceGroups/$RESOURCE_GROUP \
  --sdk-auth
```

Copy the entire JSON output to the `AZURE_CREDENTIALS` secret.

### 4. Deploy

Push to the `main` branch or manually trigger the workflow:

```bash
git add .
git commit -m "Initial setup"
git push origin main
```

Or trigger manually from GitHub Actions tab.

## Container App Descriptor

The deployment uses a YAML descriptor (`containerapp.yaml`) similar to Kubernetes deployments. This descriptor defines:

- **Both containers**: Flask app and nginx sidecar
- **Resources**: CPU and memory allocation for each container
- **Ingress**: External access through nginx on port 8080
- **Scaling**: Min/max replicas and scaling rules

The GitHub Actions workflow uses the official `azure/container-apps-deploy-action` for simplified deployment and automatically replaces placeholders in the descriptor with actual values.

## Access the Application

After deployment, the workflow will output the Container App URL. Access it using:

- **URL**: `https://<your-app>.region.azurecontainerapps.io`
- **Username**: `admin`
- **Password**: `password123`

### Change Default Credentials

To change the default credentials, regenerate the `.htpasswd` file:

```bash
# Install htpasswd (if not available)
# Ubuntu/Debian: apt-get install apache2-utils
# macOS: brew install httpd

# Generate new credentials
htpasswd -nb your-username your-password > nginx-sidecar/.htpasswd
```

## API Endpoints

- `/` - Main endpoint with service info
- `/health` - Health check (no authentication required)
- `/api/info` - Additional service information

## Local Testing

### Build and run locally:

```bash
# Build images
docker build -t flask-demo ./flask-app
docker build -t nginx-proxy ./nginx-sidecar

# Run Flask container
docker run -d --name flask-app -p 5000:5000 flask-demo

# Run Nginx container
docker run -d --name nginx-proxy -p 8080:8080 --network container:flask-app nginx-proxy

# Test
curl -u admin:password123 http://localhost:8080
```

### Using Docker Compose:

Create a `docker-compose.yml` file:

```yaml
version: '3.8'
services:
  flask-app:
    build: ./flask-app
    ports:
      - "5000:5000"
    environment:
      - ENVIRONMENT=development

  nginx-proxy:
    build: ./nginx-sidecar
    ports:
      - "8080:8080"
    depends_on:
      - flask-app
    network_mode: "service:flask-app"
```

Run with:
```bash
docker-compose up --build
```

## Monitoring and Logs

View logs in Azure:

```bash
# Stream logs
az containerapp logs show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --follow

# View specific container logs
az containerapp logs show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --container flask-demo

az containerapp logs show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --container nginx-sidecar
```

## Cleanup

Remove all resources:

```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

## License

MIT