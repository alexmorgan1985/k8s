# Service types: ClusterIP vs NodePort vs LoadBalancer

The three types build on each other. Each one includes everything the one before it does, and adds one
more way in.

```
LoadBalancer  = NodePort     + an external load balancer
NodePort      = ClusterIP    + a port opened on every node
ClusterIP     = a virtual IP inside the cluster, balanced across pods
```

See also: `how-service-traffic-reaches-a-pod.md` (the two-stage path: client to node, then node to pod).

## 1. The vocabulary, as a picture

Things that exist as real objects, and the one that doesn't:

```
 OUTSIDE THE CLUSTER (not Kubernetes)
 ┌─────────────────┐          ┌──────────────────────────┐  cloud LB, MetalLB, HAProxy ...
 │ External client │─────────►│ External load balancer   │  Kubernetes only asks for it;
 │ (your Mac)      │          │ one stable IP; health    │  it does not make it
 └─────────────────┘          │ checks the nodes         │
          │                   └──────────────────────────┘
          │ straight to a node IP           │ to node IPs : nodePort
          │                                 │
══════════╪═════════════════════════════════╪═════════════════════════════════════════ cluster boundary
          ▼                                 ▼
 ┌─ CLUSTER ──────────────────────────────────────────────────────────────────────────┐
 │ ┌─ cp1 ──────────────────┐  ┌─ w1 ───────────────────┐  ┌─ w2 ───────────────────┐ │
 │ │ 192.168.252.2          │  │ 192.168.252.3          │  │ 192.168.252.4          │ │
 │ │ control plane          │  │ worker node            │  │ worker node            │ │
 │ │                        │  │                        │  │                        │ │
 │ │ kube-proxy rules       │  │ kube-proxy rules       │  │ kube-proxy rules       │ │
 │ │                        │  │                        │  │                        │ │
 │ │ (no web pods:          │  │ ┌─ pod A ────────────┐ │  │ ┌─ pod B ────────────┐ │ │
 │ │ control plane is       │  │ │ 10.244.190.65      │ │  │ │ 10.244.80.193      │ │ │
 │ │ tainted)               │  │ │ [nginx  :80]       │ │  │ │ [nginx  :80]       │ │ │
 │ └────────────────────────┘  │ └────────────────────┘ │  │ └────────────────────┘ │ │
 │                             └────────────────────────┘  │ ┌─ pod C ────────────┐ │ │
 │                                                         │ │ 10.244.80.194      │ │ │
 │                                                         │ │ [nginx  :80]       │ │ │
 │                                                         │ └────────────────────┘ │ │
 │                                                         └────────────────────────┘ │
 │                                                                                    │
 │                                                                                    │
 │ Service "web" is NOT a box. It is the same set of kube-proxy rules on every        │
 │ node: "traffic for this IP:port -> pick a ready pod (A, B or C)".                  │
 │ Its stable name and IP: web / 10.111.51.47.                                        │
 │                                                                                    │
 └────────────────────────────────────────────────────────────────────────────────────┘
```

- **Cluster:** the set of nodes plus the control plane.
- **Node:** a machine (here a VM) that runs pods. Each node has its own IP and its own kube-proxy.
- **Pod:** one or more containers sharing one IP. Each pod here runs one app container (nginx on port 80).
- **Service:** a stable virtual IP and DNS name. It exists only as rules on every node, not as a process
  you can point at.
- **External load balancer:** outside the cluster. Kubernetes asks for it with `type: LoadBalancer`, but
  something else has to provide it.

## 2. ClusterIP (the default)

A virtual IP and DNS name that exist only inside the cluster.

