---
title: Authentication and authorization concepts in Azure Kubernetes Service (AKS)
titleSuffix: Azure Kubernetes Service
description: Learn how authentication and authorization for the Kubernetes API work in Azure Kubernetes Service (AKS), and how to choose between Kubernetes RBAC and Microsoft Entra ID authorization.
ms.topic: concept-article
ms.subservice: aks-security
ms.date: 04/19/2026
author: shashankbarsin
ms.author: shasb
ai-usage: ai-assisted
# Customer intent: "As a cluster operator, I want to understand how authentication and authorization for the Kubernetes API work in AKS so that I can choose the right model for governing access at scale across my clusters."
---

# Authentication and authorization concepts in Azure Kubernetes Service (AKS)

This article describes how AKS handles **control-plane access** — that is, who can authenticate to the Kubernetes API and what they're allowed to do once they're in. It covers the recommended authentication path, the two authorization models AKS supports, fine-grained custom resource control with ABAC conditions, and how to lock down break-glass access.

For the other identity scenarios in AKS, see:

* [Managed identities in AKS](use-managed-identity.md) for cluster-to-Azure access (such as pulling images from ACR or attaching disks).
* [Microsoft Entra Workload ID overview](workload-identity-overview.md) for pod-to-Azure access (such as workloads calling Key Vault).

For an orientation across all four AKS identity scenarios, see [Access and identity options for AKS](concepts-identity.md).

## Authenticate cluster users with Microsoft Entra ID

Kubernetes itself doesn't provide an identity directory. Without an external identity provider, you'd need to manage local credentials per cluster, which doesn't scale and creates audit gaps.

We recommend deploying AKS clusters with [Microsoft Entra ID authentication for the control plane][entra-id-cp-auth]. With this integration, the cluster validates incoming Kubernetes API requests against Microsoft Entra ID and uses the caller's Entra identity for authorization decisions. Microsoft Entra ID centralizes the identity layer — any change in user or group status is automatically reflected in cluster access — and enables Conditional Access, multifactor authentication, and Privileged Identity Management.

![Cluster-level authentication for Microsoft Entra integration with AKS](media/operator-best-practices-identity/cluster-level-authentication-flow.png)

For setup, see [Enable Microsoft Entra ID authentication for the AKS control plane][entra-id-cp-auth].

## Authorize the Kubernetes API

After a caller is authenticated, AKS evaluates whether the caller is authorized to perform the requested action. AKS supports two authorization models for the Kubernetes API:

* **Kubernetes role-based access control (RBAC).** The native Kubernetes authorization model. Permissions are defined as `Role` and `ClusterRole` objects and granted to subjects through `RoleBinding` and `ClusterRoleBinding` objects stored in each cluster.
* **Microsoft Entra ID authorization.** An AKS authorization webhook that delegates authorization decisions to Microsoft Entra ID. Permissions are granted as Azure role assignments to Entra ID identities, and can be optionally refined with [Azure ABAC conditions](/azure/role-based-access-control/conditions-overview).

You can use both models on the same cluster. We recommend Microsoft Entra ID authorization as the default and reserve Kubernetes RBAC for fine-grained intra-cluster permissions. The rest of this section explains why and when to use each.

### Kubernetes RBAC

Kubernetes RBAC is the upstream Kubernetes authorization model. You author `Role` or `ClusterRole` objects that grant verbs (such as `get`, `list`, `create`) on resources (such as `pods`, `deployments`), and bind them to subjects (users, groups, or service accounts) using `RoleBinding` or `ClusterRoleBinding` objects. The Kubernetes API server's built-in RBAC authorizer evaluates these bindings on every request.

Use Kubernetes RBAC when you want:

* Fine-grained, intra-cluster, per-namespace access control authored as Kubernetes manifests alongside the workloads they protect.
* GitOps-managed authorization that lives in the same source of truth as your application configuration.
* Permissions for in-cluster service accounts that workloads use to call the Kubernetes API.

Kubernetes RBAC permissions are scoped to a single cluster. To apply the same policy to many clusters, you must apply the manifests to each cluster (typically through GitOps). Use Microsoft Entra users and groups as subjects in Kubernetes `RoleBinding` and `ClusterRoleBinding` objects so that human identities still come from your central directory.

For background on the Kubernetes RBAC model, see the [upstream Kubernetes RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). For setup in AKS, see [Use Kubernetes RBAC with Microsoft Entra integration](kubernetes-rbac-entra-id.md).

### Microsoft Entra ID authorization for the Kubernetes API

With Entra ID authorization, AKS deploys an authorization webhook that delegates Kubernetes API authorization decisions to Microsoft Entra ID. When a request reaches the API server, the webhook calls the Entra ID `checkaccess` API to evaluate the caller's Azure role assignments (and any attached ABAC conditions) and returns an allow or deny decision.

![Entra ID authorization webhook flow for the Kubernetes API](media/concepts-identity/azure-rbac-k8s-authz-flow.png)

Entra ID authorization gives you the following benefits over managing Kubernetes RBAC manifests on every cluster:

