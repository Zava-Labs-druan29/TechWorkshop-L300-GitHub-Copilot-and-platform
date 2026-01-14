# Azure Infrastructure for ZavaStorefront

This directory contains the Bicep templates for deploying the ZavaStorefront application infrastructure to Azure.

## Architecture

The infrastructure includes:

- **Azure Container Registry (ACR)**: Stores Docker container images
- **Linux App Service Plan**: Hosts the web application
- **Linux App Service**: Runs the containerized .NET application
- **Application Insights**: Monitors application performance and logs
- **Log Analytics Workspace**: Stores monitoring data
- **Azure OpenAI**: Provides GPT-4 and Phi-3 models for AI capabilities
- **Managed Identity**: App Service uses system-assigned identity for secure authentication
- **RBAC**: App Service has AcrPull role to pull images from ACR without passwords

## Region

All resources are deployed to **westus3** region for GPT-4 and Phi model availability.

## Prerequisites

1. [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
2. [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
3. An Azure subscription

## Deployment

### Using Azure Developer CLI (Recommended)

1. **Login to Azure**:
   ```bash
   azd auth login
   ```

2. **Initialize the environment** (first time only):
   ```bash
   azd env new
   ```
   
   When prompted, provide an environment name (e.g., `dev`, `staging`, `prod`).

3. **Set the location** to westus3:
   ```bash
   azd env set AZURE_LOCATION westus3
   ```

4. **Provision and deploy**:
   ```bash
   azd up
   ```
   
   This will:
   - Create all Azure resources
   - Build the Docker container using ACR tasks (no local Docker required)
   - Push the image to ACR
   - Deploy to App Service
   - Configure Application Insights
   - Set up Azure OpenAI with GPT-4 and Phi-3 models

### Manual Deployment

If you prefer to deploy manually:

1. **Login to Azure**:
   ```bash
   az login
   ```

2. **Set subscription**:
   ```bash
   az account set --subscription <subscription-id>
   ```

3. **Deploy infrastructure**:
   ```bash
   az deployment sub create \
     --name zavastorefront \
     --location westus3 \
     --template-file infra/main.bicep \
     --parameters environmentName=dev location=westus3 principalId=$(az ad signed-in-user show --query id -o tsv)
   ```

4. **Build and push Docker image** (using ACR tasks - no local Docker needed):
   ```bash
   # Get the ACR name from deployment outputs
   ACR_NAME=$(az deployment sub show --name zavastorefront --query properties.outputs.AZURE_CONTAINER_REGISTRY_NAME.value -o tsv)
   
   # Build and push using ACR tasks
   az acr build --registry $ACR_NAME --image zavastorefront:latest --file Dockerfile .
   ```

5. **Configure and restart Web App**:
   ```bash
   # Get the web app name
   WEB_APP_NAME=$(az deployment sub show --name zavastorefront --query properties.outputs.WEB_APP_NAME.value -o tsv)
   RESOURCE_GROUP=$(az deployment sub show --name zavastorefront --query properties.outputs.AZURE_RESOURCE_GROUP.value -o tsv)
   
   # Restart to pull the new image
   az webapp restart --name $WEB_APP_NAME --resource-group $RESOURCE_GROUP
   ```

## Environment Variables

The following environment variables are automatically configured in the App Service:

- `APPLICATIONINSIGHTS_CONNECTION_STRING`: Connection string for Application Insights
- `AZURE_OPENAI_ENDPOINT`: Endpoint URL for Azure OpenAI service
- `DOCKER_REGISTRY_SERVER_URL`: URL of the Container Registry

## Accessing Resources

After deployment, you can access:

1. **Web Application**: Check the `WEB_APP_URI` output
2. **Application Insights**: View in Azure Portal
3. **Azure OpenAI**: Available with GPT-4 and Phi-3 deployments

## Monitoring

Application Insights is configured to collect:
- Request telemetry
- Dependency telemetry
- Exception telemetry
- Custom events and metrics

Access the monitoring dashboard in the Azure Portal.

## Security Features

- **No passwords**: App Service uses managed identity with RBAC
- **HTTPS only**: All traffic is encrypted
- **TLS 1.2+**: Minimum TLS version enforced
- **FTPS disabled**: Only secure protocols allowed
- **System-assigned managed identity**: For secure Azure resource access

## Cost Optimization

This is configured as a **dev environment** with cost-effective SKUs:
- App Service Plan: B1 (Basic)
- Container Registry: Basic
- Log Analytics: Pay-as-you-go

For production, consider upgrading to Premium SKUs for better performance and SLA.

## Clean Up

To delete all resources:

```bash
azd down
```

Or manually delete the resource group:

```bash
az group delete --name rg-<environmentName> --yes
```

## Troubleshooting

### Container not pulling from ACR

If the web app fails to pull the container:
1. Verify the managed identity has AcrPull role
2. Check the container image exists in ACR: `az acr repository list --name <acr-name>`
3. View app service logs: `az webapp log tail --name <app-name> --resource-group <rg-name>`

### Azure OpenAI not available

Ensure you're deploying to westus3 region where GPT-4 and Phi models are available.

### Build failures

Check ACR task logs:
```bash
az acr task logs --registry <acr-name>
```

## Additional Resources

- [Azure Developer CLI documentation](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- [Azure App Service documentation](https://docs.microsoft.com/azure/app-service/)
- [Azure Container Registry documentation](https://docs.microsoft.com/azure/container-registry/)
- [Azure OpenAI Service documentation](https://learn.microsoft.com/azure/cognitive-services/openai/)
