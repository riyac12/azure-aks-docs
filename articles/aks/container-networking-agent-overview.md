---
title: Container Networking Agent for AKS overview
description: Learn about Container Networking Agent, an AI-powered diagnostic assistant that helps you troubleshoot networking issues in Azure Kubernetes Service (AKS) clusters.
author: shaifaligargmsft
ms.author: shaifaligarg
ms.date: 02/18/2026
ms.topic: overview
ms.service: azure-kubernetes-service
---

# What is Container Networking Agent for AKS?

Container Networking Agent is an AI-powered diagnostic assistant that helps you identify and resolve networking issues in your Azure Kubernetes Service (AKS) clusters. Once deployed, it runs as an in-cluster web application that you access through your browser. You type questions or describe networking problems in natural language — just as you would in any chat conversation — and the agent runs real diagnostic commands against your cluster and returns a structured, evidence-backed report with root cause analysis and remediation guidance.

No CLI expertise is required to get started. Open the chatbot URL in your browser, describe what's going wrong, and Container Networking Agent handles the rest.

Container Networking Agent doesn't modify your cluster. It operates with read-only access, so you can safely run diagnostics without risk to running workloads.

> [!NOTE]
> Container Networking Agent is also referred to internally as **CNA**. You may see this abbreviation in Azure portal experiences and CLI output.

## What can you do with Container Networking Agent?

Container Networking Agent helps you troubleshoot the most common — and most time-consuming — categories of AKS networking issues:

| Capability | What it does |
|-----------|-------------|
| **DNS troubleshooting** | Diagnoses CoreDNS failures, misconfigured DNS policies, network policies blocking DNS traffic, NodeLocal DNS issues, and Cilium FQDN egress restrictions |
| **Packet drop analysis** | Investigates NIC-level RX drops, kernel packet loss, socket buffer overflow, SoftIRQ saturation, and ring buffer exhaustion across cluster nodes |
| **Kubernetes networking diagnostics** | Identifies pod connectivity failures, service port misconfigurations, network policy conflicts, missing endpoints, and Hubble flow analysis |
| **Cluster resource queries** | Answers questions about pods, services, deployments, nodes, and namespaces to give you quick situational awareness |

Each diagnostic produces a structured report that includes what was checked, what's healthy, what failed, the identified root cause, and the exact commands to fix and verify the issue.

## When to use Container Networking Agent

### Use Container Networking Agent when you need to

- Diagnose why pods can't resolve DNS names in your cluster
- Investigate intermittent packet drops or high network latency on AKS nodes
- Determine why a service is unreachable from certain pods or namespaces
- Identify which network policy is blocking traffic
- Check if Cilium, Hubble, or CoreDNS components are healthy
- Get a quick overview of cluster networking state

### Container Networking Agent isn't designed for

- Application code debugging or software development assistance
- Storage, PersistentVolume, or disk troubleshooting
- RBAC configuration, secrets management, or security auditing (except network policies)
- Workload scheduling, resource optimization, or cost management
- Non-Azure cloud environments (AWS, GCP)
- Making changes to your cluster — all access is read-only

## How it works

When you describe a networking issue, Container Networking Agent follows a structured diagnostic workflow:

```
You describe the issue → Agent classifies it → Collects evidence from the cluster → Analyzes findings → Reports results
```

**1. Classify** — The agent determines the issue category (DNS, connectivity, network policy, service routing, or packet drops) based on your description.

**2. Collect evidence** — The agent runs the appropriate diagnostic commands against your cluster using `kubectl`, `cilium`, and `hubble` through a read-only interface. Each diagnostic category has a purpose-built evidence collection workflow that gathers the right data automatically.

**3. Analyze** — The agent examines collected evidence to separate healthy signals from anomalies. It follows a strict evidence-first standard: conclusions are based only on what the commands actually returned, never on speculation.

