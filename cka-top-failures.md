# Top Kubernetes Failures & Troubleshooting Guide for the CKA Exam

The Certified Kubernetes Administrator (CKA) exam heavily tests troubleshooting skills across control plane components, workloads, networking, storage, and security. Below are the most common ways Kubernetes clusters and workloads fail, how to identify the symptoms using `kubectl` and system logs, and how to fix them under exam time pressure.

---

## Table of Contents
1. [Control Plane & Component Failures](#1-control-plane--component-failures)
2. [Node & Kubelet Failures](#2-node--kubelet-failures)
3. [Pod Scheduling & Startup Failures](#3-pod-scheduling--startup-failures)
4. [Networking & Service Failures](#4-networking--service-failures)
5. [Storage & PVC Failures](#5-storage--pvc-failures)
6. [RBAC & Security Failures](#6-rbac--security-failures)
7. [Essential CKA Troubleshooting Toolkit](#7-essential-cka-troubleshooting-toolkit)

---

## 1. Control Plane & Component Failures

In a non-managed cluster (like those built with `kubeadm`), control plane components run as **Static Pods** managed by the kubelet on control plane nodes.

### A. Static Pod Misconfiguration or Crash
* **Symptoms:** `kubectl get pods -n kube-side` shows API server, controller manager, or scheduler crashing or missing. `kubectl cluster-info` fails with connection refused.
* **Root Causes:** 
  - YAML syntax error in static pod manifests.
  - Incorrect flag passed to API server, controller manager, or scheduler.
  - Port conflict or missing certificate path.
* **Exam Location:** `/etc/kubernetes/manifests/` (`kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml`, `etcd.yaml`).
* **Troubleshooting & Fix:**
  1. SSH into the control plane node.
  2. Inspect container runtime logs if pods aren't starting: `crictl ps -a` and `crictl logs <container-id>` (or check journalctl for kubelet: `journalctl -u kubelet -e`).
  3. Edit the manifest in `/etc/kubernetes/manifests/`. The kubelet automatically detects changes and restarts the static pod.

### B. Expired TLS Certificates
* **Symptoms:** Control plane components fail to communicate; `kubectl` commands fail with x509 certificate expiry errors.
* **Exam Location:** `/etc/kubernetes/pki/` or checked via kubeadm.
* **Troubleshooting & Fix:**
  1. Check certificate expiration status:
     ```bash
     kubeadm certs check-expiration
     ```
  2. Renew all certificates instantly with kubeadm:
     ```bash
     kubeadm certs renew all
     ```
  3. Restart control plane static pods (move manifests out of `/etc/kubernetes/manifests/` and back in, or restart the kubelet).

---

## 2. Node & Kubelet Failures

### A. Kubelet Service Down
* **Symptoms:** Nodes show `NotReady` status. `kubectl describe node <node-name>` shows unknown conditions or stale heartbeats.
* **Root Causes:** Kubelet daemon stopped, crashed, or failed to start due to configuration errors in `/var/lib/kubelet/config.yaml` or `/etc/systemd/system/kubelet.service.d/`.
* **Troubleshooting & Fix:**
  1. Check node status: `kubectl get nodes`
  2. SSH to the affected node and check kubelet service:
     ```bash
     sudo systemctl status kubelet
     sudo journalctl -u kubelet -e -n 100
     ```
  3. Start and enable kubelet:
     ```bash
     sudo systemctl daemon-reload
     sudo systemctl restart kubelet
     sudo systemctl enable kubelet
     ```

### B. Container Runtime Issues (containerd)
* **Symptoms:** Kubelet cannot create containers; pods stuck in `ContainerCreating`.
* **Troubleshooting & Fix:**
  1. Check container runtime status:
     ```bash
     sudo systemctl status containerd
     crictl ps
     ```
  2. Verify container runtime endpoint in kubelet configuration (`/var/lib/kubelet/config.yaml` or `/etc/default/kubelet`), usually `unix:///var/run/containerd/containerd.sock`.

---

## 3. Pod Scheduling & Startup Failures

### A. Pods Stuck in `Pending` (Scheduling Failures)
* **Symptoms:** `kubectl get pods` shows `Pending` indefinitely.
* **Root Causes:**
  - **Insufficient CPU/Memory:** Cluster nodes lack enough unallocated resources.
  - **Taints & Tolerations:** Node has a taint (e.g., `node-role.kubernetes.io/control-plane:NoSchedule`) that the pod does not tolerate.
  - **Node Affinity / NodeSelector:** Pod specifies labels that no node possesses.
  - **PersistentVolume unattached:** PVC cannot be bound.
* **Troubleshooting & Fix:**
  1. Run describe to see the scheduler's event log:
     ```bash
     kubectl describe pod <pod-name>
     ```
  2. Look at the `Events:` section at the bottom (e.g., `0/3 nodes are available: 3 insufficient cpu`).
  3. Inspect node capacities and allocatable resources:
     ```bash
     kubectl describe nodes | grep -A 5 Capacity:
     ```

### B. CrashLoopBackOff & Error States
* **Symptoms:** Pod status cycles between `Running`, `Error`, and `CrashLoopBackOff`.
* **Root Causes:**
  - Application code error causing immediate exit.
  - Missing environment variables, ConfigMaps, or Secrets.
  - Incorrect command/args override in pod spec.
  - Failing **Liveness Probe**.
* **Troubleshooting & Fix:**
  1. Check previous container logs (crucial if container restarts immediately):
     ```bash
     kubectl logs <pod-name> --previous
     ```
  2. Describe the pod to check exit codes, last state, and failed probes:
     ```bash
     kubectl describe pod <pod-name>
     ```

### C. ImagePullBackOff / ErrImagePull
* **Symptoms:** Pod cannot start; container image fails to download.
* **Root Causes:** Typo in image name/tag, private registry without `imagePullSecrets`, or network/DNS issue on the node.
* **Troubleshooting & Fix:**
  1. Check exact error in `kubectl describe pod <pod-name>`.
  2. Verify image tag and name.
  3. Ensure `imagePullSecrets` is correctly configured and attached to the ServiceAccount or Pod spec.

---

## 4. Networking & Service Failures

### A. Service Not Routing Traffic to Pods (Endpoints Missing)
* **Symptoms:** Requests to a ClusterIP, NodePort, or LoadBalancer time out or return 503/connection refused.
* **Root Causes:**
  - Service `selector` labels do not match Pod `labels`.
  - Service `port` / `targetPort` mismatch (e.g., Service listens on port 80, but container expects 8080).
* **Troubleshooting & Fix:**
  1. Check if the Service has active endpoints:
     ```bash
     kubectl get endpoints <service-name>
     ```
     *(If endpoints show `<none>`, the selector labels don't match any running pods).*
  2. Compare service selectors with pod labels:
     ```bash
     kubectl get svc <service-name> -o yaml
     kubectl get pods --show-labels
     ```

### B. CoreDNS / DNS Resolution Failures
* **Symptoms:** Pods cannot resolve internal service names (e.g., `nslookup my-svc` fails).
* **Root Causes:** CoreDNS pods crashing, Corefile misconfiguration, or node network issues.
* **Troubleshooting & Fix:**
  1. Check CoreDNS pod status in `kube-system`:
     ```bash
     kubectl get pods -n kube-system -l k8s-app=kube-dns
     ```
  2. Test DNS resolution from a debug pod:
     ```bash
     kubectl run tmp-shell --rm -it --image=busybox:1.36 -- sh
     nslookup kubernetes.default
     ```
  3. Check ConfigMap `coredns` in `kube-system` for syntax errors in the Corefile.

### C. NetworkPolicies Blocking Traffic
* **Symptoms:** Pods can communicate with some workloads but are blocked from others.
* **Troubleshooting & Fix:**
  1. Check if NetworkPolicies exist in the namespace:
     ```bash
     kubectl get networkpolicies -n <namespace>
     ```
  2. Review ingress/egress rules and pod selectors in the NetworkPolicy. Remember: Kubernetes NetworkPolicies are **additive** (default deny if any policy selects the pod, unless explicitly allowed).

---

## 5. Storage & PVC Failures

### A. PVC Stuck in `Pending`
* **Symptoms:** `kubectl get pvc` shows status `Pending`.
* **Root Causes:**
  - No available PersistentVolume (PV) matches the requested size, access modes (`RWO`, `ROX`, `RWX`), or StorageClass.
  - Dynamic provisioning failed (CSI driver down or misconfigured).
* **Troubleshooting & Fix:**
  1. Inspect the PVC and StorageClass:
     ```bash
     kubectl describe pvc <pvc-name>
     ```
  2. Check existing PVs:
     ```bash
     kubectl get pv
     ```

### B. Pod Stuck in `ContainerCreating` / Volume Attach Error
* **Symptoms:** Pod starts, but hangs at `ContainerCreating` or `VolumeAttachLimitExceeded`.
* **Troubleshooting & Fix:**
  1. Describe the pod or check cluster events:
     ```bash
     kubectl describe pod <pod-name>
     kubectl get events --sort-by='.metadata.creationTimestamp'
     ```
  2. Check CSI provisioner / node plugin logs in `kube-system`.

---

## 6. RBAC & Security Failures

### A. Forbidden Errors (`403` / `Unauthorized`)
* **Symptoms:** `kubectl auth can-i` returns false, or service accounts / users get permission errors.
* **Root Causes:** Missing Role/ClusterRole, incorrect RoleBinding/ClusterRoleBinding, or wrong ServiceAccount attached to the Pod.
* **Troubleshooting & Fix:**
  1. Test permissions using `can-i`:
     ```bash
     kubectl auth can-i get pods --as=system:serviceaccount:<namespace>:<sa-name>
     ```
  2. Verify RBAC resources:
     ```bash
     kubectl get role,rolebinding,clusterrole,clusterrolebinding -A | grep <name>
     ```

---

## 7. Essential CKA Troubleshooting Toolkit

Keep these commands memorized and ready for quick diagnosis during the exam:

| Goal | Command |
| :--- | :--- |
| **Quick cluster health** | `kubectl get nodes`, `kubectl get pods -A` |
| **Detailed diagnostics** | `kubectl describe pod <pod> -n <ns>` |
| **Container logs (current)** | `kubectl logs <pod> -n <ns> -c <container>` |
| **Container logs (previous crash)** | `kubectl logs <pod> -n <ns> --previous` |
| **Check service endpoints** | `kubectl get endpoints <service>` |
| **Test permissions** | `kubectl auth can-i <verb> <resource>` |
| **Check node logs (systemd)** | `sudo journalctl -u kubelet -e` |
| **Low-level container runtime** | `sudo crictl ps`, `sudo crictl logs <id>` |
| **Check static pods** | Check `/etc/kubernetes/manifests/` on control plane node |
| **Check cert expiration** | `kubeadm certs check-expiration` |
