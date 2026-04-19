---
title: Concepts - Access and identity in Azure Kubernetes Service (AKS)
description: Learn the four identity scenarios in Azure Kubernetes Service (AKS) — control-plane authentication and authorization, cluster identity, and workload identity — and where to find the right deep-dive doc for each.
ms.topic: concept-article
ms.subservice: aks-security
ms.date: 04/18/2026
author: shashankbarsin
ms.author: shasb
ai-usage: ai-assisted

# Customer intent: As a Kubernetes administrator, I want a clear orientation to the identity scenarios in AKS so that I can pick the right authentication and authorization model for each one.
---

# Access and identity options for Azure Kubernetes Service (AKS)

AKS uses identity in four distinct scenarios. Each scenario answers a different question and has its own configuration model. This article gives a brief introduction to each and points to the deep-dive documentation.

## The four identity scenarios in AKS

| Scenario | Question it answers | Deep-dive docs |
|---|---|---|
| **A. Control-plane authentication** | Who is the caller hitting the Kubernetes API? | [Microsoft Entra integration](#microsoft-entra-integration), [external identity providers](external-identity-provider-authentication-overview.md) |
| **B. Control-plane authorization** | What is the caller allowed to do once authenticated? | [Kubernetes API authorization concepts](concepts-authentication-authorization.md) |
| **C. Cluster identity (cluster → Azure)** | How does the AKS cluster act on Azure to manage resources on your behalf? | [Managed identities in AKS](use-managed-identity.md) |
| **D. Workload identity (pod → Azure)** | How do pods authenticate to Azure services such as Key Vault or Storage? | [Microsoft Entra Workload ID overview](workload-identity-overview.md) |

The rest of this article gives a brief orientation to each scenario.

## A. Control-plane authentication

Control-plane authentication establishes the identity of a user or service principal calling the Kubernetes API server. AKS supports:

* **Microsoft Entra ID (recommended).** Use Entra ID identities and groups to sign in to the cluster. AKS-managed Entra integration provisions and rotates the integration on your behalf. To enable, see [Use AKS-managed Microsoft Entra integration](entra-id-control-plane-authentication.md).
* **Local accounts.** A built-in cluster admin certificate that bypasses Entra ID. We recommend disabling local accounts in production. See [Manage local accounts](local-accounts.md).
* **External identity providers.** Use an OIDC-compliant identity provider other than Microsoft Entra ID. See [External identity provider authentication](external-identity-provider-authentication-overview.md).

<a name='azure-ad-integration'></a>

### Microsoft Entra integration

Microsoft Entra ID is a cloud identity service that combines directory services, application access management, and identity protection. With AKS-managed Entra integration, you can grant Entra users or groups access to Kubernetes resources within a namespace or across the cluster.

![Microsoft Entra integration with AKS clusters](media/concepts-identity/aad-integration.png)

Authentication uses OpenID Connect on top of OAuth 2.0. The Kubernetes API server validates incoming tokens through a webhook against Microsoft Entra ID. The high-level flow is:

1. `kubectl` signs in the user with the Microsoft Entra client application.
1. Microsoft Entra ID issues an access token.
1. `kubectl` sends the token to the API server.
1. The API server's authentication webhook verifies the token signature against Microsoft Entra public signing keys.
1. The API server makes an authorization decision (see [Kubernetes API authorization concepts](concepts-authentication-authorization.md)).

For setup, see [Use AKS-managed Microsoft Entra integration](entra-id-control-plane-authentication.md). For Conditional Access and Privileged Identity Management with cluster access, see [Cluster and node access control with Conditional Access](access-control-managed-azure-ad.md) and [Cluster and node access control with PIM](privileged-identity-management.md).

## B. Control-plane authorization

After a caller is authenticated, AKS authorizes the request using one (or both) of two models:

* **Kubernetes RBAC.** The native Kubernetes `Role` / `ClusterRole` / `RoleBinding` model evaluated by the API server. Permissions live in the cluster as Kubernetes manifests.
* **Microsoft Entra ID authorization.** An AKS authorization webhook delegates authorization decisions to Microsoft Entra ID using Azure role assignments, optionally extended with Azure ABAC conditions. Permissions are managed centrally in Microsoft Entra ID and can govern many clusters from a single role assignment at subscription, management group, or resource group scope.

For a side-by-side comparison and guidance on when to use each model, see [Kubernetes API authorization concepts](concepts-authentication-authorization.md).

<a name='kubernetes-rbac'></a>
<a name='azure-rbac-for-kubernetes-authorization'></a>
<a name='azure-rbac-to-authorize-access-to-the-aks-resource'></a>

### Authorization for the AKS resource (Azure Resource Manager)

In addition to authorizing calls to the Kubernetes API, you also need to authorize calls to Azure Resource Manager that manage the AKS resource itself — for example, scaling or upgrading the cluster, or pulling the `kubeconfig`. This is standard Azure RBAC against the `Microsoft.ContainerService` resource provider, separate from authorizing the Kubernetes API. See [Limit access to the cluster configuration file](control-kubeconfig-access.md) and the built-in roles in [Azure built-in roles](/azure/role-based-access-control/built-in-roles#containers).

## C. Cluster identity (cluster → Azure)

AKS clusters use Azure managed identities to act on Azure resources on your behalf — for example, to create load balancers, attach disks, or pull images from Azure Container Registry. The main identities are:

* **Control-plane identity.** Used by the cluster control plane to manage Azure resources for the cluster.
* **Kubelet identity.** Used by the kubelet on each node to authenticate to services such as Azure Container Registry.
* **Add-on identities.** Some AKS add-ons use their own managed identities.

For details on each identity type and how to use system-assigned vs user-assigned identities, see [Managed identities in AKS](use-managed-identity.md).

## D. Workload identity (pod → Azure)

Workload identity lets pods running in your AKS cluster authenticate to Microsoft Entra–protected Azure services (such as Key Vault, Storage, or Cosmos DB) without storing secrets in the cluster. AKS uses [Microsoft Entra Workload ID](workload-identity-overview.md), which projects a Kubernetes service account token federated to a Microsoft Entra application or user-assigned managed identity.

Don't use the deprecated [Microsoft Entra pod-managed identity](use-azure-ad-pod-identity.md) for new workloads.

## Decision guide

| Goal | Use these docs |
|---|---|
| Sign users into the cluster with Microsoft Entra ID | [Enable AKS-managed Entra integration](entra-id-control-plane-authentication.md) |
| Govern who can do what in the Kubernetes API across many clusters | [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md) |
| Restrict access to specific custom resource types | [ABAC conditions in Entra ID authorization](manage-entra-id-authorization.md#restrict-custom-resource-access-using-abac-conditions-preview) |
| Author per-cluster, per-namespace permissions as Kubernetes manifests | [Use Kubernetes RBAC with Entra integration](kubernetes-rbac-entra-id.md) |
| Let the cluster pull from ACR or attach disks | [Managed identities in AKS](use-managed-identity.md) |
| Let pods reach Key Vault or Storage without secrets | [Microsoft Entra Workload ID overview](workload-identity-overview.md) |
| Restrict who can download the cluster `kubeconfig` | [Limit access to cluster configuration file](control-kubeconfig-access.md) |

<a name='aks-service-permissions'></a>

## AKS service permissions reference

When creating a cluster, AKS generates or modifies resources it needs (like VMs and NICs) to create and run the cluster on behalf of the user. This identity is distinct from the cluster's identity permission, which is created during cluster creation.

For the built-in roles used to grant these permissions, see [Azure built-in roles for Containers](/azure/role-based-access-control/built-in-roles#containers). For a worked example of granting a service principal the permissions needed for a custom virtual network, see [Use a service principal with AKS](kubernetes-service-principal.md).

### Identity creating and operating the cluster permissions

The following permissions are needed by the identity creating and operating the cluster.

> [!div class="mx-tableFixed"]
> | Permission | Reason |
> |---|---|
> | `Microsoft.Compute/diskEncryptionSets/read` | Required to read disk encryption set ID. |
> | `Microsoft.Compute/proximityPlacementGroups/write` | Required for updating proximity placement groups. |
> | `Microsoft.Network/applicationGateways/read` <br/> `Microsoft.Network/applicationGateways/write` <br/> `Microsoft.Network/virtualNetworks/subnets/join/action` | Required to configure application gateways and join the subnet. |
> | `Microsoft.Network/virtualNetworks/subnets/join/action` | Required to configure the Network Security Group for the subnet when using a custom VNET.|
> | `Microsoft.Network/publicIPAddresses/join/action` <br/> `Microsoft.Network/publicIPPrefixes/join/action` | Required to configure the outbound public IPs on the Standard Load Balancer. |
> | `Microsoft.OperationalInsights/workspaces/sharedkeys/read` <br/> `Microsoft.OperationalInsights/workspaces/read` <br/> `Microsoft.OperationsManagement/solutions/write` <br/> `Microsoft.OperationsManagement/solutions/read` <br/> `Microsoft.ManagedIdentity/userAssignedIdentities/assign/action` | Required to create and update Log Analytics workspaces and Azure monitoring for containers. |
> | `Microsoft.Network/virtualNetworks/joinLoadBalancer/action` | Required to configure the IP-based Load Balancer Backend Pools. |

### AKS cluster identity permissions

The following permissions are used by the AKS cluster identity, which is created and associated with the AKS cluster. Each permission is used for the reasons below:

> [!div class="mx-tableFixed"]
> | Permission | Reason |
> |---|---|
> | `Microsoft.ContainerService/managedClusters/*`  <br/> | Required for creating users and operating the cluster
> | `Microsoft.Network/loadBalancers/delete` <br/> `Microsoft.Network/loadBalancers/read` <br/> `Microsoft.Network/loadBalancers/write` | Required to configure the load balancer for a LoadBalancer service. |
> | `Microsoft.Network/publicIPAddresses/delete` <br/> `Microsoft.Network/publicIPAddresses/read` <br/> `Microsoft.Network/publicIPAddresses/write` | Required to find and configure public IPs for a LoadBalancer service. |
> | `Microsoft.Network/publicIPAddresses/join/action` | Required for configuring public IPs for a LoadBalancer service. |
> | `Microsoft.Network/networkSecurityGroups/read` <br/> `Microsoft.Network/networkSecurityGroups/write` | Required to create or delete security rules for a LoadBalancer service. |
> | `Microsoft.Compute/disks/delete` <br/> `Microsoft.Compute/disks/read` <br/> `Microsoft.Compute/disks/write` <br/> `Microsoft.Compute/locations/DiskOperations/read` | Required to configure AzureDisks. |
> | `Microsoft.Storage/storageAccounts/delete` <br/> `Microsoft.Storage/storageAccounts/listKeys/action` <br/> `Microsoft.Storage/storageAccounts/read` <br/> `Microsoft.Storage/storageAccounts/write` <br/> `Microsoft.Storage/operations/read` | Required to configure storage accounts for AzureFile or AzureDisk. |
> | `Microsoft.Network/routeTables/read` <br/> `Microsoft.Network/routeTables/routes/delete` <br/> `Microsoft.Network/routeTables/routes/read` <br/> `Microsoft.Network/routeTables/routes/write` <br/> `Microsoft.Network/routeTables/write` | Required to configure route tables and routes for nodes. |
> | `Microsoft.Compute/virtualMachines/read` | Required to find information for virtual machines in a VMAS, such as zones, fault domain, size, and data disks. |
> | `Microsoft.Compute/virtualMachines/write` | Required to attach AzureDisks to a virtual machine in a VMAS. |
> | `Microsoft.Compute/virtualMachineScaleSets/read` <br/> `Microsoft.Compute/virtualMachineScaleSets/virtualMachines/read` <br/> `Microsoft.Compute/virtualMachineScaleSets/virtualmachines/instanceView/read` | Required to find information for virtual machines in a virtual machine scale set, such as zones, fault domain, size, and data disks. |
> | `Microsoft.Network/networkInterfaces/write` | Required to add a virtual machine in a VMAS to a load balancer backend address pool. |
> | `Microsoft.Compute/virtualMachineScaleSets/write` | Required to add a virtual machine scale set to a load balancer backend address pools and scale out nodes in a virtual machine scale set. |
> | `Microsoft.Compute/virtualMachineScaleSets/delete` | Required to delete a virtual machine scale set to a load balancer backend address pools and scale down nodes in a virtual machine scale set. |
> | `Microsoft.Compute/virtualMachineScaleSets/virtualmachines/write` | Required to attach AzureDisks and add a virtual machine from a virtual machine scale set to the load balancer. |
> | `Microsoft.Network/networkInterfaces/read` | Required to search internal IPs and load balancer backend address pools for virtual machines in a VMAS. |
> | `Microsoft.Compute/virtualMachineScaleSets/virtualMachines/networkInterfaces/read` | Required to search internal IPs and load balancer backend address pools for a virtual machine in a virtual machine scale set. |
> | `Microsoft.Compute/virtualMachineScaleSets/virtualMachines/networkInterfaces/ipconfigurations/publicipaddresses/read` | Required to find public IPs for a virtual machine in a virtual machine scale set. |
> | `Microsoft.Network/virtualNetworks/read` <br/> `Microsoft.Network/virtualNetworks/subnets/read` | Required to verify if a subnet exists for the internal load balancer in another resource group. |
> | `Microsoft.Compute/snapshots/delete` <br/> `Microsoft.Compute/snapshots/read` <br/> `Microsoft.Compute/snapshots/write` | Required to configure snapshots for AzureDisk. |
> | `Microsoft.Compute/locations/vmSizes/read` <br/> `Microsoft.Compute/locations/operations/read` | Required to find virtual machine sizes for finding AzureDisk volume limits. |

### Additional cluster identity permissions

When creating a cluster with specific attributes, you will need the following additional permissions for the cluster identity. Since these permissions are not automatically assigned, you must add them to the cluster identity after it's created.

> [!div class="mx-tableFixed"]
> | Permission | Reason |
> |---|---|
> | `Microsoft.Network/networkSecurityGroups/write` <br/> `Microsoft.Network/networkSecurityGroups/read` | Required if using a network security group in another resource group. Required to configure security rules for a LoadBalancer service. |
> | `Microsoft.Network/virtualNetworks/subnets/read` <br/> `Microsoft.Network/virtualNetworks/subnets/join/action` | Required if using a subnet in another resource group such as a custom VNET. |
> | `Microsoft.Network/routeTables/routes/read` <br/> `Microsoft.Network/routeTables/routes/write` | Required if using a subnet associated with a route table in another resource group such as a custom VNET with a custom route table. Required to verify if a subnet already exists for the subnet in the other resource group. |
> | `Microsoft.Network/virtualNetworks/subnets/read` | Required if using an internal load balancer in another resource group. Required to verify if a subnet already exists for the internal load balancer in the resource group. |
> | `Microsoft.Network/privatednszones/*` | Required if using a private DNS zone in another resource group such as a custom privateDNSZone. |

### AKS Node Access

By default Node Access is not required for AKS. The following access is needed for the node if a specific component is leveraged.

| Access | Reason |
|---|---|
| `kubelet` | Required to grant MSI access to ACR. |
| `http app routing` | Required for write permission to "random name".aksapp.io. |
| `container insights` | Required to grant permission to the Log Analytics workspace. |

## Next steps

* [Authentication and authorization concepts](concepts-authentication-authorization.md)
* [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md)
* [Managed identities in AKS](use-managed-identity.md)
* [Microsoft Entra Workload ID overview](workload-identity-overview.md)

For more information on core Kubernetes and AKS concepts, see the following articles:

* [Kubernetes / AKS clusters and workloads][aks-concepts-clusters-workloads]
* [Kubernetes / AKS security][aks-concepts-security]
* [Kubernetes / AKS virtual networks][aks-concepts-network]
* [Kubernetes / AKS storage][aks-concepts-storage]
* [Kubernetes / AKS scale][aks-concepts-scale]

<!-- LINKS - Internal -->
[aks-concepts-clusters-workloads]: concepts-clusters-workloads.md
[aks-concepts-security]: concepts-security.md
[aks-concepts-scale]: concepts-scale.md
[aks-concepts-storage]: concepts-storage.md
[aks-concepts-network]: concepts-network.md
