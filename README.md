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

---

# Part 1 — The Troubleshooting Playbook

## Step 1 — Widen the lens (scope the problem)

```bash
# Are the nodes healthy? (Age/when they came up tells you if they rebooted)
kubectl get nodes -o wide

# EVERYTHING at once — look for a shared failure time across namespaces
kubectl get pods -A -o wide

# Quick status census
kubectl get pods -A --no-headers | awk '{print $4}' | sort | uniq -c

# Most recent events cluster-wide = the timeline of the incident
kubectl get events -A --sort-by='.lastTimestamp' | tail -50
```

**Heuristic:**

- One namespace broken → app/config issue.
- Many namespaces broke at the same time → cluster infra (nodes rebooted, CNI/kube-proxy down, DNS, API server, host limits).
- High restart counts on *everything* → systemic, not per-app.

## Step 2 — Inspect the infrastructure layer

```bash
# Node conditions: ready? memory/disk/PID pressure?
kubectl describe node <node> | grep -A10 Conditions

# System pod health = the "plumbing" of the cluster
kubectl get pods -n kube-system -o wide
kubectl get ds -A           # daemonsets: kindnet/calico (CNI), kube-proxy, node-exporter...
```

**The plumbing layer:** `kube-proxy` (ClusterIP/NodePort NAT), CNI/kindnet (pod-to-pod), and CoreDNS (name resolution).

## Step 3 — Read the crashing pod's real story

```bash
kubectl describe pod <pod> -n <ns>                        # State, Last State, ExitCode, events
kubectl logs <pod> -n <ns> --tail=100                     # current logs
kubectl logs <pod> -n <ns> --previous --tail=100          # why the LAST run died
kubectl logs <pod> -c <container> -n <ns>                 # specific container (incl. initContainers)
kubectl get events -n <ns> --sort-by='.lastTimestamp'
```

### Decode the exit code / reason

