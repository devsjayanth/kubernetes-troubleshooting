# kubernetes-troubleshooting
Learn how to troubleshoot the most common Kubernetes Issues

### ImagePullBackOff

When a kubelet starts creating containers for a Pod using a container runtime, it might be possible the container is in Waiting state because of ImagePullBackOff.

The status ImagePullBackOff means that a container could not start because Kubernetes could not pull a container image for reasons such as 

- Invalid image name or 
- Pulling from a private registry without imagePullSecret. 

The BackOff part indicates that Kubernetes will keep trying to pull the image, with an increasing back-off delay.

Kubernetes raises the delay between each attempt until it reaches a compiled-in limit, which is 300 seconds (5 minutes).

### CrashLoopBackOff

When you see "CrashLoopBackOff," it means that kubelet is trying to run the container, but it keeps failing and crashing. After crashing, Kubernetes tries to restart the container automatically, but if the container keeps failing repeatedly, you end up in a loop of crashes and restarts, thus the term "CrashLoopBackOff." 

This situation indicates that something is wrong with the application or the configuration that needs to be fixed.

### Pods not schedulable

In Kubernetes, the scheduler is responsible for assigning pods to nodes in the cluster based on various criteria. Sometimes, you might encounter situations where pods are not being scheduled as expected. This can happen due to factors such as node constraints, pod requirements, or cluster configurations.

1. Node Selector
2. Node Affinity
3. Taints
4. Tolerations

---
# K8s Troubleshooting Playbook

A practical playbook for finding the root cause of cluster problems and fixing them, in the right order.

## The Golden Rules

1. **Symptom ≠ root cause.** Don't fix crash-looping pods one by one — find the *first* broken thing. Everything else is usually collateral damage.
2. **Timestamps are the detective's best friend.** When did things break? If dozens of pods across many namespaces broke at the *same minute*, it's cluster-level (nodes, DNS, network proxy, API server, resources) — not your app.
3. **Read `events`, `logs`, and `describe` before touching anything.** The answers are almost always printed there.
4. **Change one thing at a time.** After each fix, re-check before moving on.

---

# Part 0 — The Dependency Stack (mental model)

Everything in Kubernetes is layered. A problem at a lower layer looks like a problem at every layer above it.

| Layer | What it includes | What it provides |
|---|---|---|
| **Workloads** | Deployments / StatefulSets / Pods | your apps |
| **Exposure** | Ingress / LoadBalancer / NodePort | external access (LB controller, ingress controller) |
| **DNS** | CoreDNS / cloud DNS | name resolution |
| **Service proxy** | kube-proxy or cloud-native | ClusterIP/NodePort/LB NAT |
| **Pod network (CNI)** | Calico, Cilium, Flannel, … | pod IPs, pod↔pod, pod↔node |
| **Control plane** | apiserver, etcd, scheduler | the API everything talks to |
| **Nodes + host** | kubelet, container runtime | compute, storage, limits |

**Key rule:** Diagnose from the bottom up. If the pod can't reach the API server, the network/LB/Ingress layers will look broken too — but they're not the cause.

---

# Part 1 — Triage in 3 commands

Do these before anything else:

```bash
# 1) Nodes healthy? (Age tells you if a node rebooted)
kubectl get nodes -o wide

# 2) Scope & status census across all namespaces
kubectl get pods -A -o wide
kubectl get pods -A --no-headers | awk '{print $4}' | sort | uniq -c

# 3) Event timeline = the incident's story
kubectl get events -A --sort-by='.lastTimestamp' | tail -50
```

## Decision tree (where is the real problem?)

| Observation | Point of failure |
|---|---|
| One namespace only | App / its config / its dependencies |
| Many namespaces, same timestamp | Cluster-level: nodes rebooted, CNI, service proxy, DNS, control plane, host limits |
| `Unknown` pod statuses | Kubelet / node contact lost |
| `No resources found` from apiserver at all, `kubectl` hangs | Control plane / apiserver / your kubeconfig |
| ClusterIP/NodePort/LB calls time out, pod↔pod fine | Service proxy (kube-proxy) |
| Pod↔pod also fails, pods have no IPs | CNI |
| DNS names fail, IPs work | DNS / resolv.conf / dnsPolicy |

