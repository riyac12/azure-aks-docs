---
title: Deploy and use Container Networking Agent on AKS
description: Learn how to deploy Container Networking Agent as an AKS extension to troubleshoot networking issues in Azure Kubernetes Service (AKS) clusters.
author: azure-networking
ms.author: azure-networking
ms.date: 02/23/2026
ms.topic: how-to
ms.service: azure-kubernetes-service
---

# Deploy and use Container Networking Agent on AKS

In this article, you learn how to deploy Container Networking Agent on your Azure Kubernetes Service (AKS) cluster, configure authentication and identity, and use the agent to troubleshoot networking issues.

Container Networking Agent is an AI-powered diagnostic assistant that runs as an in-cluster web application. You describe networking problems in natural language, and the agent runs diagnostic commands (`kubectl`, `cilium`, `hubble`) against your cluster and returns structured, evidence-backed reports with root cause analysis and remediation guidance.

Container Networking Agent helps you troubleshoot:

- DNS failures, including CoreDNS misconfigurations, network policies blocking DNS traffic, NodeLocal DNS issues, and Cilium FQDN egress restrictions.
- Packet drops, including NIC-level RX drops, kernel packet loss, socket buffer overflow, SoftIRQ saturation, and ring buffer exhaustion across cluster nodes.
- Kubernetes networking issues, including pod connectivity failures, service port misconfigurations, network policy conflicts, missing endpoints, and Hubble flow analysis.
- Cluster resource queries for quick answers about pods, services, deployments, nodes, and namespaces.

Container Networking Agent operates with read-only access to your cluster. It doesn't modify running workloads, configurations, or network policies. Remediation guidance is advisory only.

Container Networking Agent is deployed as an AKS extension (`microsoft.retinaagent`). It integrates with the Azure resource management plane and is managed through the `az k8s-extension` CLI commands.

**Supported regions:** centralus, eastus, eastus2euap, centralusueap, eastus2, uksouth, westus2.

## Prerequisites

