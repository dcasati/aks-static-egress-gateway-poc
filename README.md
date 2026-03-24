# AKS Static Egress Gateway — Non-Routable Node PoC

A proof-of-concept demonstrating how to solve IPv4 exhaustion in AKS when using
**Azure CNI Overlay + Cilium**, by placing worker nodes on a **non-routable RFC 6598
address space** while routing pod egress to internal corporate networks through a
dedicated **Static Egress Gateway** on a routable subnet.

---

## Problem

Customers running Azure CNI Overlay already solved pod IP exhaustion (pods use an
overlay CIDR, not VNet IPs). However, **nodes still consume routable VNet IPs** —
and large clusters can exhaust the corporate address space allocated to AKS.

The goal: move worker nodes to a **non-routable CIDR** so they don't consume
precious corporate IPs, while still allowing pods to egress to internal corporate
networks with a predictable, routable source IP.

**No NAT Gateway** — the Static Egress Gateway replaces it for corp-internal traffic.

---

## Architecture

```
VNet: 10.16.68.0/22  (routable — peered to corp hub)
│
├── master-subnet       10.16.68.0/24   ← AKS API server (VNet integrated)
├── gateway-subnet      10.16.69.0/26   ← Gateway node pool (ROUTABLE, 2 nodes)
│
│   NON-ROUTABLE (RFC 6598 — not advertised to corp)
├── worker-subnet-az1   100.68.0.0/24   ← System + worker nodes AZ1
├── worker-subnet-az2   100.68.1.0/24   ← System + worker nodes AZ2
└── worker-subnet-az3   100.68.2.0/24   ← System + worker nodes AZ3

Pod CIDR (overlay):  100.64.0.0/14    ← Never in VNet
Service CIDR:        192.168.128.0/17
Outbound:            LoadBalancer      ← Node bootstrap only (image pulls, Azure APIs)
                                         No NAT Gateway
```

**Egress flow for annotated pods:**
```
Pod (100.64.x.x overlay)
  → WireGuard tunnel (managed by egress gateway controller)
    → Gateway node (10.16.69.6 or 10.16.69.7)  ← routable corp IP
      → Corp internal network
```

Only **2 gateway node IPs** from the routable corp address space are consumed,
regardless of how many worker nodes exist.

---

## Prerequisites

- Azure CLI (`az`) with an active subscription
- `kubectl`
- `k9s` (optional, for cluster inspection)

---

## Step 1 — Create VNet and Subnets

```bash
RG="<your-resource-group>"
VNET="eph-static-egress-vnet"
LOC="eastus2"

# VNet spans both routable and non-routable space
az network vnet create \
  -g $RG -n $VNET -l $LOC \
  --address-prefixes 10.16.68.0/22 100.68.0.0/22

# Routable subnets
az network vnet subnet create -g $RG --vnet-name $VNET \
  -n master-subnet --address-prefix 10.16.68.0/24

az network vnet subnet create -g $RG --vnet-name $VNET \
  -n gateway-subnet --address-prefix 10.16.69.0/26

# Non-routable worker node subnets (RFC 6598 — not advertised to corp)
az network vnet subnet create -g $RG --vnet-name $VNET \
  -n worker-subnet-az1 --address-prefix 100.68.0.0/24

az network vnet subnet create -g $RG --vnet-name $VNET \
  -n worker-subnet-az2 --address-prefix 100.68.1.0/24

az network vnet subnet create -g $RG --vnet-name $VNET \
  -n worker-subnet-az3 --address-prefix 100.68.2.0/24
```

---

## Step 2 — Create the AKS Cluster

Worker nodes land on `worker-subnet-az1` (non-routable). Azure CNI Overlay + Cilium.
Static Egress Gateway feature enabled at cluster creation.

```bash
WORKER_AZ1=$(az network vnet subnet show -g $RG --vnet-name $VNET \
  -n worker-subnet-az1 --query id -o tsv)

az aks create \
  -n ephStaticEgress \
  -g $RG \
  -l $LOC \
  --kubernetes-version 1.34.1 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --enable-static-egress-gateway \
  --outbound-type loadBalancer \
  --pod-cidr 100.64.0.0/14 \
  --service-cidr 192.168.128.0/17 \
  --dns-service-ip 192.168.128.10 \
  --vnet-subnet-id "$WORKER_AZ1" \
  --nodepool-name systemnap1 \
  --node-count 1 \
  --zones 1 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys
```

> **Note on `--outbound-type loadBalancer`:** Worker nodes on `100.68.x.x` need to
> reach MCR and Azure APIs during bootstrap. The LB handles this via SNAT. In a
> fully private production deployment, replace with `--outbound-type none` and
> configure ACR private endpoints and Azure Private Link for all required services.