```
   OUTSIDE
 ┌─────────────────┐
 │ External client │───✗  no route in: a ClusterIP is internal only
 └─────────────────┘
════════════════════════════════════════════════════════════════════════════════ cluster boundary
 ┌─ CLUSTER ────────────────────────────────────────────────────────────────────┐
 │                                                                              │
 │ ┌─ w1 ───────────────────────────────────┐  ┌─ w2 ─────────────────────────┐ │
 │ │ 192.168.252.3                          │  │ 192.168.252.4                │ │
 │ │ ┌─ frontend pod ─────────┐             │  │ ┌─ pod B ────────────────┐   │ │
 │ │ │ [app]                  │             │  │ │ 10.244.80.193          │   │ │
 │ │ │ GET http://web         │             │  │ │ [nginx  :80]           │   │ │
 │ │ └────────────────────────┘             │  │ └────────────────────────┘   │ │
 │ │       │ (1) DNS: web = 10.111.51.47    │  │ ┌─ pod C ────────────────┐   │ │
 │ │       ▼                                │  │ │ 10.244.80.194          │   │ │
 │ │ kube-proxy rules (2) pick one:         │  │ │ [nginx  :80]           │   │ │
 │ │   ├─► pod A  (this node)               │  │ └────────────────────────┘   │ │
 │ │   ├─► pod B  (on w2)                   │  └──────────────────────────────┘ │
 │ │   └─► pod C  (on w2)                   │                                   │
 │ │ ┌─ pod A ────────────────┐             │                                   │
 │ │ │ 10.244.190.65          │             │                                   │
 │ │ │ [nginx  :80]           │             │                                   │
 │ │ └────────────────────────┘             │                                   │
 │ └────────────────────────────────────────┘                                   │
 │                                                                              │
 └──────────────────────────────────────────────────────────────────────────────┘
```

- Reached by: other pods and nodes, at `web` or `10.111.51.47:80`.
- No external client can use it. There is no stage 1 (client to node), because the caller is already on a node.
- Use it for: internal services, such as a frontend calling a backend, or a database.

```yaml
spec:
  type: ClusterIP        # can be left out
  selector: {app: web}
  ports:
  - port: 80
    targetPort: 80
```
```bash
k expose deploy web --port=80 --target-port=80
```

## 3. NodePort

ClusterIP plus the same port opened on every node.

```
                                  ┌─────────────────┐
                                  │ External client │  the client picks the node by the IP it uses
                                  │ (your Mac)      │  (stage 1: nothing in Kubernetes does this)
                                  └─────────────────┘
                                           │
               ┌───────────────────────────┬───────────────────────────┐
               │ (alt)                     │ (1) used                  │ (alt)
═══════════════╪═══════════════════════════╪═══════════════════════════╪══════════════ cluster boundary
               ▼                           ▼                           ▼
 ┌─ CLUSTER ──────────────────────────────────────────────────────────────────────────┐
 │ ┌─ cp1 ──────────────────┐  ┌─ w1 ───────────────────┐  ┌─ w2 ───────────────────┐ │
 │ │ 192.168.252.2          │  │ 192.168.252.3          │  │ 192.168.252.4          │ │
 │ │ control plane          │  │ worker node            │  │ worker node            │ │
 │ │                        │  │                        │  │                        │ │
 │ │ NodePort :30080 open   │  │ NodePort :30080 open   │  │ NodePort :30080 open   │ │
 │ │ kube-proxy rules       │  │ kube-proxy rules       │  │ kube-proxy rules       │ │
 │ │                        │  │                        │  │                        │ │
 │ │ (no web pods:          │  │ ┌─ pod A ────────────┐ │  │ ┌─ pod B ────────────┐ │ │
 │ │ control plane is       │  │ │ 10.244.190.65      │ │  │ │ 10.244.80.193      │ │ │
 │ │ tainted)               │  │ │ [nginx  :80]       │ │  │ │ [nginx  :80]       │ │ │
 │ └────────────────────────┘  │ └────────────────────┘ │  │ └────────────────────┘ │ │
 │                             └────────────────────────┘  │ ┌─ pod C ────────────┐ │ │
 │                                                         │ │ 10.244.80.194      │ │ │
 │                                                         │ │ [nginx  :80]       │ │ │
 │                                                         │ └────────────────────┘ │ │
 │                                                         └────────────────────────┘ │
 │                                                                                    │
 │                                                                                    │
 │ (2) the rules on the node that was hit pick a ready pod at random:                 │
 │     e.g. via w1 -> pod A (local) | pod B or C (over the pod network to w2)         │
 │     e.g. via cp1 -> A, B or C (cp1 has no web pod itself)                          │
 │                                                                                    │
 └────────────────────────────────────────────────────────────────────────────────────┘
```