**4. Report** — You receive a structured report that includes:
- A summary of the issue and its status
- An evidence table showing each check, its result, and whether it passed or failed
- Analysis of what's working and what's broken
- Root cause identification with specific evidence citations
- Exact commands to fix the issue and verify the fix

### Integrations

Container Networking Agent works with the AKS networking tools you already use:

| Integration | How it's used |
|------------|---------------|
| **kubectl** | Queries pods, services, endpoints, nodes, network policies, and other Kubernetes resources |
| **Cilium** | Analyzes CiliumNetworkPolicy, CiliumClusterWideNetworkPolicy, and Cilium agent health |
| **Hubble** | Observes network flows between pods and identifies dropped traffic (requires ACNS) |
| **CoreDNS** | Checks pod health, service endpoints, configuration, and Prometheus metrics |
| **Azure OpenAI** | Powers the conversational AI that interprets your questions and generates diagnostic reports |

> [!TIP]
> For the full diagnostic feature set — including Hubble flow analysis and Cilium policy diagnostics — deploy Container Networking Agent on an AKS cluster with [Azure CNI powered by Cilium](/azure/aks/azure-cni-powered-by-cilium) and [Advanced Container Networking Services (ACNS)](/azure/aks/advanced-container-networking-services-overview) enabled.

## Safety model and limitations

### Read-only access

Container Networking Agent operates with a dedicated Kubernetes service account (`container-networking-agent-reader`) that has **read-only** permissions:

- Cannot create, update, or delete any cluster resource
- Cannot modify running workloads, configurations, or network policies
- Limited `pods/exec` access only for diagnostic commands (Cilium status and endpoint information)
- All API calls are logged by the Kubernetes audit system

Remediation guidance is always **advisory** — the agent tells you what commands to run, but it never runs them for you.

### Scope restrictions

The agent responds only to networking and Kubernetes-related questions. Off-topic requests are politely declined. The system also includes prompt injection defenses to prevent misuse.

### Session and conversation limits

| Limit | Default | Notes |
|-------|---------|-------|
| Chat context window | ~15 exchanges | Older messages are dropped from the agent's working context. Start a new conversation for unrelated issues. |
| Messages per conversation | 100 | Older messages are automatically removed when this limit is reached |
| Conversations per user | 20 | Least-recently-used conversations are cleaned up at 90% capacity |
| Session idle timeout | 30 minutes | Sessions expire after 30 minutes of inactivity |
| Session absolute timeout | 8 hours | Sessions expire after 8 hours regardless of activity |

### Concurrency

