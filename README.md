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
|-------------|-------------|---------|
| `ACR_NAME` | ACR name (without .azurecr.io) | `myacrname` |
| `ACA_RG` | Azure resource group name | `aca-demo-rg` |
| `ACA_NAME` | Container app name | `flask-demo-app` |
| `ACA_ENV_NAME` | Container Apps environment name | `aca-demo-env` |
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

The workflow can be triggered in two ways:

1. **Manually**: Go to GitHub Actions tab and run the workflow manually using `workflow_dispatch`
2. **Automatically**: Push any changes to the repository (except `.md` files)

## Container App Descriptor

The deployment uses a YAML descriptor (`containerapp.yaml`) that defines:

- **Both containers**: Flask app (`qa-demo`) and nginx sidecar (`nginx-proxy`)
- **Resources**: CPU (0.25) and memory (0.5Gi) allocation for each container
- **Ingress**: External access through nginx on port 8080
- **Scaling**: Min 1 / Max 3 replicas with HTTP scaling rule (10 concurrent requests)
- **Configuration**: Single revision mode with auto transport

The GitHub Actions workflow uses `az acr build` to build and push images to ACR, then uses the official `azure/container-apps-deploy-action` to deploy. The workflow automatically replaces `{acr-name}` and `{image-tag}` placeholders in the descriptor with actual values.

## Access the Application

After deployment, the Container App will be accessible via the generated URL. Access it using:

- **URL**: `https://<your-app>.<region>.azurecontainerapps.io`
- **Username**: `admin`
- **Password**: `password123`

The `/health` endpoint is accessible without authentication.

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

- `/` - Main endpoint with service info (returns JSON with message, hostname, and version)
- `/health` - Health check endpoint (no authentication required, returns status)
- `/api/info` - Additional service information (returns app name, environment, and hostname)

## Local Testing

### Build and run locally:

```bash
# Build images
docker build -t flask-demo ./flask-app
docker build -t nginx-proxy ./nginx-sidecar

# Create a network
docker network create flask-network

# Run Flask container
docker run -d --name flask-app --network flask-network -p 5000:5000 flask-demo

# Run Nginx container
docker run -d --name nginx-proxy --network flask-network -p 8080:8080 nginx-proxy

# Test with authentication
curl -u admin:password123 http://localhost:8080

# Test health endpoint (no auth required)
curl http://localhost:8080/health

# Cleanup
docker stop flask-app nginx-proxy
docker rm flask-app nginx-proxy
docker network rm flask-network
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
    networks:
      - app-network

  nginx-proxy:
    build: ./nginx-sidecar
    ports:
      - "8080:8080"
    depends_on:
      - flask-app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
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
  --name $ACA_NAME \
  --resource-group $ACA_RG \
  --follow

# View specific container logs
az containerapp logs show \
  --name $ACA_NAME \
  --resource-group $ACA_RG \
  --container flask-app

az containerapp logs show \
  --name $ACA_NAME \
  --resource-group $ACA_RG \
  --container nginx-sidecar
```

## Cleanup

Remove all resources:

```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

## License

MIT