- An active Azure subscription. If you don't have one, create a [free account](https://azure.microsoft.com/free/) before you begin.
- **Contributor** role on the target resource group.
- **User Access Administrator** role on the target resource group (required for RBAC role assignments).
- Permissions to create Azure OpenAI resources in your subscription.
- An AKS cluster with [workload identity](/azure/aks/workload-identity-overview) and [OIDC issuer](/azure/aks/use-oidc-issuer) enabled. The cluster should run a [supported Kubernetes version](/azure/aks/supported-kubernetes-versions).
- (Recommended) [Azure CNI powered by Cilium](/azure/aks/azure-cni-powered-by-cilium) with [Advanced Container Networking Services (ACNS)](/azure/aks/advanced-container-networking-services-overview) enabled, for full diagnostic capabilities including Hubble flow analysis and Cilium policy diagnostics.
- Minimum node size: `Standard_D4_v3` or equivalent (three nodes recommended).
- An [Azure OpenAI](/azure/ai-services/openai/) resource with a deployed model (GPT-4.1 or equivalent).
- Azure CLI version 2.77.0 or later. To find your version, run `az --version`. If you need to install or upgrade, see [Install the Azure CLI](/cli/azure/install-azure-cli).
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed. To install it using Azure CLI, run `az aks install-cli`.
- [jq](https://jqlang.github.io/jq/download/) for JSON processing.
- The AKS cluster must have outbound connectivity to the Azure OpenAI endpoint over HTTPS (port 443).
- The cluster must be able to pull container images from `acnpublic.azurecr.io`.

> [!TIP]
> Container Networking Agent works on clusters without Cilium or ACNS, but with reduced diagnostic capabilities. On non-ACNS clusters, the agent provides DNS, packet drop, and standard Kubernetes networking diagnostics. Hubble flow analysis and Cilium policy diagnostics aren't available.

## Deploy Container Networking Agent

### Step 1: Set environment variables

Set the following environment variables. Replace the placeholder values with your own.

```azurecli-interactive
export SUBSCRIPTION_ID="<your-subscription-id>"
export LOCATION="eastus2"
export RESOURCE_GROUP="<your-resource-group>"
export CLUSTER_NAME="<your-aks-cluster-name>"

az login
az account set --subscription $SUBSCRIPTION_ID
```

### Step 2: Enable workload identity and OIDC issuer

If your cluster doesn't already have workload identity and OIDC issuer enabled, enable them using the [`az aks update`](/cli/azure/aks#az-aks-update) command.

```azurecli-interactive
az aks update \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --enable-oidc-issuer \
    --enable-workload-identity
```

Get the cluster credentials using the [`az aks get-credentials`](/cli/azure/aks#az-aks-get-credentials) command.

```azurecli-interactive
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --overwrite-existing
```

### Step 3: Create an Azure OpenAI resource and deploy a model

#### [Azure CLI](#tab/step3-cli)

If you don't already have an Azure OpenAI resource, create one using the [`az cognitiveservices account create`](/cli/azure/cognitiveservices/account#az-cognitiveservices-account-create) command.

```azurecli-interactive
export OPENAI_SERVICE_NAME="<your-openai-service-name>"
export OPENAI_DEPLOYMENT_NAME="gpt-5"
export OPENAI_MODEL_NAME="gpt-5"
export OPENAI_MODEL_VERSION="2025-08-07"

az cognitiveservices account create \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --kind "OpenAI" \
    --sku "S0" \
    --custom-domain $OPENAI_SERVICE_NAME
```

Wait for provisioning to complete, then deploy the model using the [`az cognitiveservices account deployment create`](/cli/azure/cognitiveservices/account/deployment#az-cognitiveservices-account-deployment-create) command.

```azurecli-interactive
sleep 30

az cognitiveservices account deployment create \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --deployment-name $OPENAI_DEPLOYMENT_NAME \
    --model-name $OPENAI_MODEL_NAME \
    --model-version $OPENAI_MODEL_VERSION \
    --model-format OpenAI \
    --sku-name "GlobalStandard" \
    --sku-capacity 1000
```

Retrieve the endpoint URL using the [`az cognitiveservices account show`](/cli/azure/cognitiveservices/account#az-cognitiveservices-account-show) command.

```azurecli-interactive
export AZURE_OPENAI_ENDPOINT=$(az cognitiveservices account show \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "properties.endpoint" -o tsv)

echo "OpenAI Endpoint: $AZURE_OPENAI_ENDPOINT"
```

#### [Script](#tab/step3-script)

Use the `setup-azure-openai.sh` script to create the Azure OpenAI resource and deploy a model in a single step. The script handles resource group creation, service provisioning, model availability validation, quota checks, and deployment.

```azurecli-interactive
export OPENAI_SERVICE_NAME="<your-openai-service-name>"

./deployment/setup-azure-openai.sh \
    --resource-group "$RESOURCE_GROUP" \
    --service-name "$OPENAI_SERVICE_NAME" \
    --location "$LOCATION" \
    --model-name "gpt-5" \
    --model-version "2025-08-07" \
    --deployment-name "gpt-5" \
    --sku-name "GlobalStandard" \
    --sku-capacity 1000
```

The script generates an `azure-openai-config.env` file with all configuration values. After the script completes, set the endpoint variable:

```azurecli-interactive
export AZURE_OPENAI_ENDPOINT=$(az cognitiveservices account show \
    --name $OPENAI_SERVICE_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "properties.endpoint" -o tsv)

export OPENAI_DEPLOYMENT_NAME="gpt-5"
echo "OpenAI Endpoint: $AZURE_OPENAI_ENDPOINT"
```

Run `./deployment/setup-azure-openai.sh --help` to see all available options.

---

### Steps 4-6: Create a managed identity, assign RBAC roles, and configure federated credentials

#### [Azure CLI](#tab/step456-cli)

**Step 4: Create a managed identity**

Create a user-assigned managed identity for the agent using the [`az identity create`](/cli/azure/identity#az-identity-create) command.

```azurecli-interactive
export IDENTITY_NAME="container-networking-agent-reader-identity"

az identity create \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --tags purpose=k8s-workload-identity
```

Get the identity client ID and principal ID using the [`az identity show`](/cli/azure/identity#az-identity-show) command.

```azurecli-interactive
export IDENTITY_CLIENT_ID=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "clientId" -o tsv)

export IDENTITY_PRINCIPAL_ID=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "principalId" -o tsv)

export IDENTITY_RESOURCE_ID=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "id" -o tsv)

echo "Client ID: $IDENTITY_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"
```

**Step 5: Assign RBAC roles**

Assign the required roles to the managed identity using the [`az role assignment create`](/cli/azure/role/assignment#az-role-assignment-create) command. The managed identity needs the following roles:

| Role | Scope | Purpose |
|------|-------|---------|
| `Cognitive Services OpenAI User` | Azure OpenAI resource | Allows the agent to make inference calls to the deployed model. |
| `Azure Kubernetes Service Cluster User Role` | AKS cluster | Allows the agent to access AKS cluster properties. |
| `Azure Kubernetes Service Contributor Role` | AKS cluster | Allows the agent to read and write AKS cluster information for MCP AKS operations. |
| `Reader` | Resource group | Allows the agent to read resource group metadata. |

```azurecli-interactive
# Cognitive Services OpenAI User on the OpenAI resource
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Cognitive Services OpenAI User" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.CognitiveServices/accounts/$OPENAI_SERVICE_NAME"

# Azure Kubernetes Service Cluster User Role on the AKS cluster
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME"

# Azure Kubernetes Service Contributor Role on the AKS cluster
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Azure Kubernetes Service Contributor Role" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME"

# Reader on the resource group
az role assignment create \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --role "Reader" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
```

**Step 6: Configure federated credentials**

Link the managed identity to the Kubernetes service account used by the agent using the [`az identity federated-credential create`](/cli/azure/identity/federated-credential#az-identity-federated-credential-create) command.

```azurecli-interactive
export OIDC_ISSUER_URL=$(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query "oidcIssuerProfile.issuerUrl" -o tsv)

az identity federated-credential create \
    --name "container-networking-agent-k8s-fed-cred" \
    --identity-name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --issuer $OIDC_ISSUER_URL \
    --subject "system:serviceaccount:kube-system:retina-ai-agent-reader" \
    --audiences "api://AzureADTokenExchange"
```

Container Networking Agent uses [AKS workload identity](/azure/aks/workload-identity-overview) to authenticate to Azure OpenAI and other Azure services. At runtime, AKS automatically injects a federated token into the agent pod. The agent exchanges this token for a Microsoft Entra ID access token to call Azure OpenAI. This approach eliminates the need for secrets or connection strings in your cluster.

#### [Script](#tab/step456-script)

Use the `setup-managed-identity-federated.sh` script to create the managed identity, assign all required RBAC roles, and configure federated credentials in a single step.

**1. Create an environment file from the template:**

```azurecli-interactive
cp deployment/setup-managed-identity.env.template setup-managed-identity.env
```

**2. Edit `setup-managed-identity.env` with your values:**

```text
RESOURCE_GROUP="<your-resource-group>"
AKS_CLUSTER_NAME="<your-aks-cluster-name>"
LLM_NAME="<your-openai-service-name>"
LOCATION="eastus2"
IDENTITY_NAME="container-networking-agent-reader-identity"
SERVICE_ACCOUNT_NAMESPACE="kube-system"
SERVICE_ACCOUNT_NAME="retina-ai-agent-reader"
FEDERATED_CREDENTIAL_NAME="container-networking-agent-k8s-fed-cred"
SUBSCRIPTION_ID="<your-subscription-id>"
```

**3. Source the environment file and run the script:**

```azurecli-interactive
set -a && source setup-managed-identity.env && set +a
./deployment/setup-managed-identity-federated.sh
```

The script handles:

- Checking and enabling AKS workload identity and OIDC issuer
- Creating the user-assigned managed identity (idempotent)
- Assigning RBAC roles: `Cognitive Services OpenAI User`, `Azure Kubernetes Service Cluster User Role`, `Azure Kubernetes Service Contributor Role`, and `Reader`
- Creating the federated credential linking the identity to the Kubernetes service account
- Detecting running pod service accounts and reconciling annotations
- Generating a deployment environment file for subsequent steps

After the script completes, set the identity variables for use in later steps:

```azurecli-interactive
export IDENTITY_CLIENT_ID=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "clientId" -o tsv)

echo "Client ID: $IDENTITY_CLIENT_ID"
```

---

### Step 7: (Optional) Create an App Registration for Entra ID authentication

Container Networking Agent supports two methods for user sign-in:

| Method | Use case |
|--------|----------|
| Simple username login | Development and testing environments. |
| Microsoft Entra ID (MSAL) | Production environments with OAuth2/OIDC through an App Registration. |

#### [Azure CLI](#tab/step7-cli)

To enable Microsoft Entra ID user authentication in production, create an App Registration using the [`az ad app create`](/cli/azure/ad/app#az-ad-app-create) command.

```azurecli-interactive
export APP_DISPLAY_NAME="container-networking-agent-oauth2-user-auth"
export APP_HOST="<your-app-hostname>"
export APP_REDIRECT_URI="https://${APP_HOST}/auth/callback"

export APP_ID=$(az ad app create \
    --display-name $APP_DISPLAY_NAME \
    --web-redirect-uris $APP_REDIRECT_URI \
    --enable-id-token-issuance true \
    --query "appId" -o tsv)

export TENANT_ID=$(az account show --query tenantId -o tsv)
```

Add the required Microsoft Graph delegated permissions (`openid`, `profile`, `User.Read`, `offline_access`) using the [`az ad app permission add`](/cli/azure/ad/app/permission#az-ad-app-permission-add) command.

```azurecli-interactive
az ad app permission add \
    --id $APP_ID \
    --api 00000003-0000-0000-c000-000000000000 \
    --api-permissions \
        e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope \
        14dad69e-099b-42c9-810b-d002981feec1=Scope \
        37f7f235-527c-4136-accd-4a02d197296e=Scope \
        7427e0e9-2fba-42fe-b0c0-848c9e6a8182=Scope

echo "App Registration Client ID: $APP_ID"
echo "Tenant ID: $TENANT_ID"
```

#### [Script](#tab/step7-script)

Use the `setup-azure-app-registration.sh` script to create the App Registration, configure redirect URIs, add federated credentials for workload identity, and assign Microsoft Graph and AKS delegated permissions in a single step.

**1. Set the required environment variables:**

```azurecli-interactive
export RESOURCE_GROUP="<your-resource-group>"
export AKS_CLUSTER_NAME="<your-aks-cluster-name>"
export APP_DISPLAY_NAME="container-networking-agent-oauth2-user-auth"
export APP_REDIRECT_URI="https://<your-app-hostname>/auth/callback"
export SERVICE_MANAGEMENT_REFERENCE="<your-service-management-reference>"
export SERVICE_ACCOUNT_NAMESPACE="kube-system"
export SERVICE_ACCOUNT_NAME="retina-ai-agent-reader"
```

**2. Run the script:**

```azurecli-interactive
./deployment/setup-azure-app-registration.sh
```

The script handles:

- Creating or reusing an existing App Registration by display name
- Configuring redirect and logout URLs
- Creating a workload identity federated credential for the AKS service account
- Assigning Microsoft Graph delegated permissions (`openid`, `profile`, `User.Read`, `offline_access`)
- Assigning AKS server delegated permissions
- Optionally granting admin consent (set `APP_GRANT_GRAPH_ADMIN_CONSENT=true`)

After the script completes, it outputs the App Registration Client ID and Tenant ID.

---

### Step 8: Install the extension

Install the Container Networking Agent AKS extension using the [`az k8s-extension create`](/cli/azure/k8s-extension#az-k8s-extension-create) command. Use the command that matches your cluster configuration.

**For clusters with ACNS and Cilium dataplane:**

```azurecli-interactive
export TENANT_ID=$(az account show --query tenantId -o tsv)

az k8s-extension create \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name container-networking-agent \
    --extension-type microsoft.retinaagent \
    --scope cluster \
    --release-train dev \
    --configuration-settings config.AZURE_CLIENT_ID=$IDENTITY_CLIENT_ID \
    --configuration-settings config.AZURE_TENANT_ID=$TENANT_ID \
    --configuration-settings config.AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID \
    --configuration-settings config.AKS_CLUSTER_NAME=$CLUSTER_NAME \
    --configuration-settings config.AKS_RESOURCE_GROUP=$RESOURCE_GROUP \
    --configuration-settings config.ENTRA_TENANT_ID=$TENANT_ID \
    --configuration-settings config.AZURE_OPENAI_DEPLOYMENT=$OPENAI_DEPLOYMENT_NAME \
    --configuration-settings config.AZURE_OPENAI_API_VERSION=2025-03-01-preview \
    --configuration-settings config.AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT
```

**For clusters without ACNS (Hubble disabled):**

```azurecli-interactive
export TENANT_ID=$(az account show --query tenantId -o tsv)

az k8s-extension create \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name container-networking-agent \
    --extension-type microsoft.retinaagent \
    --scope cluster \
    --release-train dev \
    --configuration-settings config.AZURE_CLIENT_ID=$IDENTITY_CLIENT_ID \
    --configuration-settings config.AZURE_TENANT_ID=$TENANT_ID \
    --configuration-settings config.AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID \
    --configuration-settings config.AKS_CLUSTER_NAME=$CLUSTER_NAME \
    --configuration-settings config.AKS_RESOURCE_GROUP=$RESOURCE_GROUP \
    --configuration-settings config.ENTRA_TENANT_ID=$TENANT_ID \
    --configuration-settings config.AZURE_OPENAI_DEPLOYMENT=$OPENAI_DEPLOYMENT_NAME \
    --configuration-settings config.AZURE_OPENAI_API_VERSION=2025-03-01-preview \
    --configuration-settings config.AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT \
    --configuration-settings hubble.enabled=false \
    --configuration-settings config.AKS_MCP_ENABLED_COMPONENTS=kubectl
```

> [!NOTE]
> On clusters without ACNS, set `hubble.enabled=false` and `config.AKS_MCP_ENABLED_COMPONENTS=kubectl`. The agent still provides DNS, packet drop, and standard Kubernetes networking diagnostics. Hubble flow analysis and Cilium policy diagnostics aren't available.

### Step 9: Verify the extension installation

Check the extension provisioning state using the [`az k8s-extension show`](/cli/azure/k8s-extension#az-k8s-extension-show) command.

```azurecli-interactive
az k8s-extension show \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name container-networking-agent \
    --query "provisioningState" -o tsv
```

The output should show `Succeeded`.

Verify the agent pods are running:

```azurecli-interactive
kubectl get pods -n kube-system | grep retina-ai-agent
```

You should see one or more pods in `Running` status.

### Step 10: Access the agent

Forward the service port to your local machine:

```azurecli-interactive
kubectl port-forward svc/retina-ai-agent-service -n kube-system 8080:80
```

Open your browser and go to `http://localhost:8080`. Sign in and start asking networking questions.

## Validate the deployment

After you deploy Container Networking Agent, verify that the deployment is healthy.

### Check pod status

```azurecli-interactive
kubectl get pods -n kube-system | grep retina-ai-agent
```

**Expected output:** One or more pods in `Running` status with `1/1` ready containers.

### Check health endpoints

Forward the service port if you haven't already:

```azurecli-interactive
kubectl port-forward svc/retina-ai-agent-service -n kube-system 8080:80
```

Check the health, readiness, and liveness endpoints:

```azurecli-interactive
# Health check
curl http://localhost:8080/health

# Readiness probe
curl http://localhost:8080/ready

# Liveness probe
curl http://localhost:8080/live
```

All three endpoints should return HTTP 200 responses.

### Check pod logs

```azurecli-interactive
kubectl logs -n kube-system -l app=retina-ai-agent --tail=100
```

A healthy deployment shows:

- Successful Azure OpenAI connectivity validation.
- Agent warmup pool initialization (default: three pre-warmed agents).
- No authentication or connection errors.

### Verify extension state

Verify the extension state in Azure using the [`az k8s-extension show`](/cli/azure/k8s-extension#az-k8s-extension-show) command.

```azurecli-interactive
az k8s-extension show \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name container-networking-agent \
    --query "{provisioningState:provisioningState, version:version}" -o table
```

The `provisioningState` should show `Succeeded` with Version number.

### Verify service account

Confirm the service account exists and has the correct workload identity annotation:

```azurecli-interactive
kubectl get serviceaccount retina-ai-agent-reader -n kube-system -o yaml
```

The annotation `azure.workload.identity/client-id` should match your managed identity client ID.

## Use Container Networking Agent

After deployment and validation, you can access the agent through the web chat interface.

### Access the chat interface

1. Forward the service port:

   ```azurecli-interactive
   kubectl port-forward svc/retina-ai-agent-service -n kube-system 8080:80
   ```

1. Open `http://localhost:8080` in your browser.

1. Sign in using either simple username login (development) or Microsoft Entra ID (production), depending on your configuration.

### Start a diagnostic conversation

Type a question or describe a networking problem in the chat input. The agent follows a structured diagnostic workflow:

1. **Classify**: Determines the issue category (DNS, connectivity, network policy, service routing, or packet drops).
1. **Collect evidence**: Runs the appropriate diagnostic commands against your cluster.
1. **Analyze**: Examines the collected evidence to identify anomalies and root causes.
1. **Report**: Returns a structured report with evidence tables, root cause analysis, and remediation commands.

### Sample prompts

| Scenario | Prompt |
|----------|--------|
| DNS failure | *"A pod in namespace `my-app` cannot resolve any DNS names"* |
| Packet drops | *"I see packet drops on node `aks-nodepool1-12345678-vmss000000`"* |
| Service unreachable | *"My client pod cannot connect to the backend-service in namespace `production`"* |
| Network policy blocking traffic | *"Pods in namespace `frontend` cannot communicate with the `backend` namespace"* |
| Cluster-wide DNS failure | *"All DNS is broken in the cluster"* |
| Proactive health check | *"Check network health on node `my-node`"* |

### Diagnostic workflows

- **DNS troubleshooting**: The agent checks CoreDNS pod health, service endpoints, CoreDNS configuration (including custom ConfigMaps), NodeLocal DNS status, DNS resolution from multiple paths, and network policies that might block DNS traffic.
- **Packet drop analysis**: The agent deploys a lightweight debug DaemonSet to collect host-level network statistics. It examines NIC ring buffer utilization, kernel softnet statistics, per-CPU SoftIRQ distribution, socket buffer saturation, and network interface statistics. Delta measurements detect active drops versus historical counters.
- **Kubernetes networking diagnostics**: The agent examines pod status and scheduling, service configuration and endpoint registration, network policies (both Kubernetes and Cilium), Hubble flows, and service-to-pod port mapping.

### Diagnostic output

Each diagnostic response includes:

- A summary of the issue and its status.
- An evidence table showing each check, its result, and whether it passed or failed.
- Analysis of what's working and what's broken.
- Root cause identification with specific evidence citations.
- Exact commands to fix the issue and verify the fix.

### Session and conversation limits

| Limit | Default |
|-------|---------|
| Chat context window | ~15 exchanges |
| Messages per conversation | 100 |
| Conversations per user | 20 |
| Session idle timeout | 30 minutes |
| Session absolute timeout | 8 hours |

Start a new conversation for unrelated issues to keep context fresh.

## Cluster access and security

Container Networking Agent uses a dedicated service account (`retina-ai-agent-reader`) with a read-only ClusterRole in the `kube-system` namespace. The RBAC configuration follows the principle of least privilege:

- **Read access** to core Kubernetes resources: Pods, Services, Nodes, Namespaces, ConfigMaps, Events, Deployments, ReplicaSets, DaemonSets, StatefulSets, Ingresses, NetworkPolicies, Endpoints, EndpointSlices, PersistentVolumes, and PersistentVolumeClaims.
- **Read access** to Cilium CRDs: CiliumNetworkPolicies, CiliumEndpoints, CiliumIdentities, CiliumLoadBalancerIPPools, CiliumL2AnnouncementPolicies, and other Cilium resources.
- **Read access** to metrics: Node and Pod metrics via metrics-server.
- **Limited exec access**: `pods/exec` is permitted only for diagnostic commands (Cilium status and endpoint information).
- **No write access**: The agent can't create, update, or delete any cluster resource.

The agent pod makes outbound HTTPS calls to your Azure OpenAI endpoint. If you use egress restrictions (network policies, Azure Firewall, or NSGs), allow outbound traffic from the `kube-system` namespace to your Azure OpenAI endpoint on port 443.

For packet drop diagnostics, the agent deploys a lightweight debug DaemonSet (`retina-debug-daemonset`) in `kube-system` that requires `hostNetwork` and `NET_ADMIN` capabilities. This DaemonSet is shared across diagnostic sessions and cleaned up automatically.

The agent doesn't persist diagnostic data externally. Session data (chat history, agent assignments) is stored in-memory within the pod and is lost if the pod restarts.

## Manage the extension

### Update the extension

Update the extension to a new version using the [`az k8s-extension update`](/cli/azure/k8s-extension#az-k8s-extension-update) command.

```azurecli-interactive
az k8s-extension update \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name container-networking-agent \
    --version <new-version>
```

### Uninstall the extension

Remove the extension using the [`az k8s-extension delete`](/cli/azure/k8s-extension#az-k8s-extension-delete) command.

```azurecli-interactive
az k8s-extension delete \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --cluster-type managedClusters \
    --name container-networking-agent
```

## Troubleshoot

### Identity or permission errors

**Symptom:** The agent pod starts but returns `401 Unauthorized` or `403 Forbidden` errors when processing requests. Pod logs show authentication or authorization failures.

**Cause:** The managed identity is missing required RBAC role assignments, or the federated credential is misconfigured.

**Resolution:**

1. Verify the managed identity has the correct role assignments:

   ```azurecli-interactive
   az role assignment list --assignee <identity-principal-id> --all -o table
   ```

   Confirm that `Cognitive Services OpenAI User`, `Azure Kubernetes Service Cluster User Role`, and `Reader` roles are assigned.

1. Verify workload identity is enabled on the cluster:

   ```azurecli-interactive
   az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME \
       --query "securityProfile.workloadIdentity.enabled"
   ```

1. Verify the federated credential subject matches the service account:

   ```azurecli-interactive
   az identity federated-credential list \
       --identity-name $IDENTITY_NAME \
       --resource-group $RESOURCE_GROUP
   ```

   The `subject` should be `system:serviceaccount:kube-system:retina-ai-agent-reader`.

1. Verify the service account annotation:

   ```azurecli-interactive
   kubectl get serviceaccount retina-ai-agent-reader -n kube-system -o yaml
   ```

   The `azure.workload.identity/client-id` annotation must match your managed identity's client ID.

### Agent not running or crashing

**Symptom:** The agent pod is in `CrashLoopBackOff`, `Error`, or `Pending` state.

**Cause:** Misconfiguration, missing Azure OpenAI connectivity, or insufficient cluster resources.

**Resolution:**

1. Check pod events:

   ```azurecli-interactive
   kubectl describe pod -n kube-system -l app=retina-ai-agent
   ```

1. Check pod logs:

   ```azurecli-interactive
   kubectl logs -n kube-system -l app=retina-ai-agent --tail=200
   ```

1. Verify the Azure OpenAI endpoint is reachable from the cluster. If you use network policies or firewalls, ensure outbound HTTPS traffic to the OpenAI endpoint is allowed.

1. Verify the configuration values in the ConfigMap:

   ```azurecli-interactive
   kubectl get configmap -n kube-system -l app=retina-ai-agent -o yaml
   ```

   Confirm that `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT`, and `AZURE_CLIENT_ID` have correct values.

### Connectivity or network policy issues

**Symptom:** The agent pod is running but can't connect to Azure OpenAI. Logs show connection timeouts or SSL errors.

**Cause:** Network policies, Azure Firewall rules, or NSG rules are blocking outbound traffic from the `kube-system` namespace.

**Resolution:**

1. Verify the Azure OpenAI endpoint is accessible from the pod:

   ```azurecli-interactive
   kubectl exec -n kube-system <pod-name> -- curl -s -o /dev/null -w "%{http_code}" \
       $AZURE_OPENAI_ENDPOINT
   ```

1. Check if network policies are blocking outbound traffic:

   ```azurecli-interactive
   kubectl get networkpolicies -n kube-system
   ```

1. If you use Azure Firewall or NSGs, ensure outbound HTTPS (port 443) is allowed to `*.openai.azure.com`.

### Extension installation fails

**Symptom:** The `az k8s-extension create` command fails or the extension provisioning state shows `Failed`.

**Cause:** Unsupported region, missing cluster features, or insufficient permissions.

**Resolution:**

1. Check the extension provisioning state:

   ```azurecli-interactive
   az k8s-extension show \
       --cluster-name $CLUSTER_NAME \
       --resource-group $RESOURCE_GROUP \
       --cluster-type managedClusters \
       --name container-networking-agent \
       --query "provisioningState"
   ```

1. Verify your cluster is in a supported region: centralus, eastus, eastus2euap, centralusueap, eastus2, uksouth, westus2.

1. Verify your cluster has workload identity and OIDC issuer enabled.

1. Check that you have `Contributor` and `User Access Administrator` roles on the resource group.

### Hubble commands fail

**Symptom:** The agent reports errors when running Hubble-related diagnostics, or Hubble flow analysis is unavailable.

**Cause:** The cluster doesn't have ACNS enabled, or the Cilium dataplane isn't configured.

**Resolution:**

- If your cluster doesn't use ACNS, deploy with `hubble.enabled=false` and `config.AKS_MCP_ENABLED_COMPONENTS=kubectl`. The agent still provides DNS, packet drop, and standard Kubernetes networking diagnostics.
- To enable Hubble, your cluster must use [Azure CNI powered by Cilium](/azure/aks/azure-cni-powered-by-cilium) with [Advanced Container Networking Services (ACNS)](/azure/aks/advanced-container-networking-services-overview) enabled.

### Debug DaemonSet persists after crash

**Symptom:** The `retina-debug-daemonset` DaemonSet remains in `kube-system` after a diagnostic session.

**Cause:** The Container Networking Agent pod crashed unexpectedly during a packet drop diagnostic.

**Resolution:**

Manually delete the DaemonSet:

```azurecli-interactive
kubectl delete ds retina-debug-daemonset -n kube-system
```

## Next steps

- [Container Networking Agent overview](./container-networking-agent-overview.md)
- [RBAC configuration for Container Networking Agent](../helm/container-networking-agent/README-RBAC.md)
- [Conversation limits](./CONVERSATION_LIMITS.md)
- [Agent architecture design](./AGENT_ARCHITECTURE_DESIGN.md)