---

## Step 3 — Add Gateway Node Pool (Routable Subnet)

The gateway pool is the only pool placed on the routable subnet. It consumes just
2 corp IPs (`10.16.69.6`, `10.16.69.7`) regardless of cluster scale.

```bash
GW_SUBNET=$(az network vnet subnet show -g $RG --vnet-name $VNET \
  -n gateway-subnet --query id -o tsv)

az aks nodepool add \
  --cluster-name ephStaticEgress \
  --name gatewaypool \
  --resource-group $RG \
  --mode gateway \
  --node-count 2 \
  --gateway-prefix-size 30 \
  --vnet-subnet-id "$GW_SUBNET" \
  --zones 1 2 3
```

---

## Step 4 — Grant Network Permissions

The AKS managed identity needs Network Contributor on the VNet to manage
gateway LB configuration and VMSS NIC assignments.

```bash
PRINCIPAL_ID=$(az aks show -n ephStaticEgress -g $RG \
  --query "identity.principalId" -o tsv)

VNET_ID=$(az network vnet show -g $RG -n $VNET --query id -o tsv)

az role assignment create \
  --assignee "$PRINCIPAL_ID" \
  --role "Network Contributor" \
  --scope "$VNET_ID"
```

---

## Step 5 — Apply StaticGatewayConfiguration

`provisionPublicIps: false` keeps all egress on private routable IPs only.
No public IPs are allocated to gateway nodes.

```bash
kubectl apply -f manifests/static-gateway-config.yaml
```

See [`manifests/static-gateway-config.yaml`](manifests/static-gateway-config.yaml).

---

## Step 6 — Validate

Deploy two test pods — one annotated, one not:

```bash
kubectl apply -f manifests/test-pods.yaml
kubectl wait --for=condition=Ready pod/test-no-gateway pod/test-with-gateway --timeout=120s
```

Check egress paths:

```bash
# Non-annotated pod: egresses via worker node → LB SNAT → public IP
kubectl exec test-no-gateway -- curl -s https://ifconfig.me

# Annotated pod: egresses via gateway node (10.16.69.6/7) → no public IP
# Will timeout on internet targets — correct! Designed for corp-internal only.
kubectl exec test-with-gateway -- curl -s --max-time 10 https://ifconfig.me

# Both pods can reach excluded CIDRs (IMDS bypasses gateway)
kubectl exec test-no-gateway  -- curl -s -H "Metadata:true" \
  "http://169.254.169.254/metadata/instance/compute/location?api-version=2021-02-01&format=text"
kubectl exec test-with-gateway -- curl -s -H "Metadata:true" \
  "http://169.254.169.254/metadata/instance/compute/location?api-version=2021-02-01&format=text"
```

**Expected results:**

| Test | `test-no-gateway` | `test-with-gateway` |
|---|---|---|
| `ifconfig.me` | Public IP (LB SNAT) | Timeout ✅ — no public IP on gateway |
| Azure IMDS | `eastus2` | `eastus2` — excluded CIDR bypasses gateway |
| Egress source IP (corp) | Worker node → LB | `10.16.69.6` or `10.16.69.7` |

The timeout on the internet test **confirms the gateway is working** — the annotated
pod's traffic is correctly routed through the gateway nodes, which have no public IP.
In production, those pods reach corp-internal `10.x.x.x` targets and appear as
`10.16.69.6/7`.

---

## Key Takeaways

| Concern | This Architecture |
|---|---|
| Routable IPs consumed by nodes | **0** — all worker nodes on `100.68.x.x` |
| Routable IPs consumed by gateway | **2** (`10.16.69.6`, `10.16.69.7`) |
| NAT Gateway | **None** |
| Public IPs on gateway nodes | **None** (`provisionPublicIps: false`) |
| Pod overlay CIDR in VNet | **No** — `100.64.0.0/14` is overlay only |
| Corp-internal egress source IP | Predictable: `10.16.69.6` or `10.16.69.7` |

---

## Limitations

- `provisionPublicIps: false` requires Kubernetes 1.34+
- Static Egress Gateway is not supported with Azure CNI Pod Subnet (only Overlay)
- Network policies do not apply to traffic leaving via the gateway node pool
- Gateway node pool does not support autoscaling — size according to `/30` prefix capacity
- Pods must be in the same namespace as the `StaticGatewayConfiguration` resource

---

## References

- [AKS Static Egress Gateway docs](https://learn.microsoft.com/en-us/azure/aks/configure-static-egress-gateway)
- [Azure CNI Overlay](https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay)
- [RFC 6598 — Shared Address Space](https://datatracker.ietf.org/doc/html/rfc6598)