- Reached by: anything that can reach a node IP, at `<node-ip>:30080`.
- The node that receives the request picks a ready pod at random. The pod can be on a different node.
- Weakness: the client chooses the node, so if it hard-codes `192.168.252.3` and `w1` dies, it fails.
  `.2:30080` and `.4:30080` would still work, but nothing tells the client to switch.
- Use it for: simple external access, labs, and as the base that load balancers send traffic to.

```yaml
spec:
  type: NodePort
  selector: {app: web}
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080      # optional; auto-assigned if omitted
```
```bash
k expose deploy web --port=80 --type=NodePort
```

## 4. LoadBalancer

NodePort plus a request for an external load balancer in front of the nodes.

```
                          ┌────────────────────────────────┐
 ┌─────────────────┐ (1)  │ External load balancer         │  outside the cluster; made by a
 │ External client │─────►│ EXTERNAL-IP 203.0.113.10:80    │  cloud or MetalLB, not Kubernetes
 │ (your Mac)      │      │ health-checks the nodes        │
 └─────────────────┘      └────────────────────────────────┘
                                           │ (2) picks a healthy node (stage 1, done by the LB)
               ┌───────────────────────────┬───────────────────────────┐
               │                           │                           │
═══════════════╪═══════════════════════════╪═══════════════════════════╪══════════════ cluster boundary
               ▼                           ▼                           ▼
 ┌─ CLUSTER ──────────────────────────────────────────────────────────────────────────┐
 │ ┌─ cp1 ──────────────────┐  ┌─ w1 ───────────────────┐  ┌─ w2 ───────────────────┐ │
 │ │ 192.168.252.2          │  │ 192.168.252.3          │  │ 192.168.252.4          │ │
 │ │ control plane          │  │ worker node            │  │ worker node            │ │
 │ │                        │  │                        │  │                        │ │
 │ │ NodePort :30080 open   │  │ NodePort :30080 open   │  │ NodePort :30080 open   │ │
 │ │ kube-proxy rules       │  │ kube-proxy rules       │  │ kube-proxy rules       │ │
 │ │                        │  │                        │  │                        │ │
 │ │ (no web pods:          │  │ ┌─ pod A ────────────┐ │  │ ┌─ pod B ────────────┐ │ │
 │ │ control plane is       │  │ │ 10.244.190.65      │ │  │ │ 10.244.80.193      │ │ │
 │ │ tainted)               │  │ │ [nginx  :80]       │ │  │ │ [nginx  :80]       │ │ │
 │ └────────────────────────┘  │ └────────────────────┘ │  │ └────────────────────┘ │ │
 │                             └────────────────────────┘  │ ┌─ pod C ────────────┐ │ │
 │                                                         │ │ 10.244.80.194      │ │ │
 │                                                         │ │ [nginx  :80]       │ │ │
 │                                                         │ └────────────────────┘ │ │
 │                                                         └────────────────────────┘ │
 │                                                                                    │
 │                                                                                    │
 │ (3) kube-proxy on that node picks a ready pod (stage 2), on any node.              │
 │                                                                                    │
 └────────────────────────────────────────────────────────────────────────────────────┘
```

- Reached by: anything that can reach the load balancer, at the single `EXTERNAL-IP`.
- Two real choices happen: the load balancer picks the node (stage 1), then kube-proxy picks the pod (stage 2).
- The load balancer can health-check nodes, so a dead node stops receiving traffic and the client's address doesn't change.
- **On the kubeadm VM lab:** nothing provides the load balancer, so `EXTERNAL-IP` stays `<pending>`.
  The NodePort part still works, exactly as in section 3. The IP `203.0.113.10` above is only an
  example from a documentation range.
- Use it for: production external access in a cloud, where you want a stable address and failover.

