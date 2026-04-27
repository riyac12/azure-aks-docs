---
title: Enable Federal Information Processing Standard (FIPS) for Azure Kubernetes Service (AKS) Cluster
description: Learn how to enable Federal Information Processing Standard (FIPS) for Azure Kubernetes Service (AKS) Cluster.
author: riychoudhary
ms.author: riychoudhary
ms.topic: how-to 
ms.date: 04/01/2026
ms.custom: template-how-to, linux-related-content
ai-usage: ai-assisted
zone_pivot_groups: azure-cli-arm-bicep-terraform-portal
# Customer intent: "As a cloud administrator, I want to enable FIPS compliance for AKS cluster, so that I can ensure the security of cryptographic modules and meet regulatory requirements while deploying applications."
---

# Enable Federal Information Processing Standard (FIPS) for Azure Kubernetes Service (AKS) Cluster

The Federal Information Processing Standard (FIPS) is a US government standard that defines minimum security requirements for cryptographic modules in information technology products and systems. This article describes how to enable FIPS mode at the **cluster level** for Azure Kubernetes Service (AKS). Cluster-level FIPS enforces that all AKS-managed compute and components operate using FIPS-validated cryptographic modules. This capability is intended for customers with compliance requirements such as FedRAMP who need a single, declarative way to ensure FIPS usage across their AKS cluster. For more information on FIPS, see [Federal Information Processing Standard (FIPS) 140][fips].

---

## What is FIPS mode in AKS?

FIPS 140-2/140-3 defines U.S. government standards for cryptographic modules. When you enable FIPS mode on an AKS cluster:

- All **node pools** in the cluster will use FIPS-enabled OS images.
- AKS blocks creation or scaling of **non-FIPS node pools**.
- Only **AKS-managed components** (system pods, add-ons, extensions) built with FIPS-approved cryptographic libraries are allowed.
- Data **in transit** and **at rest** uses FIPS-validated cryptography for AKS-managed paths.

> FIPS mode applies to **AKS-managed infrastructure and components**. Customers remain responsible for ensuring their **own application containers** use FIPS-compliant cryptography.

---

## Prerequisites

- An active Azure subscription. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/) before you begin.
- Set your subscription context using the [`az account set`][az-account-set] command. For example:

  ```azurecli-interactive
  az account set --subscription "00000000-0000-0000-0000-000000000000"
  ```

