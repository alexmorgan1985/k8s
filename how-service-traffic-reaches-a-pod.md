# How a request to a Service ends up at a pod

Worked out on the kubeadm VM lab (`cp1` / `w1` / `w2`), with a 3-replica `web` Deployment and the
`svc-lab.yaml` Service (NodePort 30080 + externalIP 192.168.252.3).

## The finding in one paragraph

"The Service load balances" is only half the story. There are **two separate stages**, and only the
second one is done by Kubernetes:

1. **Client to node: the client's choice.** Without an external load balancer, the node that receives
   the request is whichever node IP the client used. `192.168.252.3` means `w1`, every time. Nothing in
   the cluster spreads clients across nodes or notices a dead node.
2. **Node to pod: kube-proxy's rules, in the kernel.** Whichever node receives the packet picks one of
   the **ready** pods at random, per new connection. The pod can be on any node, including a different
   one from the entry node.

```
client ──(stage 1: client's choice of IP)──► node ──(stage 2: iptables rules)──► pod
        no Kubernetes involvement                   "Service load balancing"
```

Why it's confusing: "load balancing" is used for stage 2, while "LoadBalancer" (the Service type) is
about stage 1. Cloud clusters hide stage 1 because the cloud provides a load balancer, so the gap only
shows up on local clusters. Inside the cluster there is no stage 1: a pod calling `web` is already on a
node.

## Diagram: hitting `192.168.252.3:80`

IPs and chain names below are from the real cluster.

```
 Mac (client)
   │  curl 192.168.252.3:80          new TCP connection
   ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ w1   node IP 192.168.252.3                                                │
│                                                                           │
│  packet arrives: dst = 192.168.252.3:80                                   │
│     │                                                                     │
│     ▼                                                                     │
│  iptables (nat table, in the kernel; programmed by kube-proxy)            │
│     │                                                                     │
│     │ KUBE-SERVICES   match: -d 192.168.252.3 --dport 80                  │
│     ▼                                                                     │
│  KUBE-EXT-LOLE4...    marks traffic for masquerade (source NAT)           │
│     │                                                                     │
│     ▼                                                                     │
│  KUBE-SVC-LOLE4...    picks ONE rule at random, per new connection:       │
│     │                                                                     │
│     ├─ 1/3 ─► KUBE-SEP-DXCF...  DNAT dst → 10.244.190.65:80 ─────┐        │
│     ├─ 1/3 ─► KUBE-SEP-LP3E...  DNAT dst → 10.244.80.193:80 ──┐  │        │
│     └─ 1/3 ─► KUBE-SEP-7MOM...  DNAT dst → 10.244.80.194:80 ─┐│  │        │
│                                                              ││  │        │
│   ┌──────────────────────────────────────────────────────────┼┼──┼──┐     │
│   │ pod web-...-tslbl  10.244.190.65 (on w1)  ◄──────────────┼┼──┘  │     │
│   │   nginx :80   ← local delivery, never leaves w1          ││     │     │
│   └──────────────────────────────────────────────────────────┼┼─────┘     │
│                                                              ││           │
└──────────────────────────────────────────────────────────────┼┼───────────┘
                                                               ││
          Calico pod network (10.244.0.0/16) between nodes     ││
                                                               ▼▼
┌───────────────────────────────────────────────────────────────────────────┐
│ w2   node IP 192.168.252.4                                                │
│                                                                           │
│   pod web-...-hlh55  10.244.80.193 :80  (nginx)                           │
│   pod web-...-jmq9j  10.244.80.194 :80  (nginx)                           │
└───────────────────────────────────────────────────────────────────────────┘

 Reply path: pod → source-NAT'd back through w1 → conntrack un-does the
             NAT → Mac sees the reply coming from 192.168.252.3:80.
```

Steps:
1. The Mac sends a TCP SYN to `192.168.252.3:80`. That IP belongs to `w1`, so the packet arrives there.
2. iptables on `w1` matches `KUBE-SERVICES`, which has a rule for the Service's external IP and port.
3. The packet goes through `KUBE-EXT-...` (marked for masquerade) into `KUBE-SVC-...`.
4. `KUBE-SVC-...` picks one pod chain using random probabilities (1/3, then 1/2, then the remainder).
5. The chosen `KUBE-SEP-...` chain rewrites the destination to the pod's IP and port (DNAT).
6. The kernel routes to the pod: local delivery if it is on `w1`, over the pod network if it is on `w2`.
7. The reply returns the same way and conntrack reverses the NAT.
8. For the rest of that connection, conntrack keeps sending packets to the same pod. The random
   choice happens once, at step 4.

The diagram draws Calico's cross-node transport as a plain box. I did not check whether it uses IP-in-IP,
VXLAN, or routing in this install.

## How the "random" split gives an even spread

The three rules in `KUBE-SVC-...` are checked in order:

```
rule 1: probability 0.333  → pod A      (takes 1/3)
rule 2: probability 0.500  → pod B      (half of the remaining 2/3 = 1/3 overall)
rule 3: no condition       → pod C      (the remaining 1/3)
```

It is random per connection, not round-robin, so 12 requests will not necessarily split 4/4/4.

## Evidence from the lab

kube-proxy mode (from its log and ConfigMap; `mode: ""` means the default):
```
"Using iptables Proxier"
```

Service state:
```
NAME  TYPE      CLUSTER-IP     EXTERNAL-IP    PORT(S)
web   NodePort  10.111.51.47   192.168.252.3  80:30080/TCP
Endpoints: 10.244.190.65:80, 10.244.80.193:80, 10.244.80.194:80
Session Affinity: None     External Traffic Policy: Cluster
```

