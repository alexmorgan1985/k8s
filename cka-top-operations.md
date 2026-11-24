# Top Operations & Tasks for the CKA Exam

The Certified Kubernetes Administrator (CKA) exam tests hands-on operational tasks under strict time constraints. Below are the most critical operations, step-by-step procedures, and exam tips for mastering them.

---

## Table of Contents
1. [Cluster Upgrades (kubeadm)](#1-cluster-upgrades-kubeadm)
2. [etcd Backup and Restore](#2-etcd-backup-and-restore)
3. [Node Maintenance (Cordon & Drain)](#3-node-maintenance-cordon--drain)
4. [Managing Static Pods](#4-managing-static-pods)
5. [Configuring Network Policies](#5-configuring-network-policies)
6. [Storage: PV, PVC, and StorageClasses](#6-storage-pv-pvc-and-storageclasses)
7. [JSONPath Querying & Advanced `kubectl` Output](#7-jsonpath-querying--advanced-kubectl-output)

---

## 1. Cluster Upgrades (`kubeadm`)

Upgrading a `kubeadm` cluster is a multi-step process. You typically upgrade the control plane node first, followed by worker nodes.

### Step 1: Upgrade the First Control Plane Node
1. **Unfreeze/unhold kubeadm & upgrade kubeadm:**
   ```bash
   sudo apt-mark unhold kubeadm
   sudo apt-get update && sudo apt-get install -y kubeadm=<target-version>-00
   sudo apt-mark hold kubeadm
   ```
2. **Plan the upgrade:**
   ```bash
   sudo kubeadm upgrade plan
   ```
3. **Apply the upgrade:**
   ```bash
   sudo kubeadm upgrade apply v<target-version>
   ```
4. **Upgrade kubelet and kubectl on the control plane node:**
   ```bash
   sudo apt-mark unhold kubelet kubectl
   sudo apt-get install -y kubelet=<target-version>-00 kubectl=<target-version>-00
   sudo apt-mark hold kubelet kubectl
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

### Step 2: Upgrade Worker Nodes
1. **Drain the node from the control plane:**
   ```bash
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```
2. **SSH to the worker node and upgrade kubeadm & kubelet:**
   ```bash
   sudo apt-mark unhold kubeadm kubelet
   sudo apt-get update && sudo apt-get install -y kubeadm=<target-version>-00 kubelet=<target-version>-00
   sudo apt-mark hold kubeadm kubelet
   sudo kubeadm upgrade node
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```
3. **Uncordon the node:**
   ```bash
   kubectl uncordon <node-name>
   ```

---

## 2. etcd Backup and Restore

etcd is the consistent and highly-available key-value store holding all cluster state. Knowing how to take a snapshot and restore it is a guaranteed exam topic.

### A. Take a Snapshot
You need etcd client certificates (`etcdctl` expects API version 3).
```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```
*Verify snapshot:*
```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db -w table
```

### B. Restore from a Snapshot
Restoring etcd creates a new data directory.
1. **Run restore command:**
   ```bash
   sudo ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
     --data-dir=/var/lib/etcd-restore
   ```
2. **Update etcd static pod manifest** (`/etc/kubernetes/manifests/etcd.yaml`):
   - Change `--data-dir` and volume mounts from `/var/lib/etcd` to `/var/lib/etcd-restore` (or move/rename directories).
   - The kubelet will automatically restart etcd with the restored data.

---

## 3. Node Maintenance (Cordon & Drain)

Before performing maintenance on a node (reboot, kernel upgrade, hardware swap), you must evict workloads safely.

* **Cordon** (mark node unschedulable so new pods aren't placed on it):
  ```bash
  kubectl cordon <node-name>
  ```
* **Drain** (evict all running pods gracefully, respecting PodDisruptionBudgets):
  ```bash
  kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
  ```
* **Uncordon** (return node to active scheduling state):
  ```bash
  kubectl uncordon <node-name>
  ```

---

## 4. Managing Static Pods

Static pods are managed directly by the kubelet daemon on a specific node, without the API server observing them directly.

* **Default Manifest Path:** `/etc/kubernetes/manifests/`
* **Exam Task Example:** *Create a static pod named `static-web` using nginx image on control plane node.*
  1. SSH to the specified node.
  2. Generate or write a pod YAML in `/etc/kubernetes/manifests/static-web.yaml`:
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: static-web
     spec:
       containers:
         - name: web
           image: nginx
     ```
  3. Verify via `kubectl get pods` (static pods automatically append the node name as a suffix, e.g., `static-web-controlplane`).

---

## 5. Configuring Network Policies

NetworkPolicies control traffic flow at the IP address or port level (ingress/egress). They require a CNI that supports NetworkPolicies (e.g., Calico, Cilium).

### Exam Example: Restrict Ingress to a Namespace/Pod
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## 6. Storage: PV, PVC, and StorageClasses

### A. PersistentVolume (PV) - Static Provisioning
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/data
```

### B. PersistentVolumeClaim (PVC)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-claim1
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## 7. JSONPath Querying & Advanced `kubectl` Output

The CKA exam frequently asks for specific lists sorted, filtered, or formatted using JSONPath.

* **List all pod names:**
  ```bash
  kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
  ```
* **List pod names and their IP addresses:**
  ```bash
  kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
  ```
* **Sort nodes by CPU capacity:**
  ```bash
  kubectl get nodes --sort-by='.status.capacity.cpu'
  ```
* **Save resource definition to a file without running it (Dry-run):**
  ```bash
  kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
  ```
