---

## 🏗️ Azure Kubernetes (AKS) Architecture - Production Grade

> **Disclaimer:** This ain't no playground setup. This is the real deal for production workloads that need to scale and stay secure 🔥

### 📋 Architecture Overview
- [Network Layer (Azure CNI Deep Dive)](#network-layer)
- [Identity & Security (Workload Identity)](#identity--security)
- [Ingress & Traffic Management](#ingress--traffic-management)
- [Secrets Management (Key Vault CSI)](#secrets-management)
- [Monitoring Stack](#monitoring-stack)

---

### 🌐 Network Layer (Azure CNI)

We use **Azure CNI (Advanced Networking)** instead of Kubenet because we need direct Pod-to-VNet communication without NAT BS.

```
┌─────────────────────────────────────────────────────────────────┐
│                         AZURE VNET                              │
│                      10.0.0.0/8 (Supernet)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐      ┌──────────────────────────────┐ │
│  │   Subnet: AKS        │      │   Subnet: Private Endpoints  │ │
│  │   10.240.0.0/16      │      │   10.0.2.0/24                │ │
│  │  (4094 IPs available)│      │                              │ │
│  │                      │      │  ┌──────────┐  ┌──────────┐  │ │
│  │  ┌────────────────┐  │      │  │ SQL DB   │  │ KeyVault │  │ │
│  │  │   AKS Nodes    │  │      │  │ 10.0.2.5 │  │ 10.0.2.6 │  │ │
│  │  │   (VMSS)       │  │      │  └──────────┘  └──────────┘  │ │
│  │  │                │  │      └──────────────────────────────┘ │
│  │  │  ┌──────────┐  │  │                                       │
│  │  │  │ Pod: Web │  │  │◀────── Direct communication           │
│  │  │  │10.240.0.4│  │  │       (No NAT bullsh*t)               │
│  │  │  └────┬─────┘  │  │                                       │
│  │  │       │        │  │                                       │
│  │  │  ┌────▼─────┐  │  │                                       │
│  │  │  │ Pod: API │  │  │                                       │
│  │  │  │10.240.0.5│  │  │                                       │
│  │  │  └────┬─────┘  │  │                                       │
│  │  └───────┼────────┘  │                                       │
│  │          │           │                                       │
│  └──────────┼───────────┘                                       │
│             │                                                   │
│             └──────────────────┬──────────────────┐             │
│                                ▼                  ▼             │
│                        ┌──────────────┐   ┌──────────────┐      │
│                        │ App Gateway  │   │   Firewall   │      │
│                        │  (Ingress)   │   │   (WAF)      │      │
│                        └──────────────┘   └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘


🔢 IP Math (Don't Run Out!)
The Formula:


Total IPs needed = (Max Nodes × 1) + (Max Nodes × Max Pods per Node) + 20% Buffer

Real World Example:
- Nodes: 10-20 (autoscaler)
- Max Pods per Node: 50 (we bump from default 30)
- Calculation: 20 + (20 × 50) = 1,020 IPs
- Safe Subnet: /21 (2,046 IPs) ← Always go bigger than you think


⚠️ WARNING: If you run out of IPs, new pods get stuck in FailedScheduling. Resizing an AKS subnet requires recreating the entire cluster (total downtime!). Plan this right from day one.


# Check IP availability before scaling up
az network vnet subnet show \
  --resource-group rg-networking \
  --vnet-name vnet-prod \
  --name subnet-aks \
  --query "availableIpAddresses"

# Check current IP usage
kubectl get nodes -o json | jq '.items[].spec.podCIDR'
kubectl get pods -A -o wide | wc -l


🛡️ Identity & Security (Workload Identity)
We moved from the old AAD Pod Identity (deprecated) to Azure Workload Identity (OIDC-based). It's faster, more secure, and no heavy daemonset drama.

How The Magic Happens:

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Pod App    │────────▶│     AKS      │────────▶│  Azure AD    │
│              │         │  OIDC Issuer │         │              │
│  (Service    │         │              │         │  (Federated  │
│   Account)   │         │  "This token │         │   Credential)│
└──────────────┘         │   is valid   │         └──────┬───────┘
                         │   from AKS X"│                │
                         └──────────────┘                │
                                                         ▼
                                                  ┌──────────────┐
                                                  │ Access Token │
                                                  │  for KeyVault│
                                                  └──────┬───────┘
                                                         │
                             ┌───────────────────────────┘
                             ▼
                      ┌──────────────┐
                      │ Azure Key    │
                      │ Vault        │
                      │ (Secrets)    │
                      └──────────────┘
Setup Steps:
1. Enable OIDC on AKS (One-time):

az aks update \
  --name aks-prod-cluster \
  --resource-group rg-kubernetes \
  --enable-oidc-issuer \
  --enable-workload-identity

# Save the issuer URL
export AKS_OIDC_ISSUER=$(az aks show \
  --name aks-prod-cluster \
  --resource-group rg-kubernetes \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

2. Create Managed Identity & Service Account:

# Create User Assigned Managed Identity
az identity create \
  --name id-aks-apps \
  --resource-group rg-security \
  --location southeastasia

export USER_ASSIGNED_CLIENT_ID=$(az identity show \
  --name id-aks-apps \
  --resource-group rg-security \
  --query 'clientId' -otsv)

# Create K8s Service Account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-workload-sa
  namespace: production
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
EOF


3. Create Federation (The Glue):


az identity federated-credential create \
  --name "k8s-federated-cred" \
  --identity-name id-aks-apps \
  --resource-group rg-security \
  --issuer ${AKS_OIDC_ISSUER} \
  --subject "system:serviceaccount:production:app-workload-sa" \
  --audiences api://AzureADTokenExchange

4. Deploy with the Magic Label:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: myapp
        # THIS LABEL IS MANDATORY!
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: app-workload-sa  # Must match federation
      containers:
      - name: app
        image: myacr.azurecr.io/myapp:v1.0
        env:
        - name: AZURE_CLIENT_ID
          value: "<USER_ASSIGNED_CLIENT_ID>"


🚪 Ingress & Traffic Management
Choose your fighter:

Type	Best For			Pros					Cons
AGIC	Enterprise, WAF needs		Native Azure, Auto SSL, WAF built-in	Expensive, less flexible
NGINX	Custom rules, complex routing	Flexible, community support, cheap	Manage SSL yourself


NGINX + Cert-Manager Example:


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.yourcompany.com
    secretName: api-tls-cert
  rules:
  - host: api.yourcompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

🔐 Secrets Management (Azure Key Vault + CSI Driver)
Never, EVER commit secrets to Git. We mount them directly from Azure Key Vault using the Secrets Store CSI Driver.


┌─────────────────────────────────────────────────────────────┐
│                        Kubernetes                           │
│                                                             │
│  ┌──────────────┐      ┌──────────────┐     ┌──────────┐    │
│  │   Pod App    │      │ Secrets Store│     │  Azure   │    │
│  │              │◀────▶│ CSI Driver   │◀───▶│ KeyVault │    │
│  │ /mnt/secrets │      │ (Provider)   │     │          │    │
│  │   - db-pass  │      │              │     │ Secrets  │    │
│  │   - api-key  │      │              │     │ Certs    │    │
│  └──────────────┘      └──────────────┘     └──────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
SecretProviderClass Config:

apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
  namespace: production
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "<USER_ASSIGNED_CLIENT_ID>"  # From Workload Identity
    keyvaultName: "kv-prod-secrets"
    cloudName: AzurePublicCloud
    objects: |
      array:
        - |
          objectName: db-connection-string
          objectType: secret
        - |
          objectName: redis-password
          objectType: secret
        - |
          objectName: tls-cert
          objectType: secret
    tenantId: "<AZURE_TENANT_ID>"
  
  # Sync to K8s Secret (optional but recommended for env vars)
  secretObjects:
  - secretName: app-secrets
    type: Opaque
    data:
    - objectName: db-connection-string
      key: DATABASE_URL
Usage in Deployment:
YAML

spec:
  containers:
  - name: app
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: app-secrets  # From sync above
          key: DATABASE_URL
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-kvname"
📊 Monitoring Stack
Don't fly blind. Setup monitoring so you can sleep at night.

┌──────────────────────────────────────────────────────────────┐
│                    Azure Monitor (Log Analytics)             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ Container    │  │ Metrics      │  │ Alerts       │        │
│  │ Insights     │  │ (Prometheus) │  │ (Email/Slack)│        │
│  │ (Logs)       │  │              │  │              │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
│         │                 │                 │                │
│         └─────────────────┼─────────────────┘                │
│                           │                                  │
│                    ┌──────▼──────┐                           │
│                    │   AKS       │                           │
│                    │  Cluster    │                           │
│                    └─────────────┘                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
Enable Container Insights:
Bash

az aks enable-addons \
  --resource-group rg-kubernetes \
  --name aks-prod-cluster \
  --addons monitoring \
  --workspace-resource-id "/subscriptions/xxx/.../workspaces/logs-prod"

🚑 Troubleshooting Common Issues
1. Pod Stuck ContainerCreating (IP Exhaustion)
Error: no IP addresses available in range set

Fix:

# Check IP pool status
kubectl get nodes -o json | jq '.items[].metadata.name, .items[].spec.podCIDR'

# Solution: Scale down nodes OR rebuild cluster with bigger subnet (pain!)
# Prevention: Always use /20 or larger subnet
2. Workload Identity Authentication Failing
Debug Checklist:

# 1. Is OIDC enabled?
az aks show -n aks-prod -g rg-kubernetes --query "oidcIssuerProfile.enabled"

# 2. Check federation subject matches exactly (typos common here!)
az identity federated-credential list \
  --identity-name id-aks-apps \
  --resource-group rg-security

# 3. Service Account has annotation?
kubectl get sa app-workload-sa -n production -o jsonpath='{.metadata.annotations}'

# 4. Pod has the required label?
kubectl get pod <pod-name> -n production -o jsonpath='{.metadata.labels}' | grep workload
3. Key Vault CSI Driver Errors
Bash

# Check if driver is running
kubectl get pods -n kube-system | grep csi-secrets-store

# Check logs
kubectl logs -n kube-system -l app=csi-secrets-store-provider-azure

# Verify secret sync worked
kubectl get secret app-secrets -n production
✅ Production Readiness Checklist
Before you say "it's done", verify this:

 AKS subnet is /20 or larger (prevent IP exhaustion)
 Workload Identity configured for all apps (no static credentials!)
 Network Policies active (default deny, explicit allow)
 Key Vault integration via CSI Driver (not hardcoded env vars)
 Ingress with SSL termination (cert-manager or App Gateway)
 Pod Disruption Budget (PDB) set for zero-downtime updates
 HPA configured for auto-scaling pods
 Container Insights enabled with Azure Monitor dashboards
 Velero backup configured for cluster state
 Azure Policy for K8s enabled (compliance guardrails)
Next Level: Once this is solid, add GitOps with ArgoCD/Flux for fully automated deployments. Want that section too, bro? 🚀