---

# Part 2 — Per-Layer Diagnosis

## 2.1 Nodes

```bash
kubectl get nodes -o wide                 # Ready? Version? Internal IPs?
kubectl describe node <node> | grep -A10 Conditions   # MemoryPressure/DiskPressure/PIDPressure/Ready
kubectl describe node <node> | grep -A5  Resource    # capacity vs allocatable
kubectl top node                          # live usage (needs metrics-server / cluster monitoring)
```

Host-level checks (SSH to the node, or `docker exec`/`crictl` if it's a containerized node like kind/k3d):

```bash
uptime                    # rebooted recently? (high uptime change = reboot)
free -h; df -h            # resource pressure
cat /proc/sys/fs/file-nr  # overall FD usage
# fd/inotify limits — a silent, very common killer after restarts:
sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches
cat /proc/sys/net/core/somaxconn
journalctl -u kubelet --since "1 hour ago" | tail -50    # kubelet errors
crictl ps -a; crictl logs <id>            # container runtime view (containerd)
```

Common node failures:

- **NotReady + `Unknown` pods** — kubelet down, node rebooted, or network partition. Check kubelet service, container runtime, host load.
- **`MemoryPressure`/`DiskPressure`** — the scheduler will start evicting and pods get `OOMKilled`/`Evicted`. Fix on the node, not in the app.
- **`too many open files` / `fsnotify watcher init` errors in system pods** — inotify or file-descriptor limits exhausted (defaults are small and reset on reboot). Bump them and persist:
  - `sysctl -w fs.inotify.max_user_instances=1024 fs.inotify.max_user_watches=1048576`
  - persist in `/etc/sysctl.d/*.conf`; for containerized nodes also consider `--default-ulimit nofile` on the container runtime daemon.

## 2.2 Control plane (self-managed focus)

```bash
# Reachability + healthy components
kubectl get componentstatuses            # older, limited
kubectl get pods -n kube-system -o wide  # apiserver, etcd, controller-manager, scheduler, coredns
kubectl logs -n kube-system <apiserver-pod> --tail=100
# etcd health (inside the node that runs it):
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=... --key=... endpoint health
```

Control-plane symptoms:

- **`kubectl` hangs or `connection refused`** → apiserver down/unreachable, kubeconfig wrong, or the LB/firewall in front of it.
- **Pods stuck `Pending`** at scale → scheduler or apiserver overload.
- **`etcd` unhealthy** → read-heavy load, disk latency, or its own network issues. Everything downstream stalls.

> Managed clsters: you won't SSH to control-plane nodes. Use your provider's console/API health + CloudWatch/GCP/Azure logs; `kubectl` failures start at your kubeconfig and the provider's control-plane status.

## 2.3 CNI / pod network

```bash
kubectl get pods -A -o wide            # do pods have Pod IPs?
kubectl get ds -A                       # find your CNI daemonset (calico, cilium, flannel, …)
kubectl -n <cni-namespace> get pods -o wide
kubectl -n <cni-namespace> logs <cni-pod> --tail=50

# pod → pod reachability test (from inside any pod):
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -- \
  ping -c 3 <other-pod-ip>
```

Symptoms → cause:

- **No pod IPs** (`<none>` in the IP column) → CNI not assigning; the CNI daemonset or its config is broken.
- **Pod↔pod fails but everything else works** → overlay/VXLAN/BGP on the CNI, host routing, or pod CIDR conflicts.
- **New pods have IPs but can't communicate** → network policies (Part 4), CNI policy controller, or MTU mismatch.

## 2.4 Service proxy (kube-proxy & friends)

This layer implements **ClusterIP**, **NodePort**, and **LoadBalancer** traffic. If it dies on a node, every pod there times out talking to any Service.

```bash
# THE key test: can a pod reach a ClusterIP?
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- -T 5 http://kubernetes.default:443/api    # 200/400/401 = OK; timeout = proxy broken

# Is the proxy healthy everywhere?
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide   # or your cloud's equivalent
kubectl logs -n kube-system <kube-proxy-pod> --tail=50
```

Inside the node you can verify the actual rules:

```bash
iptables -t nat -L -n | grep -i kube-system     # iptables mode
ipvsadm -Ln                                      # IPVS mode
nft list ruleset | grep -i kube                  # nftables mode, newer distros
```

Interpretation:

- **Pod can't reach ClusterIP but CAN ping pods directly** → proxy layer down (kube-proxy crash-looping, dead on the node, or wrong mode).
- **One node broken, others fine** → per-node proxy failure there.
- **`fsnotify … too many open files` in the proxy's logs** → host inotify/FD limits (see 2.1).

## 2.5 DNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide   # CoreDNS
kubectl logs -n kube-system <coredns-pod> --tail=50

# from inside a pod:
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup <svc>.<ns>.svc.cluster.local
... -- cat /etc/resolv.conf     # dnsPolicy + search domains + ndots
```

Symptoms → cause:

- **FQDN resolves, short name doesn't** → search-domain issue or `ndots` (default 5 causes successive dots in names to try search domains; apps often need `ndots:1` or absolute names).
- **Nothing resolves, IPs work** → CoreDNS down, or CoreDNS itself can't reach upstreams (its `forward` block).
- **DNS resolution extremely slow** → keepalive/connection timeouts, CoreDNS overloaded.

## 2.6 External exposure (NodePort / LoadBalancer / Ingress)

```bash
kubectl get svc -A                          # types, ClusterIPs, EXTERNAL-IP, PORT(S)
kubectl get svc <svc> -n <ns> -o yaml       # type, selector, port/targetPort/nodePort
kubectl get endpoints <svc> -n <ns>         # populated? empty = selector mismatch
kubectl get endpointslices -n <ns>          # v1 replacement for endpoints

# LB layer: your controller's pods must be healthy
kubectl get pods -n <lb-controller-ns> -l <your-lb-selector>
kubectl logs -n <lb-controller-ns> <lb-pod> --tail=50
```

External checks:

```bash
curl -v http://<node-ip>:<nodeport>
curl -vk https://<lb-external-ip>
kubectl port-forward svc/<svc> 8080:80     # isolation test: if this works but ClusterIP fails → proxy
```

Symptoms → cause:

- **Service exists, `EXTERNAL-IP` never assigned** → your LB controller (MetalLB, cloud LB, etc.) is down or unprovisioned.
- **LB IP exists but no external response** → controller's per-node agents (`speaker`-style) dead; VIP not announced.
- **Endpoints empty** → label/selector mismatch; nothing will respond regardless of the network layer.
- **Port-forward works, in-cluster calls fail** → NetworkPolicy or the proxy/CNI layer.

---

# Part 3 — Symptom → Cause → Next Command

## Pod lifecycle statuses

| STATUS | Meaning | Next command |
|---|---|---|
| `Pending` | Not scheduled | `kubectl describe pod <p>` → look for scheduling events, taints, resources, PVC |
| `ContainerCreating` stuck | Image pull, volume mount, CNI | `kubectl describe pod <p>` + `kubectl get events` |
| `ImagePullBackOff` / `ErrImagePull` | Image name/tag/auth/registry | `kubectl describe pod <p>` (Events) |
| `CrashLoopBackOff` | Starts, then dies | See Part 5 |
| `Running` but `0/1` | Process up, not ready | `kubectl logs <p> --previous`; check readinessProbe |
| `Evicted` | Node pressure eviction | `kubectl describe node <node>` conditions |
| `OOMKilled` | Memory limit | `kubectl describe pod <p>`; `kubectl top pod` |
| `Unknown` | Kubelet lost contact | Check node `Ready`, kubelet logs, host reboot |
| `Terminating` stuck | Finalizers / unmountable volume / kubelet issue | `kubectl get pod <p> -o yaml | grep finalizers`; check node |

## Exit codes

| Code | Typical cause |
|---|---|
| `0` | Clean exit (bad for a long-running app — expected to stay up) |
| `1`/`2` | App error / bad args or config — **read `logs --previous`** |
| `137` | OOMKilled (SIGKILL); also a `SIGKILL` from the kubelet or runtime |
| `139` | Segfault (native crash, wrong platform image) |
| `143` | SIGTERM (graceful shutdown — only a loop if it repeats) |
| `255` | Runtime-level failure (init crash, seccomp, exec format) |

## Probe failures

| Probe type | Fails because | Fix direction |
|---|---|---|
| `livenessProbe` | Process up but unhealthy; wrong path/port; too strict (`full=true` style) | Align probe with reality; add startupProbe for slow boots |
| `readinessProbe` | App not ready to serve | Usually the app's fault (deps, slow start) |
| `startupProbe` missing + slow app | Killed during boot → looks like CrashLoop | Add `startupProbe` with generous `failureThreshold` |

---

# Part 4 — Walk-the-Hops Connectivity Flow

To isolate *exactly* where traffic dies, test each hop in order:

1. **Endpoint population** — `kubectl get endpoints <svc>` has entries? If not, it's selector/labels. Everything downstream is moot.
2. **Port mapping** — `spec.ports[].port` vs `targetPort` vs `nodePort` all correct vs the container's actual listening port?
3. **Pod → Service (ClusterIP)** — the kube-proxy test from Part 2.4.
4. **Pod → Pod IP** — CNI test.
5. **Pod → DNS** — name resolution test.
6. **external → NodePort** — proxy + node reachability.
7. **external → LB** — LB controller/announcement layer.
8. **Port-forward isolation** — `kubectl port-forward` goes through the apiserver, bypassing the pod network. If it works while in-cluster calls fail → NetworkPolicy / pod-network / proxy.

NetworkPolicy is a silent blocker — check early:

```bash
kubectl get networkpolicy -A
kubectl get netpol -n <ns> <name> -o yaml    # inspect allowed/denied selectors
```

| Works via port-forward, fails in-cluster | Suspect NetworkPolicy or CNI |
|---|---|
| Works from the node host, fails from pods | Node routing / kube-proxy localhost hairpin vs pod CIDR |
| Works on `node IP:NodePort`, fails on ClusterIP from pods | Proxy fine, but something weird in pod→proxy path (rare) |

---

# Part 5 — Deep-Dive: CrashLoopBackOff

A container **starts then exits** repeatedly, on an exponential backoff (10s→20s→…→300s). Differs from `ImagePullBackOff` (never started) and `Pending` (never scheduled).

## Diagnostic drill

```bash
kubectl describe pod <pod> -n <ns>
# Focus: "Last State: Terminated → Exit Code: N" and "how long it lived"
kubectl logs <pod> -n <ns> --previous --tail=100    # why the dying run failed
kubectl logs <pod> -n <ns> --tail=50                # current attempt
kubectl logs <pod> -c <container> -n <ns> --previous  # multi-container: be specific
kubectl logs <pod> -c <initContainer> -n <ns>         # initContainers that never finish
kubectl get events -n <ns> --sort-by='.lastTimestamp' # probe kills, OOM, etc.
```

**Timing clue:** dies in ~seconds → usually probes (killed 3–10s in). Dies after minutes → leak/OOM or a dependency dropping.

## The 10 most common root causes

1. **Bad config** — wrong args/env/indentation. Validate before applying: `kubectl apply --dry-run=client -f -` + schema-lint with `kubeconform`.
2. **Secret/ConfigMap missing or wrong key** — `valueFrom secretKeyRef` typo. Verify `kubectl get secret <name>`.
3. **Permissions** — `runAsNonRoot` + `runAsUser: 999` but writes to `/`; `readOnlyRootFilesystem: true` but the app writes. Add volumes/mounts or relax.
4. **Hardened seccomp/caps** — `capabilities.drop: ["ALL"]` breaks `setcap`/privileged ports; add the specific caps back.
5. **Liveness probe wrong** — path/port mismatch or overly strict semantics; add a `startupProbe` for slow apps.
6. **OOM** — exit `137`; raise `resources.limits.memory`, inspect `kubectl top`.
7. **Dependency unreachable** — app can't reach DB/redis/the API server; panics in a loop. Logs will show the dial error (e.g. `i/o timeout`); that's a network-layer problem, not an app bug.
8. **InitContainer stuck** — app container never starts; check the init container's logs; it often needs API/secret access.
9. **Port conflict / too many ports** — app can't bind.
10. **Wrong image** — bad tag, `:latest` regression, or wrong platform (exec format error, exit 255).

## Fix and verify

```bash
# Rollout restart resets the backoff timer too
kubectl -n <ns> rollout restart deploy <name>
kubectl -n <ns> rollout status deploy <name> --timeout=180s

# Debug without disrupting a Deployment:
kubectl run <name>-debug --image=nicolaka/netshoot --restart=Never --rm -it -- /bin/bash
# Ephemeral debug container on a running (crash-looping) pod:
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --target=<container>

# Copy files out of a dying pod for inspection:
kubectl cp <ns>/<pod>:<path> ./local
```

---

# Part 6 — Fix Ordering & Making It Permanent

## Rebuild the dominoes — bottom up

1. Host/Nodes (limits, disk, network).
2. Control plane reachability.
3. CNI (pod-to-pod).
4. Service proxy (ClusterIP).
5. DNS.
6. Workloads — **most collateral damage heals itself once 1–5 are fixed**. Only then `rollout restart` workloads that stay stuck.
7. External (LB/Ingress).

Restart plumbing cleanly:

```bash
kubectl -n kube-system rollout restart ds kube-proxy     # daemonsets
kubectl -n <cni-ns> rollout restart ds <cni-daemonset>
kubectl -n <ns> rollout restart deploy <name>            # deployments
kubectl -n <ns> delete pod <statefulset-pod>             # StatefulSets: delete pod, don't rolout
```

## Persist anything that dies on reboot

- **inotify/FD limits** → `/etc/sysctl.d/*.conf` (nodes).
- **Container-runtime ulimits** → daemon config (`containerd` config.toml / Docker `daemon.json` `default-ulimits`).
- **Systemd service env** → `/etc/systemd/system/<svc>.d/override.conf`.
- Re-verify after a reboot: `sysctl fs.inotify.max_user_instance` (this is the classic "breaks again every reboot" trap).

---

# Part 7 — Troubleshooting Toolkit (portable reference)

## Probe pods (throwaway, in any cluster)

```bash
# network + DNS probes
kubectl run nettest --image=nicolaka/netshoot --restart=Never --rm -it -- /bin/bash
# minimal, image-light (busybox lacks dig but has wget/nslookup in most distro tags)
kubectl run nettest --image=busybox:1.36 --restart=Never --rm -it -- /bin/sh
```

## The "is it healthy end-to-end" checklist

```bash
kubectl get nodes -o wide
kubectl get pods -A | awk '{print $4}' | sort | uniq -c
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl get pods -n kube-system -o wide        # api server, coredns, proxy, CNI
kubectl get ds -A                               # daemonsets must be 1/1 everywhere
kubectl get endpoints -A | grep -v 192|10|172  # empty endpoints = selector problem
```

## The one dependency to memorize

> **Service proxy = every ClusterIP / NodePort / LoadBalancer.** If it dies on a node, that node's pods cannot reach any Service or the API server — and every workload there looks like "CrashLoopBackOff with i/o timeouts." Check `kube-proxy`-type pods first whenever failure is widespread.

## Minor tips

- `kubectl get pods -A -w` to watch transitions.
- Use `--tail`, `--previous`, `-c` generously; logs beyond 1000 lines get truncated silently.
- `kubectl get pod -o yaml` shows the *spec* — compare `spec.containers[*].resources`, `securityContext`, and `probes` against what the app needs.
- When stuck, asking `kubectl get events` with `--field-selector involvedObject.kind=Pod` filters to useful signal.