- [kubectl](https://kubernetes.io/releases/download/) installed. You can install it locally using the [`az aks install-cli`][az-aks-install-cli] command.


### Version compatibility

- Azure CLI version 2.32.0 or later installed and configured. To find the version, run `az --version`. For more information about installing or upgrading the Azure CLI, see [Install Azure CLI][install-azure-cli].

---

## Limitations

The following limitations apply when using FIPS-enabled AKS clusters.

- FIPS-enabled clusters are supported only on **AKS version 1.34 or later**.

- Some AKS-managed containers are not yet built with FIPS-compliant cryptographic libraries. These containers are **blocked from being deployed** when FIPS mode is enabled at the cluster level.
  <details>
  <summary><strong>Blocked AKS-managed containers</strong></summary>
  > This list applies only to AKS-managed components. Attempts to deploy or enable these components in a FIPS-enabled cluster will fail.
  > Customer application containers are not validated by AKS. 

  - `<container-name-1>`
  - `<container-name-2>`
  - `<container-name-3>`
  - `<container-name-4>`
  - `<container-name-5>`
  - `<container-name-6>`
  - `<container-name-7>`
  - `<container-name-8>`
  - `<container-name-9>`
  - `<container-name-10>`
  - `<container-name-11>`
  - `<container-name-12>`
  - `<container-name-13>`
  - `<container-name-14>`
  - `<container-name-15>`
  - `<container-name-16>`
  - `<container-name-17>`
  - `<container-name-18>`
  - `<container-name-19>`
  - `<container-name-20>`

  </details>

- **All AKS extensions are currently blocked** in FIPS-enabled clusters. This includes both Microsoft and partner extensions. For more information about Microsoft AKS extensions, see: [Currently available extensions][extensions]

- All limitations documented for FIPS-enabled node pools also apply to FIPS-enabled clusters, including OS, image, and upgrade constraints. For details, see: [Enable Federal Information Processing Standard (FIPS) for Azure Kubernetes Service (AKS) node pools][fips-node-pools]


## Create a FIPS-enabled AKS cluster

You can enable FIPS at cluster creation time using the `--enable-fips` flag.

1. Create a FIPS enabled AKS Cluster using the [`az aks create`][az-aks-create] command with the `--enable-fips` parameter.

    ```azurecli-interactive
    az aks create \
      --name myAKSCluster \
      --resource-group myResourceGroup \
      --enable-fips
    ```

2. Verify your cluster is FIPS-enabled using the [`az aks show`][az-aks-show] command and query for the _enableFIPS_ value.

    ```azurecli-interactive
    az aks show \
      --name myAKSCluster \
      --resource-group myResourceGroup \
      --query "enableFIPS"
    ```

    The following example output shows your cluster is FIPS-enabled:

    ```output
    Name         enableFips
    ---------    ------------
    myAKSCluster  True
    ```

3. Creating a FIPS-enabled cluster automatically creates a FIPS-enabled node pool. You can verify this by inspecting the node pool configuration using the [`az aks show`][az-aks-show] command and query for the _enableFIPS_ value in _agentPoolProfiles_.

    ```azurecli-interactive
      az aks show \
          --resource-group myResourceGroup \
          --name myAKSCluster \
          --query="agentPoolProfiles[].{Name:name enableFips:enableFips}" \
          -o table
      ```

      The following example output shows the default node pool is FIPS-enabled:

      ```output
      Name       enableFips
      ---------  ------------
      nodepool1  True
      ```

## Update an existing AKS cluster to enable or disable FIPS

You can update an existing AKS cluster to enable FIPS mode. However, the update will **fail** if the cluster does not meet all FIPS requirements when updating from non-FIPS to FIPS cluster. Before proceeding, ensure that your cluster is fully prepared.

### Prepare your cluster for FIPS

Before enabling FIPS at the cluster level, review the following requirements.

1. FIPS-enabled clusters are supported only on **AKS version 1.34 or later**. If your cluster is running an earlier version, upgrade the cluster before enabling FIPS: [Upgrade an AKS cluster][upgrade-aks-cluster]

2. If your cluster contains any **non-FIPS node pools**, the update will fail. You must first update each node pool to use FIPS-enabled images. For instructions, see: [Update an existing node pool to enable or disable FIPS][update-fips-node-pools]. After all node pools are FIPS-enabled, you can proceed with the cluster update.

3. **All AKS extensions are currently blocked** in FIPS-enabled clusters. If you have any extensions deployed, you must unintall them before enabling FIPS. For instructions, see:[Manage AKS extensions][manage-aks-extensions]

4. Some AKS-managed add-ons are not FIPS compliant and are blocked in FIPS-enabled clusters. If any blocked add-ons are enabled, you must disable them before proceeding. See the **“AKS-managed containers blocked in FIPS clusters”** list in the [Limitations](#limitations) section of this article.

### Update an existing cluster to enable FIPS

1. Update an AKS Cluster using the [`az aks update`][az-aks-update] command with the `--enable-fips` parameter.

    ```azurecli-interactive
    az aks update \
      --name myAKSCluster \
      --resource-group myResourceGroup \
      --enable-fips
    ```

2. Verify your cluster is FIPS-enabled using the [`az aks show`][az-aks-show] command and query for the _enableFIPS_ value.

    ```azurecli-interactive
    az aks show \
      --name myAKSCluster \
      --resource-group myResourceGroup \
      --query "enableFIPS"
    ```

    The following example output shows your cluster is FIPS-enabled:

    ```output
    Name         enableFips
    ---------    ------------
    myAKSCluster  True
    ```

### Update an existing cluster to disable FIPS

You can update an existing cluster to disable FIPS. When you disable FIPS mode:

- Cluster-level FIPS enforcement is removed.
- Existing node pools are not modified.
- You can run both FIPS and non-FIPS node pools in the same cluster.
- You can deploy AKS extensions and add-ons that were previously blocked while FIPS was enabled.
- New node pools are no longer required to be FIPS-enabled.

> Disabling FIPS does not automatically convert FIPS-enabled node pools to non-FIPS. If needed, you can update node pools individually.

1. Update an AKS Cluster using the [`az aks update`][az-aks-update] command with the `--disable-fips` parameter.

    ```azurecli-interactive
    az aks update \
      --name myAKSCluster \
      --resource-group myResourceGroup \
      --disable-fips
    ```

2. Verify your cluster is isn't FIPS-enabled using the [`az aks show`][az-aks-show] command and query for the _enableFIPS_ value.

    ```azurecli-interactive
    az aks show \
      --name myAKSCluster \
      --resource-group myResourceGroup \
      --query "enableFIPS"
    ```

    The following example output shows your cluster isn't FIPS-enabled:

    ```output
    Name         enableFips
    ---------    ------------
    myAKSCluster  False
    ```

<!-- LINKS - Internal -->
[az-aks-nodepool-add]: /cli/azure/aks/nodepool#az-aks-nodepool-add
[az-aks-nodepool-update]: /cli/azure/aks/nodepool#az-aks-nodepool-update
[az-aks-show]: /cli/azure/aks#az-aks-show
[az-aks-create]: /cli/azure/aks#az-aks-create
[aks-best-practices-security]: operator-best-practices-cluster-security.md
[aks-rdp]: rdp.md
[extensions]: /azure/aks/cluster-extensions?tabs=azure-cli#currently-available-extensions
[fips]: /azure/compliance/offerings/offering-fips-140-2
[fips-node-pools]: /azure/aks/enable-fips-nodes?pivots=azure-cli
[manage-aks-extensions]: /azure/aks/cluster-extensions
[upgrade-aks-cluster]: /azure/aks/tutorial-kubernetes-upgrade-cluster?tabs=azure-cli
[update-fips-node-pools]: /azure/aks/enable-fips-nodes?pivots=azure-cli#update-an-existing-node-pool-to-enable-or-disable-fips