```yaml
spec:
  type: LoadBalancer
  selector: {app: web}
  ports:
  - port: 80
    targetPort: 80
```
```bash
k expose deploy web --port=80 --type=LoadBalancer
```

## 5. What happens when a node dies

```
  NodePort, client hard-coded to w1        LoadBalancer in front
  ─────────────────────────────────        ──────────────────────────────────
  client ──► w1 (DOWN)   ✗ times out       client ──► LB ──┬─► w1 (DOWN)  skipped
                                                           ├─► w2         ✓ serves A/B/C
  client ──► w2          ✓ works, but                      └─► cp1        ✓
             only if the client switches
             to it by itself                 the client keeps using one address
```

- Inside the cluster, a ClusterIP has no entry node, so only pods on the dead node are lost.
- After a node goes NotReady, its pods are removed from the Service endpoints and the Deployment starts
  replacements elsewhere. The time this takes depends on the timeouts in your cluster.

## 6. Side by side

| | ClusterIP | NodePort | LoadBalancer |
|---|---|---|---|
| Reachable from | inside the cluster | node IPs, outside too | one external IP |
| Address | `10.x.x.x`, `web` (DNS) | `<node-ip>:3xxxx` | `<external-ip>:80` |
| Has a ClusterIP | yes | yes | yes |
| Opens a port on nodes | no | yes | yes (automatically) |
| Needs something extra | no | no | a provider (cloud or MetalLB) |
| Chooses the node (stage 1) | n/a (already on one) | the client | the load balancer |
| Node failover | n/a | none | yes (health checks) |
| Chooses the pod (stage 2) | kube-proxy | kube-proxy | kube-proxy |
| On the VM lab | works | works | NodePort works, IP `<pending>` |

## 7. Port names (a common mix-up)

```
Mac ──► nodePort 30080 ──► Service port 80 ──► targetPort 80 ──► container (containerPort 80)
        (node)             (Service)            (pod/container)
```
- `port`: the port on the Service (its ClusterIP).
- `targetPort`: the port on the pod's container. A wrong `targetPort` gives "connection refused" and an
  otherwise healthy-looking Service.
- `nodePort`: the port on each node (NodePort and LoadBalancer only).
- `containerPort` is informational metadata on the pod. It does not open or restrict anything.

## 8. Related things (not Service types)

- **`externalIPs`:** a field on any Service that makes kube-proxy accept traffic for an IP you choose.
  It creates and manages nothing, and the IP has to be routed to a node by something else. In the
  pictures above it is just another address on one node, like `192.168.252.3:80`, so the node is still
  the client's choice. `k expose ... --external-ip=IP` sets it.
- **Ingress / Gateway API:** HTTP-level routing (hostnames, paths, TLS) in front of Services. It needs a
  controller, which is usually exposed through a NodePort or LoadBalancer Service.
- **ExternalName:** a different Service type that returns a DNS alias, with no proxying.
- **Headless Service** (`clusterIP: None`): no virtual IP and no balancing. DNS returns the pod IPs
  directly.

## 9. Useful commands

```bash
k get svc,endpoints web                  # endpoints should list the ready pod IPs
k describe svc web
k expose deploy web --port=80 --target-port=80 --type=NodePort $do > svc.yaml
k create service nodeport web --tcp=80:80 --node-port=30080 $do
k run test --rm -it --image=busybox --restart=Never -- wget -qO- -T 2 web
```
(`$do` is `--dry-run=client -o yaml`; in zsh it needs `setopt SH_WORD_SPLIT`.)

## 10. CKA relevance

The CNCF curriculum (Services and Networking, about 20%) lists "Understand ClusterIP, NodePort,
LoadBalancer service types and endpoints". Check the current curriculum on the CNCF site, since it changes.

- Know all three, and be able to create each with `k expose --type=...` or YAML.
- Know `port` vs `targetPort` vs `nodePort`.
- Know that every Service finds its pods by label selector, and `k get endpoints <svc>` is the first check.
- Know that `<pending>` on a LoadBalancer in a cluster with no provider is normal.
- You are not expected to build the load balancer itself (MetalLB, HAProxy, cloud setup).