| Signal | Meaning |
|---|---|
| ExitCode `137` / `OOMKilled` | Memory limit hit — add resources, check memory spikes |
| `CrashLoopBackOff` | App starts then dies — read `--previous` logs |
| `ImagePullBackOff` / `ErrImagePull` | Bad image name/tag, registry auth, or network to registry |
| `Pending` | Unschedulable — check node resources, taints, PVC |
| `ContainerCreating` stuck | Volume mount, CNI, or image pull hanging |
| Liveness probe failed | App is up but *not healthy* (can't even reach its own healthz) |
| `Connection refused` / `i/o timeout` to `10.96.0.1:443` | **Cannot reach API server via ClusterIP → kube-proxy (or CNI) is broken** |
| `Unknown` status | Kubelet lost contact with the node |

## Step 4 — Prove the dependency layers (never assume)

```bash
# 1) Can pods reach ClusterIP services? (if this fails → kube-proxy is broken)
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- -T 5 http://kubernetes.default:443/api

# 2) Does DNS work?
... -- nslookup kubernetes.default

# 3) Do services have healthy endpoints?
kubectl get endpoints -A
kubectl get endpointslices -A   # v1 replacement

# 4) Are kube-proxy / CNI daemonsets actually Ready?
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

**Key deduction:** if Daemonsets like `kube-proxy` are down on worker nodes, pod→ClusterIP NAT is gone, and **every pod that calls the API server, a service, or LoadBalancer will fail with timeouts** — all at once, across all namespaces. This single fact explains most "everything is broken" incidents.

## Step 5 — Chase the root cause of the root cause

When you see ugly kernel/limit errors, go to the host:

```bash
# "too many open files" / fsnotify errors (= fd or inotify limits exhausted)
sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches
cat /proc/sys/fs/file-nr
docker exec <kind-node> sh -c 'ulimit -n; cat /proc/self/limits'

# OOM / kernel events
dmesg | tail -50
journalctl -k --since "1 hour ago" | grep -E 'oom|killed process'
```

## Step 6 — Fix in the right order (rebuild the dominoes)

1. **Fix the foundational layer first** (limits → kube-proxy/CNI/DNS). Collateral pods often self-heal.
2. Restart the plumbing:
   - `kubectl -n kube-system rollout restart ds kube-proxy`
   - `kubectl rollout restart ds <cni-daemonset>`
3. Then force clean restarts of the workloads that were crashing:
   - `kubectl -n <ns> rollout restart deploy <name>...`
   - `kubectl -n <ns> delete pod <statefulset-pod>` (StatefulSets need pod delete, not rollout)
4. Watch each: `kubectl -n <ns> rollout status deploy <name> --timeout=180s`
5. **Persist fixes that die on reboot** (this is how incidents recur):
   - `/etc/sysctl.d/99-inotify.conf` for inotify limits
   - `/etc/docker/daemon.json` `default-ulimits` for fd limits

## Step 7 — Verify + make durable

```bash
kubectl get pods -A | grep -v Running   # should be empty
curl -sk https://<loadbalancer-ip>      # real-world client check
kubectl get app -n argocd               # also check apps' own health views
```

## 60-Second Triage Checklist

```bash
kubectl get nodes -o wide
kubectl get pods -A | awk '{print $4}' | sort | uniq -c
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl get pods -n kube-system -o wide      # coredns, kube-proxy, CNI
kubectl get ds -A
```

Then drill into the pod with `describe` + `logs --previous`, decode the exit code, test ClusterIP/DNS, and fix **order: infra → plumbing → workloads → persist**.

---

# Part 2 — Deep-Dive: CrashLoopBackOff

## What it actually means

A container **starts, then exits**, repeatedly. Kubernetes follows a restarter loop with an **exponential backoff** (10s → 20s → 40s → … capped at 300s), which is why the pod appears to wait longer between restarts over time (`RESTARTS` bumps, `STATUS` = `CrashLoopBackOff`). A pod in CrashLoop is *running its main process and dying* — different from `ImagePullBackOff` (never started) or `Pending` (never scheduled).

## Diagnostic drill (in this exact order)

```bash
# 1) THE most important single command
kubectl describe pod <pod> -n <ns>
```

Read this part of the output carefully:

```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error              <- exit reason
  Exit Code:    1                  <- the real signal
  Started:      ... Finished: ...  <- how long it lived (instant crash vs after 30s?)
```

```bash
# 2) What did the dying process actually print?
kubectl logs <pod> -n <ns> --previous --tail=100     # the run that just died
kubectl logs <pod> -n <ns> --tail=50                 # current attempt

# 3) Multi-container pods: always target the right container
kubectl logs <pod> -c <container> -n <ns> --previous
kubectl logs <pod> -c <init-0> -n <ns>               # initContainer hangs/stuck?

# 4) The event log is the crash timeline
kubectl get events -n <ns> --sort-by='.lastTimestamp'
```

## Exit-code decoder

| Exit code | Almost always means |
|---|---|
| `0` | Process ran and exited cleanly (job-style) — pod shouldn't be long-running |
| `1` | App-level startup/config/panic error — **read the logs** |
| `2` | Bad CLI usage/args/env |
| `137` | **OOMKilled** (SIGKILL) — hit memory `resources.limits.memory` |
| `139` | Segfault — native crash, corrupted binary, incompatible flags |
| `143` | SIGTERM (graceful shutdown during eviction/rollout — only a problem if it loops) |
| `255` | Container/runtime-level failure (init crash, bad seccomp, exec format) |
| `Unknown` | Kubelet lost contact (node issue, not app issue) |

**Rule:** if the container lives ~seconds before dying it's often probe-related (killed at 3-10s); if it lives minutes it's usually a slow leak/OOM or a satellite connection drop.

## The 10 most common root causes (tick these off)

1. **Bad config in the pod** — wrong args, missing env, bad YAML indentation. Validate offline: `kubectl apply --dry-run=client -f -` (syntax) and lint with `kubeconform` (schema) before you even hit the cluster.
2. **Secrets/ConfigMaps missing or wrong key** — env `valueFrom secretKeyRef` with a typo = container starts, errors, dies. `kubectl get secret <name>` to verify keys exist.
3. **Permission model angry** — `runAsNonRoot: true` + `runAsUser: 999` but the app expects to write `/var/lib/...` or bind a port <1024; `readOnlyRootFilesystem: true` + code that writes logs to `/`.
4. **hardened seccomp/caps** — `capabilities.drop: ["ALL"]` breaks apps needing `setcap`/`NET_BIND_SERVICE` unless `NET_BIND_SERVICE` is added back.
5. **Liveness probe wrong** — probe path/port wrong, or `full=true` semantics too strict. Check `kubectl get deploy <d> -o yaml | grep -A15 livenessProbe` vs what the process actually listens on.
6. **OOM** — `exit 137`; raise memory limit, `kubectl top pod -n <ns>`, look at `kubectl describe node` for `MemoryPressure`.
7. **Its dependency is unreachable** — app can't reach DB / redis / the API server → panic loop.
8. **InitContainer never completes** — the app container never even starts. Check `kubectl logs <pod> -c <initContainer>`.
9. **Port conflict** — app binds 8080 but something else is already there, or it tries to bind a port above the low-range setting.
10. **Duplicate/mismatched image tag** — `:latest` that now crashes, or an image built for another platform (exec format error, exit 255).

## Fixing it (and verifying fast)

```bash
# If Deployment/StatefulSet: force a clean rollout (kills the backoff timer too)
kubectl -n <ns> rollout restart deploy <name>
kubectl -n <ns> rollout status deploy <name> --timeout=180s

# Debug while it's crash-looping WITHOUT breaking the workload:
kubectl run <name>-debug --image=nicolaka/netshoot --restart=Never --rm -it -- /bin/bash
# Or attach an ephemeral debug container to the live crashlooping pod (container must be running):
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --target=<container>

# Inspect the victim's filesystem before it dies:
kubectl cp <ns>/<pod>:<path> ./local 2>/dev/null   (or exec + cat)
```

**Verification trap:** after a fix, restarts stop but the pod may sit in `CrashLoopBackOff` briefly because the kubelet still remembers a previous backoff — `rollout restart` (or delete the pod for a StatefulSet) resets it.

---

# Part 3 — Deep-Dive: Networking

## The layered model (know what each layer does)

```
Client → NodePort/LoadBalancer (32222 / 172.18.0.200)
              │  kube-proxy NAT
              ▼
        Service ClusterIP (10.96.x.x)   ← kube-proxy iptables/ipvs DNAT
              │  CNI (kindnet/calico) route the pod IP
              ▼
        Pod Endpoint (10.244.x.x)    ← CNI overlay
              │  CNI + host routing
              ▼
        Another pod's process
```

Breakdown:

- **host/node IP** — physical/docker level
- **CNI** (kindnet/calico/cilium) — assigns `Pod IP`, routes pod↔pod and pod↔node. Broken CNI ⇒ pods can't ping *anything*, `get pods` shows no IPs.
- **kube-proxy** — implements `ClusterIP`, `NodePort`, `LoadBalancer` (backend selection via Service→Endpoints). Broken kube-proxy ⇒ ClusterIP/NodePort/LB calls get connection timeouts while pod↔pod via direct IP still works.
- **CoreDNS** — resolves `svc.namespace.svc.cluster.local`; search domains + `ndots:5` affect name lookups.
- **MetalLB** — announces external `LoadBalancer` VIPs (L2/ARP). Broken LB ⇒ VIP advertises nothing even though the Service exists.
- **Ingress** — HTTP routing in front of Services (e.g. nginx-ingress).

## Diagnostic drill — walk the hops one at a time

```bash
# HOP 0: is the Service even populated? (selector mismatch kills this silently)
kubectl get endpoints <svc> -n <ns>          # if ENDPOINTS column empty → label selector mismatch
kubectl get endpointslices -n <ns> <svc>
kubectl get svc <svc> -n <ns> -o yaml        # check selector: matchLabels

# HOP 1: pod → ClusterIP (the kube-proxy test - THE key test)
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- -T 5 http://kubernetes.default:443/api
# HTTP 200/400/401 => kube-proxy + CNI OK. hang/timeout => kube-proxy or CNI layer.

# HOP 2: pod → pod (the CNI test)
kubectl run nettest --image=nicolaka/netshoot --rm -it --restart=Never -- \
  ping -c 3 <other-pod-ip>

# HOP 3: DNS test
... -- nslookup <svc>.<ns>.svc.cluster.local
... -- cat /etc/resolv.conf     # dnsPolicy=ClusterFirst adds cluster.local + ndots

# HOP 4: external → NodePort
curl -v http://<node-ip>:<nodeport>

# HOP 5: external → LoadBalancer (MetalLB test)
curl -vk https://<lb-ip>
kubectl -n <metallb-ns> get pods -l app.kubernetes.io/name=metallb   # speakers alive?
```

## The "everything times out" classifier

| Symptom | Number-one suspect |
|---|---|
| Pod can't reach `10.96.0.1:443` (API svc), logs `i/o timeout` | **kube-proxy down on that node** |
| Pod↔pod fine, but no ClusterIP works | kube-proxy |
| `get pods` shows no Pod IPs, all pings fail, DNS down | **CNI** (kindnet/calico) |
| Internal names resolve, FQDN works but short name doesn't | search domain / `ndots` issue |
| Service exists, endpoints empty | selector mismatch |
| LoadBalancer VIP never responds externally | **MetalLB speaker** (or LB assigned to a node that's down) |
| Works from one node, not another | per-node daemon (kube-proxy/CNI/speaker) broken only there |

That last row is important: in a common incident pattern, `kube-proxy` was fine on the control-plane but dead on all workers — so pods on workers couldn't reach any ClusterIP, and every namespace exploded except control-plane-hosted things (CoreDNS, etcd, kube-apiserver stayed green).

## Verify the plumbing directly (inside a node)

```bash
# kube-proxy healthy? NAT rules present?
docker exec <kind-node> sh -c 'iptables -t nat -L | grep -i kube-system'

# Daemonsets that must be 1/1 everywhere:
kubectl get ds -A
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl get pods -n kube-system | grep -E 'kindnet|coredns'
```

## Common root causes (and the fix shape)

1. **kube-proxy CrashLooping** (+ fsnotify/FD errors) → host limits → bump `fs.inotify.*`, restart `ds kube-proxy`.
2. **selector mismatch** → fix labels on the pod template or service selector.
3. **service `port` vs `targetPort`** wrong → fix to match the container port.
4. **NetworkPolicy** too strict → `kubectl get netpol -A`; `kubectl port-forward` still works (it bypasses via apiserver) — a good isolation test: works with port-forward, fails in-cluster.
5. **MetalLB speaker dead after node restart** → `kubectl -n metallb-system rollout restart ds speaker`.
6. **kube-proxy in ipvs mode vs iptables mode mismatch** with a weird kernel → check `--proxy-mode`.
7. **DNS broken** (CoreDNS pods down or loop) → check CoreDNS deployment (that's often also CNI or a forward loop to an inaccessible upstream).

## The one per-node dependency you should memorize

> **kube-proxy = every ClusterIP / NodePort / LoadBalancer.** If it dies on a node, that node's pods cannot talk to services or the API server — they will all look like "CrashLoopBackOff with i/o timeouts". `kubectl get pods -n kube-system -l k8s-app=kube-proxy` is the **first thing to check** whenever crash loops are widespread.

---

# Worked example — a real incident (the command trail)

1. `kubectl get pod -n argocd` → many unhealthy + huge restart counts, all resting around "67m ago". **Signal:** cluster-wide, timed event.
2. `kubectl get nodes -o wide` → all `Ready`. Not a node-down.
3. `kubectl get pods -A -o wide` → broken pods in *every* namespace (MetalLB, cert-manager, prometheus, nginx). **Cascade**, one shared cause.
4. `kubectl logs argocd-server --previous` → `dial tcp 10.96.0.1:443: i/o timeout`. Pods can't reach API server via ClusterIP.
5. `kubectl get pods -n kube-system -l k8s-app=kube-proxy` → **kube-proxy Error/CrashLoop on all 3 workers**. Found the link.
6. `kubectl logs <kube-proxy>` → `fsnotify watcher init: too many open files`. Root cause leads to host limits.
7. Host: `uptime` → `1:11` (**laptop rebooted ~1h ago**); `sysctl fs.inotify.*` → `128/65536` defaults, nothing persisted → boot storm exhausted the quota → kube-proxy could never init its file watcher.
8. **Fix chain:** bump + persist inotify sysctls → `rollout restart ds kube-proxy` → ClusterIP verified with throwaway busybox pod → `rollout restart` the affected deployments + delete the StatefulSet controller pod → restored the remaining workloads. Verified all pods Running + LB reachable.

**Why this worked:** nothing was fixed by editing app manifests. The first broken link (kube-proxy / host inotify limits) was repaired, and the cascade resolved itself.