Rules on `w1` (shortened; chain names change if the Service is recreated):
```
-A KUBE-SERVICES -d 10.111.51.47/32 -p tcp --dport 80   -j KUBE-SVC-LOLE4ISW44XBNF3G   # ClusterIP
-A KUBE-SERVICES -d 192.168.252.3/32 -p tcp --dport 80  -j KUBE-EXT-LOLE4ISW44XBNF3G   # externalIP
-A KUBE-NODEPORTS -p tcp --dport 30080                  -j KUBE-EXT-LOLE4ISW44XBNF3G   # NodePort
-A KUBE-SVC-LOLE4... -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-DXCF...
-A KUBE-SVC-LOLE4... -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-LP3E...
-A KUBE-SVC-LOLE4...                                                         -j KUBE-SEP-7MOM...
-A KUBE-SEP-DXCF... -p tcp -j DNAT --to-destination 10.244.190.65:80
```

To look yourself:
```bash
multipass exec w1 -- sudo iptables-save | grep default/web
multipass exec w1 -- sudo iptables -t nat -L KUBE-SVC-<id> -n
kubectl -n kube-system get cm kube-proxy -o yaml | grep mode
```

## Consequences

- **kube-proxy is not in the data path.** It only writes the rules. The kernel forwards the packets.
- **All three entry points share one pool.** ClusterIP, NodePort and externalIP all jump to the same
  `KUBE-SVC-...` chain, so they all balance across the same pods.
- **Every node knows every pod.** Each node's kube-proxy watches the same EndpointSlices, so
  `curl 192.168.252.2:30080` works even though `cp1` runs no `web` pod.
- **The externalIP adds no balancing.** `externalIPs` only adds another address that kube-proxy
  accepts. Remove it and pods are still balanced on `:30080`.
- **Single point of failure at stage 1.** `192.168.252.3` exists only on `w1`. If `w1` goes down, that
  address stops answering even though the pods on `w2` are fine. `.2:30080` and `.4:30080` still work, but
  a client that only knows `.3` is cut off. Nothing in Kubernetes moves the IP or checks node health.
- **Extra hop.** With `externalTrafficPolicy: Cluster` (the default), 2 of the 3 pods are on `w2`, so about 2/3 of
  requests entering `w1` cross to `w2`. The source address is also rewritten, so the pod sees the node's IP,
  not the client's.
- **Per connection, so keep-alive sticks.** One reused connection stays on one pod. Use `curl` in a loop to
  see the spread, not a browser.
- **Only Ready pods are in the rules.** A failing readinessProbe removes a pod from rotation.
- **`port-forward` is different.** `kubectl port-forward svc/web ...` tunnels to a single pod and is not
  load balanced.
- **`ping` doesn't work** against Services; they handle TCP/UDP only. Use `curl`.

## Closing the stage 1 gap (not needed for the CKA)

| Option | What it gives |
|---|---|
| Cloud LoadBalancer / MetalLB (`type: LoadBalancer`) | one stable IP, health checks, failover |
| HAProxy or nginx in front, balancing `.2:30080`, `.3:30080`, `.4:30080` | the same, and you can see how it works |
| keepalived floating IP | an IP that moves between nodes |
| Round-robin DNS | crude; clients may cache a dead address |

`externalTrafficPolicy: Local` removes the extra hop and preserves the client IP, but a node with no local
pod drops the traffic and the spread becomes uneven. It is meant to be used with a load balancer that
health-checks nodes.

## Try it

```bash
# see the spread: terminal 1
k logs -l app=web --prefix -f
# terminal 2, from the Mac (separate connections)
for i in $(seq 12); do curl -s -o /dev/null 192.168.252.3:30080; done

# or make each pod identify itself (lost when the pod restarts)
for p in $(k get pods -l app=web -o name); do
  k exec $p -- sh -c 'hostname > /usr/share/nginx/html/index.html'
done
for i in $(seq 12); do curl -s 192.168.252.3:30080; done

# prove the stage 1 dependency
multipass stop w1
curl -m 3 192.168.252.3:30080     # times out
curl -m 3 192.168.252.4:30080     # still works
multipass start w1
```

## Official docs

- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/) is the best match:
  kube-proxy programs each node to redirect Service traffic to an endpoint; iptables mode picks a backend at
  random by default; the same flow applies to NodePort and load-balancer traffic (with the client IP altered);
  `externalTrafficPolicy` and `internalTrafficPolicy`; `sessionAffinity: ClientIP`; load-balancer health checks
  against kube-proxy's `:10256/healthz`.
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/): NodePort "exposes the Service on
  each Node's IP at a static port"; LoadBalancer "exposes the Service externally using an external load balancer".
- Worth reading, not opened when this was written: the "Using Source IP" tutorial and the "Create an External
  Load Balancer" task.

No official page states the stage 1 point outright (that without a load balancer the client chooses the node).
It follows from the NodePort and LoadBalancer definitions, and is easy to miss.

## CKA relevance

- Know: Services find pods by label selector; check `k get endpoints <svc>`; only Ready pods are listed;
  `port` vs `targetPort` vs `nodePort`; test from inside the cluster with a `busybox` pod and `wget`.
- Understand but don't build: kube-proxy programs iptables/nftables rules; LoadBalancer needs a cloud
  provider (or MetalLB); Ingress needs a controller.
- The exam gives you a working cluster, so building a highly available entry point is not asked.