Container Networking Agent supports 1–7 concurrent users under typical conditions. Packet drop diagnostics on larger clusters (25+ nodes) may require limiting concurrent users to avoid API server load. For details, see [Scale guidance](#scale-guidance).

## Prerequisites

Before deploying Container Networking Agent, ensure you have:

- An **AKS cluster** with [workload identity](/azure/aks/workload-identity-overview) and [OIDC issuer](/azure/aks/use-oidc-issuer) enabled
- An **Azure OpenAI** resource with a deployed model (GPT-4.1 or equivalent)
- A **user-assigned managed identity** with the following role assignments:
  - `Cognitive Services OpenAI User` on the Azure OpenAI resource
  - `Azure Kubernetes Service Cluster User Role` on the AKS cluster
  - `Reader` on the resource group
- **Federated credentials** linking the managed identity to the Kubernetes service account (`system:serviceaccount:kube-system:container-networking-agent-reader`)
- (Optional) An **Azure App Registration** for Microsoft Entra ID user authentication

For Cilium and Hubble diagnostics, your cluster should use [Azure CNI powered by Cilium](/azure/aks/azure-cni-powered-by-cilium). Without Cilium, Container Networking Agent still provides DNS, packet drop, and standard Kubernetes networking diagnostics.

> [!div class="nextstepaction"]
> [Get started with Container Networking Agent](./quickstart.md)

## Example scenarios and sample prompts

### DNS troubleshooting

DNS resolution failures are one of the most common networking issues in Kubernetes. When pods can't resolve service names, external domains, or both, Container Networking Agent runs a comprehensive DNS diagnostic that checks CoreDNS health, configuration, DNS resolution from multiple paths, and network policies that might block DNS traffic.

**Common situations:**

- Pods log `Name or service not known` or `NXDOMAIN` errors
- Applications time out reaching services by name
- DNS works for some pods but not others
- External domain resolution fails while internal resolution works (or vice versa)

**Sample prompts you can try:**

| What you're seeing | Prompt |
|-------------------|--------|
| DNS completely broken | *"All DNS is broken in the cluster"* |
| Pod can't resolve names | *"A pod in namespace `my-app` cannot resolve any DNS names"* |
| Specific name not resolving | *"DNS resolution for `backend.default.svc.cluster.local` is failing"* |
| Intermittent DNS failures | *"Pods in `production` have intermittent DNS failures"* |
| External DNS blocked | *"External DNS fails for pods in `my-namespace`"* |
| NodeLocal DNS issues | *"Can you check if NodeLocal DNS is working?"* |

**What the agent checks:**

The DNS diagnostic covers CoreDNS pod health, service endpoints, CoreDNS configuration (including custom ConfigMaps), NodeLocal DNS status, DNS resolution tests across multiple paths (same-namespace, cross-namespace, FQDN, external), CoreDNS Prometheus metrics, and network policy analysis — including Cilium toFQDN egress policies that may silently restrict external domain resolution.

**Example root causes the agent identifies:**

- CoreDNS pods not running or not ready
- Custom CoreDNS ConfigMap with misconfigured rewrite or forward rules
- Network policy blocking UDP/TCP port 53 (DNS traffic)
- Cilium toFQDNs policy missing a required domain in its allow list
- NodeLocal DNS DaemonSet deployed without a Cilium LocalRedirectPolicy
- Application configured with the wrong service DNS name

### RX / Packet drop troubleshooting

Packet drops are notoriously difficult to diagnose because they can occur at multiple layers — NIC hardware, kernel networking stack, or application socket buffers. Container Networking Agent deploys a lightweight debug pod to each node to collect host-level network statistics, then uses delta measurements to identify where packets are being lost.

**Common situations:**

- Applications report intermittent connection resets or timeouts
- Tools like `iperf` show packet loss between nodes
- Network latency spikes appear on specific nodes
- High CPU usage correlated with network processing
- `ethtool -S` shows incrementing RX drop counters

**Sample prompts you can try:**

| What you're seeing | Prompt |
|-------------------|--------|
| Drops on a specific node | *"I see packet drops on node `aks-nodepool1-12345678-vmss000000`"* |
| Latency spikes | *"My application is experiencing intermittent latency spikes"* |
| Cluster-wide performance issues | *"Network performance is degraded cluster-wide"* |
| Packet loss detected | *"I'm seeing packet drops and high latency. The iperf tests show significant packet loss."* |
| Proactive health check | *"Check network health on node `my-node`"* |

**What the agent checks:**

The packet drop diagnostic examines NIC ring buffer utilization 〈`ethtool`〉, kernel softnet statistics (`/proc/net/softnet_stat`), per-CPU SoftIRQ distribution, socket buffer saturation, network interface statistics (`/proc/net/dev`), kernel buffer tunables (`tcp_rmem`, `rmem_max`, `netdev_max_backlog`), RPS/XPS/RFS configuration, and CNI-specific interface analysis. Delta measurements (before-and-after snapshots) are used to detect active drops versus historical counters.

**Example root causes the agent identifies:**

- NIC ring buffer exhaustion — active `rx_dropped` counters incrementing
- Kernel packet drops — non-zero values in `/proc/net/softnet_stat` drop column
- Socket buffer overflow — socket receive queue growing beyond buffer limits
- SoftIRQ CPU bottleneck — high `%soft` on a single CPU with imbalanced interrupt distribution
- All checks passing — agent reports "No issue detected" rather than guessing

> [!IMPORTANT]
> The packet drop diagnostic deploys a debug DaemonSet (`retina-debug-daemonset`) to your cluster's `kube-system` namespace. This DaemonSet requires `hostNetwork` and `NET_ADMIN` capabilities to access host-level network data. It's shared across diagnostic sessions and cleaned up automatically, but may persist if the agent pod crashes unexpectedly. See [Known issues](#known-issues-and-product-limitations) for cleanup guidance.

### Kubernetes networking troubleshooting

When pods can't communicate with services, network policies block expected traffic, or services have no endpoints, Container Networking Agent investigates the full networking path — from pod scheduling and readiness, through service endpoint registration, to network policy evaluation and Hubble flow observation.

**Common situations:**

- Pod-to-pod or pod-to-service communication fails
- Services are unreachable from certain namespaces
- Network policies unexpectedly block traffic
- Service endpoints exist but connections still time out
- Hubble shows `DROPPED` verdict on flows between pods

**Sample prompts you can try:**

| What you're seeing | Prompt |
|-------------------|--------|
| Service unreachable | *"My client pod cannot connect to the backend-service in `production`. The connection times out."* |
| Traffic blocked | *"My client pod can't reach the backend-service anymore. It was working before."* |
| No endpoints | *"Service has no endpoints in namespace `my-app`"* |
| Pod stuck | *"I deployed my app but the service has no endpoints and the pod doesn't have an IP"* |
| Pods not ready | *"Pods are not ready in namespace `staging`"* |
| Proactive health check | *"Everything looks fine in namespace `production` — can you verify?"* |

**What the agent checks:**

The Kubernetes networking diagnostic examines pod status and scheduling, service configuration and endpoint registration, network policies (both Kubernetes NetworkPolicy and CiliumNetworkPolicy), Hubble flows (including dropped traffic), and service-to-pod port mapping. A particularly common misconfiguration the agent catches is service `targetPort` not matching pod `containerPort` — which causes connection timeouts even though endpoints appear healthy.

**Example root causes the agent identifies:**

- Network policy (or CiliumNetworkPolicy) blocking ingress or egress traffic
- Service `targetPort` doesn't match the pod's `containerPort`
- Service selector labels don't match any pod labels (empty endpoints)
- Pod stuck in Pending due to unschedulable resource requests
- Readiness probe failing, causing pods to be excluded from service endpoints
- Cilium agent pods not healthy

> [!NOTE]
> Hubble flow analysis (`hubble observe`) requires [Advanced Container Networking Services (ACNS)](/azure/aks/advanced-container-networking-services-overview) to be enabled on your cluster. On clusters without ACNS, Container Networking Agent still provides full diagnostics using `kubectl` and standard Kubernetes resources, but flow-level visibility is unavailable.

## Known issues and product limitations

### Scale guidance

| Cluster size | Recommended concurrent users | Notes |
|-------------|------------------------------|-------|
| 1–3 nodes | Up to 7 | Optimal for most diagnostics |
| 25 nodes | Up to 3 | Packet drop diagnostics generate per-node evidence bundles |
| 50 nodes | 1 | Large evidence bundles approach AI model context limits |

The first query from a new user may take longer if the pre-warmed agent pool (default: 3 agents) is exhausted. Subsequent queries from the same session use the already-initialized agent.

### Known issues

| Issue | Description | Workaround |
|-------|-------------|------------|
| **Debug DaemonSet persists after crash** | If the Container Networking Agent pod crashes during a packet drop diagnostic, the `retina-debug-daemonset` may remain in `kube-system` | Run `kubectl delete ds retina-debug-daemonset -n kube-system` |
| **First packet drop diagnostic is slower** | The debug DaemonSet takes 30–60 seconds to schedule and become ready on first use | Subsequent diagnostics reuse existing pods and are faster |
| **Non-Cilium clusters have reduced diagnostics** | Cilium policy analysis and Hubble flow observation aren't available | Agent still provides full DNS, packet drop, and standard Kubernetes diagnostics |
| **Non-ACNS clusters lack Hubble** | `hubble observe` commands fail on clusters without Advanced Container Networking Services | Enable ACNS, or rely on `kubectl`-based diagnostics |
| **DNS tests run from agent pod** | DNS resolution tests execute from the Container Networking Agent pod, which may have a different DNS policy than the affected pod | Agent notes its own DNS policy in the evidence for comparison |
| **Session data is in-memory** | Session state (chat history, agent assignments) is lost if the pod restarts | Log back in to start a new session; no persistent conversation history |
| **Chat context window** | The agent retains only the last ~15 exchanges in its working context | For unrelated issues, start a new conversation to avoid context confusion |

### Extension availability

When deployed as an AKS extension (`microsoft.retinaagent`), Container Networking Agent is available in: **centralus**, **eastus**, **eastus2euap**, **centralusueap**, **eastus2**, **uksouth**, **westus2**.

## Pricing

Container Networking Agent runs as a pod in your AKS cluster. Direct costs include:

- **Azure OpenAI usage** — Token consumption depends on conversation length and diagnostic complexity. See [Azure OpenAI pricing](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/) for current rates.
- **AKS node compute** — The Container Networking Agent pod and (for packet drop diagnostics) the debug DaemonSet consume cluster compute resources.

Container Networking Agent itself has no separate licensing fee.

## Customer experience, support, and feedback

Container Networking Agent is a browser-based chatbot that runs inside your AKS cluster. After deployment, you open the application URL in any modern browser to start a conversation. There's no CLI tool to install on your workstation and no portal blade to navigate — it's a standalone chat interface purpose-built for network diagnostics.

<!-- TODO: Replace the placeholder below with a screenshot of the Container Networking Agent chat interface -->

:::image type="content" source="./images/container-networking-agent-chat.png" alt-text="Screenshot of the Container Networking Agent chat interface showing a user prompt and a structured diagnostic response." lightbox="./images/container-networking-agent-chat.png":::

The chat interface is where you:

- **Ask questions in natural language** — Type prompts like *"Why can't my pod resolve DNS?"* or *"Check packet drops on node aks-nodepool1-vmss000000"*. No special syntax required.
- **Receive structured diagnostic reports** — Responses include evidence tables, root cause analysis, and copy-pasteable remediation commands.
- **Start new conversations** — Each conversation maintains its own context. Switch topics by starting a new conversation.
- **Submit feedback** — After each diagnostic response, use the built-in feedback controls (thumbs up / thumbs down) to rate the quality of the diagnosis. Your feedback helps improve future diagnostic accuracy.

### Authentication

Depending on how your administrator configured the deployment, you sign in with either a simple username (development environments) or your Microsoft Entra ID credentials (production environments). After sign-in, your session is maintained server-side — you can close and reopen the browser tab within the session timeout window without losing your conversation.

### Reporting issues

If you encounter a problem with Container Networking Agent:

1. Note the **session ID** and **timestamp** of the issue (visible in the chat interface)
2. Check the health endpoints: `/health`, `/ready`, `/live`
3. Review pod logs: `kubectl logs -l app=container-networking-agent -n kube-system`
4. File an issue through your standard Azure support channel

## Next steps

- **Get started** — [Quickstart: Deploy Container Networking Agent](./quickstart.md)
- **Understand the architecture** — [Agent architecture design](./AGENT_ARCHITECTURE_DESIGN.md)
- **Learn about permissions** — [RBAC configuration](../helm/container-networking-agent/README-RBAC.md)
- **Deploy as AKS extension** — [Extension deployment guide](../deployment/EXTENSIONS.md)
- **Review conversation limits** — [Chat history and conversation limits](./CONVERSATION_LIMITS.md)
- **Explore rate limiting** — [Rate limiter design](./RateLimiter.md)