* **Single identity plane.** The same Microsoft Entra users, groups, and service principals that govern access to your Azure resources also govern access to your Kubernetes API. There's no separate user directory to provision or rotate.
* **Assign once, govern many clusters.** Azure role assignments can be made at **subscription, management group, or resource group scope**. A single role assignment at a resource group scope grants access to every current and future AKS cluster in that resource group. With Kubernetes RBAC, you must apply manifests to every cluster individually.
* **Conditional Access and Privileged Identity Management (PIM).** Cluster access automatically inherits your organization's existing Entra ID Conditional Access policies (such as multifactor authentication or location-based restrictions) and can be elevated just-in-time through PIM.
* **Centralized audit.** Every role assignment change is recorded in the Azure Activity Log alongside other Azure resource changes, so you have one audit trail for cluster access governance.
* **Fine-grained custom resource constraints.** With ABAC conditions, you can restrict access to specific custom resource (CRD) groups and kinds without writing per-cluster Kubernetes RBAC manifests.

AKS provides the following built-in roles for Entra ID authorization:

| Role | Description |
|---|---|
| Azure Kubernetes Service RBAC Reader | Read-only access to most objects in a namespace. Doesn't allow viewing roles, role bindings, or `Secrets`. |
| Azure Kubernetes Service RBAC Writer | Read/write access to most objects in a namespace. Doesn't allow viewing or modifying roles or role bindings. |
| Azure Kubernetes Service RBAC Admin | Read/write access to most resources in a namespace, plus the ability to create roles and role bindings within the namespace. |
| Azure Kubernetes Service RBAC Cluster Admin | Full control over every resource in the cluster, across all namespaces. |

For custom permission patterns, you can author custom role definitions that target specific Kubernetes API groups using the `Microsoft.ContainerService` resource provider's data actions. For step-by-step setup and custom role examples, see [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md).

### Comparison

| Capability | Kubernetes RBAC | Entra ID authorization | Entra ID authorization with ABAC conditions |
|---|---|---|---|
| Identity source | Kubernetes users, groups, service accounts | Microsoft Entra ID identities | Microsoft Entra ID identities |
| Scope of a single grant | One cluster | Resource, resource group, subscription, or management group | Resource, resource group, subscription, or management group |
| Multi-cluster governance | Apply manifests to each cluster (typically GitOps) | One role assignment at a higher scope governs many clusters | One role assignment at a higher scope governs many clusters |
| Conditional Access / PIM | Not supported | Inherited from Entra ID | Inherited from Entra ID |
| Audit trail | Cluster audit log | Azure Activity Log | Azure Activity Log |
| Filter access by CRD group or kind | Author per-CRD `Role` objects | Use wildcard roles (broad) | Use ABAC condition attributes |

## Restrict custom resource access with ABAC conditions

When you grant broad read access through Microsoft Entra ID authorization but want to restrict which custom resources (CRDs) the assignee can read, attach an Azure ABAC condition to the role assignment.

Without ABAC conditions, granting read on custom resources requires a wildcard like `Microsoft.ContainerService/managedClusters/*/read`, which covers every CRD on every cluster in scope. With ABAC, you can attach a condition that restricts access to specific CRD groups and kinds — for example, allow `templates.gatekeeper.sh` while blocking `kyverno.io` — without writing per-cluster Kubernetes RBAC manifests.

For Kubernetes API authorization, you can filter access to custom resources by their API group and kind using the following attributes:

* `Microsoft.ContainerService/managedClusters/customResources:group`
* `Microsoft.ContainerService/managedClusters/customResources:kind`

For background on Azure ABAC, see [What are Azure role assignment conditions?](/azure/role-based-access-control/conditions-overview). For step-by-step setup, see [Restrict custom resource access using ABAC conditions](manage-entra-id-authorization.md#restrict-custom-resource-access-using-abac-conditions-preview).

## Disable local accounts and enforce Conditional Access

Local accounts use a built-in cluster admin certificate that bypasses Entra ID, breaking centralized audit and Conditional Access. In production, disable local accounts on AKS clusters so that all access flows through Microsoft Entra ID. Apply Conditional Access policies (multifactor authentication, location restrictions) and use Privileged Identity Management for break-glass access. For details, see:

* [Manage local accounts in AKS](local-accounts.md)
* [Cluster and node access control with Conditional Access](access-control-managed-azure-ad.md)
* [Cluster and node access control with Privileged Identity Management](privileged-identity-management.md)

## Requirements and limitations

* Entra ID authorization for the Kubernetes API requires Microsoft Entra ID authentication to be enabled on the cluster. To enable, see [Enable Microsoft Entra ID authentication for the AKS control plane][entra-id-cp-auth].
* The Microsoft Entra tenant configured for cluster authentication must be the same as the tenant of the subscription that holds the AKS cluster.
* For non-interactive logins or older `kubectl` versions, use the [`kubelogin`](https://github.com/Azure/kubelogin) plugin.
* ABAC conditions are in preview. To use ABAC conditions, enable Entra ID authorization at cluster creation. Some clusters that have Entra ID authorization enabled post-creation might not evaluate ABAC conditions in the authorization webhook.

## Next steps

* [Enable Microsoft Entra ID authentication for the AKS control plane][entra-id-cp-auth]
* [Use Microsoft Entra ID authorization for the Kubernetes API](manage-entra-id-authorization.md)
* [Use Kubernetes RBAC with Microsoft Entra integration](kubernetes-rbac-entra-id.md)
* [Managed identities in AKS](use-managed-identity.md)
* [Microsoft Entra Workload ID overview](workload-identity-overview.md)
* [What are Azure role assignment conditions?](/azure/role-based-access-control/conditions-overview)

<!-- INTERNAL LINKS -->
[entra-id-cp-auth]: entra-id-control-plane-authentication.md
