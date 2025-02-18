# OpenShift Cluster Migration using MTC (Migration Toolkit for Containers)

## Overview
This guide covers the step-by-step process to migrate applications with Persistent Volume Claims (PVCs) from **Cluster-A (Source)** to **Cluster-B (Destination)** using **Direct Volume Migration (DVM)** in **Red Hat OpenShift Migration Toolkit for Containers (MTC)**. The migration occurs **without needing a third cluster for intermediate storage**.

## Architecture
- **Cluster-A (Source):** Running the application with PVCs.
- **Cluster-B (Destination):** Target cluster where the application will be migrated.
- **MTC is installed on Cluster-B** and manages the migration process.
- **Direct Volume Migration (DVM)** is used to transfer data directly between clusters using Rsync over SSH.

## Prerequisites
### ✅ Ensure the following conditions are met:
1. **Both clusters are reachable over the network.**
2. **OpenShift 4.x is installed on both clusters.**
3. **MTC is installed on Cluster-B.**
4. **Service accounts and RBAC roles are properly configured.**
5. **Firewall rules allow SSH (port 22) for Rsync between clusters.**

## Step-by-Step Migration Guide

### **1️⃣ Install MTC on Cluster-B**
```sh
oc apply -f mtc-operator.yaml  # Install Migration Toolkit for Containers
```

### **2️⃣ Create Service Accounts & Grant Permissions**
Create a **Service Account (SA)** for migration in both clusters.

#### **Create Service Account in Cluster-A & Cluster-B**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: migration-sa
  namespace: openshift-migration
```
```sh
oc apply -f migration-sa.yaml
```

#### **Grant Cluster-Admin Role to SA**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: migration-sa-clusterrolebinding
subjects:
  - kind: ServiceAccount
    name: migration-sa
    namespace: openshift-migration
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
```sh
oc apply -f migration-sa-clusterrolebinding.yaml
```

### **3️⃣ Extract Service Account Token from Cluster-A**
```sh
oc sa get-token migration-sa -n openshift-migration
```
✅ **Copy the token output and store it in Cluster-B as a secret.**

### **4️⃣ Store Cluster-A Token in Cluster-B**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-a-sa-token
  namespace: openshift-migration
type: Opaque
data:
  token: <base64-encoded-token>
```
```sh
echo -n "your-cluster-a-token" | base64  # Encode the token before adding it
oc apply -f cluster-a-sa-secret.yaml
```

### **5️⃣ Register Clusters in MTC (on Cluster-B)**

#### **Register Cluster-A (Source) in MTC**
```yaml
apiVersion: migration.openshift.io/v1alpha1
kind: MigCluster
metadata:
  name: cluster-a
  namespace: openshift-migration
spec:
  isHostCluster: false
  url: "https://api.cluster-a.openshift.example.com:6443"
  serviceAccountSecretRef:
    name: cluster-a-sa-token
    namespace: openshift-migration
  insecure: false
```
```sh
oc apply -f cluster-a-migcluster.yaml
```

#### **Register Cluster-B (Destination) in MTC**
```yaml
apiVersion: migration.openshift.io/v1alpha1
kind: MigCluster
metadata:
  name: cluster-b
  namespace: openshift-migration
spec:
  isHostCluster: true
  url: "https://api.cluster-b.openshift.example.com:6443"
  serviceAccountSecretRef:
    name: cluster-b-sa-token
    namespace: openshift-migration
  insecure: false
```
```sh
oc apply -f cluster-b-migcluster.yaml
```

### **6️⃣ Verify Cluster Connectivity**
```sh
oc get migcluster -n openshift-migration
```
✅ Expected Output:
```
NAME         READY
cluster-a    True
cluster-b    True
```

### **7️⃣ Create Migration Plan**
```yaml
apiVersion: migration.openshift.io/v1alpha1
kind: MigPlan
metadata:
  name: direct-migration-plan
  namespace: openshift-migration
spec:
  migClusterRef:
    name: cluster-b
    namespace: openshift-migration
  namespaces:
    - demo
  persistentVolumes:
    - name: demo-pvc
      capacity: 10Gi
      storageClass: odf-ceph-rbd
      supported: true
      selection:
        action: direct  # Direct Volume Migration (DVM)
```
```sh
oc apply -f direct-migplan.yaml
```

### **8️⃣ Run the Final Migration**
```yaml
apiVersion: migration.openshift.io/v1alpha1
kind: MigMigration
metadata:
  name: direct-migration
  namespace: openshift-migration
spec:
  migPlanRef:
    name: direct-migration-plan
    namespace: openshift-migration
  stage: false
  quiescePods: true
  keepAnnotations: true
  rollback: false
  verify: true
```
```sh
oc apply -f direct-migmigration.yaml
```

### **9️⃣ Verify Migration Status**
```sh
oc get migmigration -n openshift-migration
```
✅ Expected Output:
```
NAME                STATUS      PHASE
direct-migration    Completed   FinalMigration
```

### **🚀 Success! The application and PVCs are now migrated to Cluster-B!** 🎯

## **Troubleshooting**
| Issue | Solution |
|--------|----------|
| MigCluster is not ready | Check SA token and connectivity between clusters |
| Rsync pods are not starting | Ensure SSH (port 22) is open between clusters |
| PVCs are not binding in Cluster-B | Check StorageClass mapping |

---
✅ **Your migration from Cluster-A to Cluster-B using Direct Volume Migration (DVM) is complete!** 🚀

