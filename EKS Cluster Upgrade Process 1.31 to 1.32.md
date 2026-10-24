# 🚀 AWS EKS Cluster Upgrade — v1.31 to v1.32

![AWS](https://img.shields.io/badge/AWS-EKS-orange?style=for-the-badge&logo=amazon-aws)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.32-blue?style=for-the-badge&logo=kubernetes)
![Status](https://img.shields.io/badge/Status-Completed-green?style=for-the-badge)

---

## 📋 Project Overview

This project documents the **in-place upgrade** of an Amazon EKS cluster from Kubernetes version **1.31** to **1.32** — covering control plane upgrade, add-on updates, and managed node group upgrade with zero application downtime.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│              AWS EKS Cluster                │
│                                             │
│   Control Plane (Managed by AWS)            │
│        v1.31  ──────────►  v1.32           │
│                                             │
│   Add-ons                                   │
│   ├── CoreDNS        v1.31  ──► v1.32      │
│   ├── kube-proxy     v1.31  ──► v1.32      │
│   └── vpc-cni        v1.31  ──► v1.32      │
│                                             │
│   Managed Node Group                        │
│        v1.31  ──────────►  v1.32           │
└─────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Tool | Version | Purpose |
|------|---------|---------|
| AWS EKS | 1.31 → 1.32 | Managed Kubernetes |
| kubectl | v1.32 | Kubernetes CLI |
| eksctl | Latest | EKS management CLI |
| AWS CLI | v2 | AWS resource management |

---

## ⚠️ Golden Rules Before Upgrading

```
✅ Always upgrade ONE minor version at a time (1.31 → 1.32 only)
✅ NEVER skip versions (1.31 → 1.33 ❌)
✅ Upgrade Control Plane FIRST → Add-ons SECOND → Node Groups LAST
✅ Take backups before starting
✅ Test in dev/staging before production
✅ Notify team of maintenance window
```

---

## 📁 Repository Structure

```
eks-upgrade-1.31-to-1.32/
├── README.md                        ← You are here
├── docs/
│   ├── pre-migration-checklist.md   ← Pre-checks guide
│   ├── rollback-plan.md             ← Rollback steps
│   └── post-migration-report.md     ← Verification report
├── scripts/
│   ├── 01-pre-checks.sh             ← Pre-migration checks
│   ├── 02-backup-resources.sh       ← Backup all K8s resources
│   ├── 03-upgrade-control-plane.sh  ← Control plane upgrade
│   ├── 04-upgrade-addons.sh         ← Add-ons upgrade
│   ├── 05-upgrade-nodegroup.sh      ← Node group upgrade
│   └── 06-post-verification.sh      ← Post-migration checks
└── backup/
    └── .gitkeep
```

---

## 🔄 Upgrade Flow

```
Phase 1          Phase 2          Phase 3          Phase 4
Pre-Checks  ──►  Control   ──►   Add-ons    ──►  Node Group
& Backup         Plane                            Upgrade
                Upgrade
    ↓               ↓                ↓               ↓
Backup all     15-20 mins       10 mins         20-30 mins
resources      UPDATING →       CoreDNS         per node
               ACTIVE ✅        kube-proxy      Rolling
                                vpc-cni         update ✅
```

---

## 📋 Phase 1 — Pre-Migration Checks

### Check Current Versions
```bash
# Check cluster version
aws eks describe-cluster \
  --name demo-cluster \
  --region us-east-1 \
  --query "cluster.version"

# Check nodes version
kubectl get nodes

# Check all pods are healthy
kubectl get pods --all-namespaces

# Check kubectl client version
kubectl version --client
```

### Check Add-on Versions
```bash
# List all addons
aws eks list-addons \
  --cluster-name demo-cluster \
  --region us-east-1

# Check each addon
aws eks describe-addon --cluster-name demo-cluster --addon-name coredns --region us-east-1
aws eks describe-addon --cluster-name demo-cluster --addon-name kube-proxy --region us-east-1
aws eks describe-addon --cluster-name demo-cluster --addon-name vpc-cni --region us-east-1
```

### Check for Deprecated APIs
```bash
# Install kubent
sh -c "$(curl -sSfL https://git.io/install-kubent)"

# Run deprecation check
kubent
```

### Backup All Resources
```bash
# Backup all kubernetes resources
kubectl get all --all-namespaces -o yaml > backup/backup-all-resources.yaml
kubectl get configmaps --all-namespaces -o yaml > backup/backup-configmaps.yaml
kubectl get secrets --all-namespaces -o yaml > backup/backup-secrets.yaml
kubectl get pv,pvc --all-namespaces -o yaml > backup/backup-volumes.yaml
kubectl get ingress --all-namespaces -o yaml > backup/backup-ingress.yaml

# Backup kubeconfig
cp ~/.kube/config ~/.kube/config-backup-1.31

echo "✅ Backup completed!"
```

---

## 📋 Phase 2 — Upgrade Control Plane

### Via AWS CLI
```bash
# Initiate control plane upgrade
aws eks update-cluster-version \
  --name demo-cluster \
  --kubernetes-version 1.32 \
  --region us-east-1
```

### Monitor Upgrade Status
```bash
# Watch upgrade status (runs every 30 seconds)
watch -n 30 'aws eks describe-cluster \
  --name demo-cluster \
  --region us-east-1 \
  --query "cluster.status"'

# Expected progression:
# "UPDATING" → wait → "ACTIVE" ✅
```

### Verify Control Plane
```bash
# Confirm version is now 1.32
aws eks describe-cluster \
  --name demo-cluster \
  --region us-east-1 \
  --query "cluster.version"

# Update kubeconfig
aws eks update-kubeconfig \
  --name demo-cluster \
  --region us-east-1
```

---

## 📋 Phase 3 — Upgrade Add-ons

> ⚠️ Must be done AFTER control plane, BEFORE node groups

### Check Latest Add-on Versions for 1.32
```bash
# Check latest CoreDNS version
aws eks describe-addon-versions \
  --addon-name coredns \
  --kubernetes-version 1.32 \
  --region us-east-1 \
  --query "addons[].addonVersions[0].addonVersion"

# Check latest kube-proxy version
aws eks describe-addon-versions \
  --addon-name kube-proxy \
  --kubernetes-version 1.32 \
  --region us-east-1 \
  --query "addons[].addonVersions[0].addonVersion"

# Check latest vpc-cni version
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.32 \
  --region us-east-1 \
  --query "addons[].addonVersions[0].addonVersion"
```

### Update Add-ons
```bash
# Update CoreDNS
aws eks update-addon \
  --cluster-name demo-cluster \
  --addon-name coredns \
  --resolve-conflicts OVERWRITE \
  --region us-east-1

# Update kube-proxy
aws eks update-addon \
  --cluster-name demo-cluster \
  --addon-name kube-proxy \
  --resolve-conflicts OVERWRITE \
  --region us-east-1

# Update VPC CNI
aws eks update-addon \
  --cluster-name demo-cluster \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE \
  --region us-east-1

echo "✅ All add-ons updated!"
```

### Verify Add-ons
```bash
# List all addons and status
aws eks list-addons \
  --cluster-name demo-cluster \
  --region us-east-1

# Check coredns pods
kubectl get pods -n kube-system | grep coredns
```

---

## 📋 Phase 4 — Upgrade Node Group

### Check Node Group Status
```bash
# List node groups
eksctl get nodegroup \
  --cluster demo-cluster \
  --region us-east-1

# Current node versions
kubectl get nodes
```

### Upgrade Node Group
```bash
# Via eksctl (Recommended)
eksctl upgrade nodegroup \
  --name my-nodegroup \
  --cluster demo-cluster \
  --kubernetes-version 1.32 \
  --region us-east-1

# OR via AWS CLI
aws eks update-nodegroup-version \
  --cluster-name demo-cluster \
  --nodegroup-name my-nodegroup \
  --kubernetes-version 1.32 \
  --region us-east-1
```

### Monitor Node Upgrade
```bash
# Watch nodes upgrade (rolling update)
watch -n 15 'kubectl get nodes'

# Expected:
# Old node (v1.31) → Cordon → Drain → Terminate
# New node (v1.32) → Launch → Join → Ready ✅
```

### Update kubectl to Match
```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.32.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

## 📋 Phase 5 — Post Migration Verification

```bash
# ✅ Check cluster version = 1.32
aws eks describe-cluster \
  --name demo-cluster \
  --region us-east-1 \
  --query "cluster.version"

# ✅ Check all nodes = v1.32
kubectl get nodes

# ✅ Check all pods running
kubectl get pods --all-namespaces

# ✅ Check no failed pods
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# ✅ Check deployments healthy
kubectl get deployments --all-namespaces

# ✅ Check services working
kubectl get services --all-namespaces

# ✅ Check system pods
kubectl get pods -n kube-system

# ✅ Application smoke test
curl http://<your-app-url>/health
```

---

## ✅ Full Migration Checklist

### Pre-Migration
- [ ] Check current cluster version
- [ ] Check all pods are Running
- [ ] Check addon versions
- [ ] Run kubent (deprecated API check)
- [ ] Backup all K8s resources
- [ ] Backup kubeconfig
- [ ] Notify team of maintenance window

### Control Plane
- [ ] Initiate upgrade to 1.32
- [ ] Wait for status = ACTIVE
- [ ] Verify version = 1.32
- [ ] Update kubeconfig

### Add-ons
- [ ] Update CoreDNS
- [ ] Update kube-proxy
- [ ] Update VPC CNI
- [ ] Verify all addons healthy

### Node Groups
- [ ] Upgrade node group to 1.32
- [ ] Wait for all nodes = Ready
- [ ] Verify node version = v1.32
- [ ] Update kubectl client

### Post-Migration
- [ ] All pods = Running
- [ ] All services = healthy
- [ ] Application smoke test passed
- [ ] Document migration completed

---

## 🔄 Rollback Plan

> If upgrade fails at any phase:

```bash
# Control plane rollback is NOT possible via AWS
# → Fix add-on issues first

# Node group rollback
# → Terminate new nodes
# → Launch old nodes from previous launch template version

# Application rollback
kubectl rollout undo deployment/<app-name>
```

---

## ⏱️ Timeline

| Phase | Task | Duration |
|-------|------|---------|
| Phase 1 | Pre-checks & Backup | ~30 mins |
| Phase 2 | Control Plane Upgrade | ~15-20 mins |
| Phase 3 | Add-ons Update | ~10 mins |
| Phase 4 | Node Group Upgrade | ~20-30 mins per node |
| Phase 5 | Post Verification | ~15 mins |
| **Total** | | **~1.5 - 2 hours** |

---

## 🐛 Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Upgrade stuck | Add-on incompatible | Update add-ons first |
| Pods crashing | Deprecated API used | Run kubent before upgrade |
| Nodes not joining | Wrong AMI | Use managed node group upgrade |
| kubectl mismatch | Old kubectl version | Update kubectl to 1.32 |
| Addon update failing | IAM permissions | Check IAM role permissions |
| i/o timeout on port 443 | Security group | Add port 443 inbound to EKS SG |

---

## 👨‍💻 Author

### [Sachin Gate](https://gatesachin1.github.io/)
*DevOps & AWS Cloud Engineer*

| | |
|---|---|
| 📧 **Email** | [gatesachin1112@gmail.com](mailto:gatesachin1112@gmail.com) |
| 🌐 **Website** | [Cloudnest Technology Solutions](https://cloudnesttechnologysolutions.in) |

> 💬 Found this guide useful? Give the repo a ⭐ — questions and suggestions are welcome via [Issues](https://github.com/gatesachin1/AWS-EKS-Cluster-Upgrade-v1.31-to-v1.32/issues).

---

## 📄 License
MIT License — feel free to use and adapt for your own EKS upgrades!
