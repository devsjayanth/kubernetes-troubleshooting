# kubernetes-troubleshooting-zero-to-hero
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

Dev, below is a practical, interview-ready L2 Support preparation pack for Kubernetes infrastructure, monitoring, troubleshooting, Kafka, and ArgoCD. Focus on explaining **how you detect, investigate, mitigate, escalate, and prevent recurrence**.

---

# 1. Mindset

For every scenario, use this structure:

1. **Detect**  
   - Alert, dashboard, user report, log error, or monitoring notification.

2. **Assess impact**  
   - Is the service down?
   - Are users affected?
   - Is one pod affected or all pods?
   - Is it one namespace, one node, or the whole cluster?

3. **Validate and collect evidence**  
   - Kubernetes events
   - Pod logs
   - Previous logs
   - Metrics
   - Node status
   - Deployment rollout status
   - Service endpoints

4. **Isolate root cause**  
   - Application issue?
   - Configuration issue?
   - Resource issue?
   - Node issue?
   - Network issue?
   - Storage issue?
   - Dependency issue, such as Kafka, database, or external API?

5. **Take immediate corrective action**  
   - Restart pod or deployment
   - Roll back bad release
   - Scale replicas
   - Increase resource limits
   - Cordon or drain bad node
   - Fix ConfigMap/Secret
   - Fix image pull issue
   - Escalate if outside L2 scope

6. **Prevent recurrence**  
   - RCA
   - Alert improvement
   - Runbook update
   - Resource tuning
   - Probe tuning
   - Deployment strategy improvement
   - Load testing
   - Documentation

Good interview phrase:

> “My approach is evidence-based. I first check the alert and impact, then validate using Kubernetes events, logs, metrics, and object status. I isolate whether the issue is application, configuration, resource, node, network, or dependency related. Then I apply the safest immediate mitigation, escalate if required, and complete RCA with preventive actions.”

---

# 2. Kubernetes High Availability Concepts

## 2.1 Pod High Availability

Key components:

| Concept | Purpose |
|---|---|
| Deployment replicas | Maintains desired number of pods |
| Readiness probe | Controls whether pod receives traffic |
| Liveness probe | Restarts pod if app is stuck or unhealthy |
| Startup probe | Protects slow-starting apps from premature liveness failure |
| Pod anti-affinity | Prevents all replicas from landing on same node |
| Topology spread constraints | Distributes pods across zones/nodes |
| PriorityClass | Helps critical pods get scheduled during resource pressure |
| PodDisruptionBudget | Protects availability during voluntary disruptions |
| Resource requests | Help scheduler place pods correctly |
| Resource limits | Prevent one pod from consuming too much resource |

Interview answer:

> “For pod high availability, I ensure the Deployment has enough replicas, readiness probes are correctly configured, pods are distributed across nodes or zones, and PDBs protect availability during node maintenance. I also check requests and limits so pods are scheduled properly and do not cause node pressure.”

---

## 2.2 Service High Availability

For services, focus on:

- Readiness probes must pass before pod receives traffic.
- Service selector must match pod labels.
- Endpoints must show healthy pods.
- Ingress/load balancer health checks must point to correct port/path.
- Graceful shutdown should be handled by application.
- `terminationGracePeriodSeconds` should allow in-flight requests to finish.
- `preStop` hook can help with connection draining if needed.

Useful commands:

```bash
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl get endpointslices -n <namespace>
kubectl describe svc <service-name> -n <namespace>
```

If service is unavailable, always check endpoints first.

---

## 2.3 Resource Requests and Limits

### Requests

- Used by scheduler for placement.
- CPU request helps determine node scheduling.
- Memory request helps determine node scheduling.

### Limits

- CPU limit can cause throttling.
- Memory limit can cause OOMKill if exceeded.

Important point:

> “CPU throttling usually makes the application slow. Memory limit breach usually kills the container.”

### QoS Classes

| QoS Class | Description |
|---|---|
| Guaranteed | Requests equal limits for all containers |
| Burstable | Some requests/limits set, but not guaranteed |
| BestEffort | No requests or limits |

BestEffort pods are most likely to be evicted under node pressure.

---

## 2.4 HPA: Horizontal Pod Autoscaler

HPA scales pods based on metrics such as CPU, memory, or custom metrics.

Check HPA:

```bash
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>
```

Common HPA issues:

| Symptom | Likely Cause |
|---|---|
| `<unknown>` targets | Missing resource requests |
| Metrics unavailable | metrics-server issue |
| HPA not scaling up | Target threshold not reached |
| HPA at max replicas | Need capacity review or performance fix |
| Custom metrics missing | Custom metrics adapter issue |

Example answer:

> “If HPA is not scaling, I check whether pods have CPU/memory requests, whether metrics-server is healthy, whether the HPA target is being breached, and whether the max replica count is too low.”

---

## 2.5 VPA: Vertical Pod Autoscaler

VPA recommends or updates CPU/memory requests/limits.

Important interview point:

> “I would not use VPA in auto mode for CPU/memory together with HPA on the same metrics because they can conflict. VPA is useful for right-sizing recommendations, but changes should be reviewed carefully in production.”

---

## 2.6 Pod Disruption Budgets

PDB protects availability during voluntary disruptions such as:

- Node drain
- Cluster upgrade
- Maintenance
- Node replacement

Check PDB:

```bash
kubectl get pdb -n <namespace>
kubectl describe pdb <pdb-name> -n <namespace>
```

Example:

```yaml
minAvailable: 2
```

or:

```yaml
maxUnavailable: 1
```

Important:

> “If a node drain is blocked, I check whether a PDB is preventing eviction. I do not bypass PDB blindly because it may impact service availability.”

---

# 3. Monitoring and Alerting for L2 Support

## 3.1 What to Monitor

### Cluster level

- API server health
- etcd health, if visible
- Control plane latency
- Node Ready status
- Node CPU/memory/disk pressure
- Kubernetes events

### Node level

- CPU usage
- Memory usage
- Disk usage
- PID pressure
- Network errors
- Kubelet status
- Container runtime status

### Pod level

- Restart count
- OOMKills
- CrashLoopBackOff
- Pending pods
- ImagePullBackOff
- CreateContainerConfigError
- Evicted pods
- Terminating pods
- CPU throttling
- Memory working set
- Probe failures

### Workload level

- Deployment replica mismatch
- Rollout failures
- Unavailable replicas
- HPA status
- PDB status

### Service level

- HTTP 5xx errors
- Latency
- Timeout errors
- Endpoint count
- Ingress errors
- DNS resolution failures

### Middleware level

- Kafka broker health
- Under-replicated partitions
- Offline partitions
- Consumer lag
- Producer errors
- Request latency
- Broker disk usage

---

## 3.2 Important Alerts to Mention

| Alert | Meaning | First Check |
|---|---|---|
| PodCrashLooping | Pod repeatedly restarting | `kubectl describe pod`, previous logs |
| PodOOMKilled | Container exceeded memory limit | Pod events, memory metrics |
| PodPending | Pod not scheduled | Describe pod events, node capacity |
| NodeNotReady | Node not healthy | `kubectl describe node`, kubelet status |
| NodeMemoryPressure | Node memory low | Node metrics, evicted pods |
| NodeDiskPressure | Node disk pressure | Disk usage, eviction events |
| DeploymentReplicasMismatch | Desired replicas not available | Deployment status, pod events |
| RolloutFailed | Deployment rollout stuck | `kubectl rollout status` |
| HPAAtMax | HPA reached max replicas | Traffic/load, resource limits, app performance |
| ServiceEndpointMissing | No healthy endpoints | Readiness probes, service selector |
| KafkaUnderReplicatedPartitions | Kafka replication issue | Broker status, disk, network |
| KafkaConsumerLag | Consumers falling behind | Consumer group, app performance |

---

## 3.3 Useful Prometheus-style Queries

Exact metric names may vary, but these are common.

### Pod restarts

```promql
increase(kube_pod_container_status_restarts_total{namespace="prod"}[15m]) > 0
```

### OOMKilled containers

```promql
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
```

or:

```promql
increase(container_oom_events_total[15m]) > 0
```

### Pending pods

```promql
kube_pod_status_phase{phase="Pending"} == 1
```

### Node not ready

```promql
kube_node_status_condition{condition="Ready",status="true"} == 0
```

### Deployment unavailable replicas

```promql
kube_deployment_status_replicas_unavailable > 0
```

### Pod CPU usage

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="prod"}[5m])) by (pod)
```

### Pod memory usage

```promql
sum(container_memory_working_set_bytes{namespace="prod"}) by (pod)
```

### Memory usage versus limit

```promql
sum(container_memory_working_set_bytes{namespace="prod"}) by (pod)
/
sum(kube_pod_container_resource_limits{resource="memory",namespace="prod"}) by (pod)
```

---

# 4. Essential Kubernetes Commands for L2 Support

## 4.1 Pod Troubleshooting

```bash
kubectl get pods -A -o wide
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -c <container-name>
kubectl logs <pod-name> -n <namespace> --timestamps
```

## 4.2 Events

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
kubectl get events -n <namespace> --field-selector type=Warning
```

## 4.3 Resource Usage

```bash
kubectl top nodes
kubectl top pods -n <namespace>
kubectl top pods -n <namespace> --containers
```

## 4.4 Deployments and Rollouts

```bash
kubectl get deploy -n <namespace>
kubectl describe deploy <deployment-name> -n <namespace>
kubectl rollout status deploy/<deployment-name> -n <namespace>
kubectl rollout history deploy/<deployment-name> -n <namespace>
kubectl rollout undo deploy/<deployment-name> -n <namespace>
kubectl rollout restart deploy/<deployment-name> -n <namespace>
```

## 4.5 Services and Endpoints

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl get endpoints -n <namespace>
kubectl get endpointslices -n <namespace>
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
```

## 4.6 Nodes

```bash
kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes
kubectl cordon <node-name>
kubectl uncordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

## 4.7 Configuration Issues

```bash
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>
kubectl describe configmap <configmap-name> -n <namespace>
kubectl describe secret <secret-name> -n <namespace>
```

## 4.8 Storage

```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pvc <pvc-name> -n <namespace>
kubectl describe pv <pv-name>
```

## 4.9 Autoscaling and PDB

```bash
kubectl get hpa -A
kubectl describe hpa <hpa-name> -n <namespace>
kubectl get vpa -A
kubectl get pdb -A
```

## 4.10 Debugging

```bash
kubectl exec -it <pod-name> -n <namespace> -- sh
kubectl exec -it <pod-name> -n <namespace> -- bash
kubectl port-forward <pod-name> -n <namespace> <local-port>:<container-port>
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
kubectl debug -it <pod-name> -n <namespace> --image=nicolaka/netshoot --target=<container-name>
```

---

# 5. Symptom-Based Troubleshooting Matrix

## 5.1 Pod Stuck in Pending

### Detection

- Alert: Pod pending for too long
- `kubectl get pods`
- Deployment shows unavailable replicas

### Checks

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace>
kubectl describe nodes
kubectl top nodes
kubectl get pvc -n <namespace>
```

### Common Causes

| Cause | Evidence |
|---|---|
| Insufficient CPU | FailedScheduling: insufficient cpu |
| Insufficient memory | FailedScheduling: insufficient memory |
| Node taints | pod tolerated? |
| Node selector/affinity mismatch | scheduling constraint not satisfied |
| PVC not bound | PVC Pending |
| Too many resource requests | requests exceed node allocatable |

### Corrective Actions

- Check if requests are too high.
- Scale cluster or node group if using autoscaler.
- Fix node selector/affinity/taint issue.
- Fix PVC/storage class issue.
- Temporarily reduce requests only after understanding impact.
- Escalate to platform team if cluster capacity issue.

### Prevention

- Use namespace ResourceQuota and LimitRange.
- Use VPA recommendations carefully.
- Test manifests before production.
- Monitor node capacity and scheduling failures.

---

## 5.2 Pod in CrashLoopBackOff

### Detection

- Alert: high restart count
- `kubectl get pods`
- Deployment unavailable

### Checks

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> --timestamps
kubectl get events -n <namespace>
```

### Common Causes

| Cause | What to Check |
|---|---|
| Application startup failure | Previous logs |
| Missing ConfigMap/Secret | Events, environment variables |
| Bad image | Image tag, entrypoint |
| Probe failing too early | Liveness/startup probe config |
| Dependency unavailable | DB, Kafka, API, DNS |
| Permission issue | Filesystem, service account, secret mount |
| Invalid configuration | App config, environment variables |

### Corrective Actions

- If caused by bad deployment, roll back:

```bash
kubectl rollout undo deploy/<deployment-name> -n <namespace>
```

- If config issue, fix ConfigMap/Secret and restart:

```bash
kubectl rollout restart deploy/<deployment-name> -n <namespace>
```

- If dependency issue, validate backend service.
- If probe issue, adjust probe timing carefully.

### Prevention

- Add startup probe for slow apps.
- Validate config before deployment.
- Use canary/blue-green rollout.
- Add dependency health checks.

---

## 5.3 OOMKilled Pod

### Detection

- Pod restarts
- Event shows `OOMKilled`
- Exit code 137
- Memory metrics near limit

### Checks

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl top pods -n <namespace> --containers
kubectl get events -n <namespace>
```

Look for:

```text
Last State: Terminated
Reason: OOMKilled
Exit Code: 137
```

### Common Causes

- Memory limit too low
- Memory leak
- JVM heap too large for container limit
- Sudden traffic spike
- Large cache or in-memory processing
- Missing or improper resource limits

### Corrective Actions

Immediate:

```bash
kubectl rollout restart deploy/<deployment-name> -n <namespace>
```

If safe and approved:

- Temporarily increase memory limit.
- Scale replicas if load-related.
- Roll back recent release if memory leak suspected.

For Java apps:

- Check JVM heap settings.
- Ensure heap is less than container memory limit.
- Use options like `-XX:MaxRAMPercentage=75` carefully.

### Prevention

- Memory profiling/heap dump.
- Load testing.
- Right-size limits.
- Add memory alert before OOM.
- Review application code for memory leak.

Interview answer:

> “For OOMKilled, I first confirm from pod events and exit code 137. Then I compare memory usage against limits. If it started after a release, I consider rollback. If usage is genuinely increasing, I check for memory leak or JVM misconfiguration. Immediate mitigation may be restart or limit increase, but the RCA must identify whether it is configuration, leak, or traffic-driven.”

---

## 5.4 ImagePullBackOff or ErrImagePull

### Detection

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

### Common Causes

- Wrong image name
- Wrong tag
- Private registry authentication failure
- Missing imagePullSecret
- Network issue to registry
- Image deleted

### Checks

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get secret -n <namespace>
kubectl get deploy <deployment-name> -n <namespace> -o yaml | grep image
```

### Corrective Actions

- Fix image name/tag.
- Verify registry credentials.
- Recreate or attach correct `imagePullSecret`.
- Test pull from node if allowed.
- Roll back to previous image if bad release.

---

## 5.5 CreateContainerConfigError

### Detection

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Common Causes

- Missing ConfigMap
- Missing Secret
- Missing key in ConfigMap/Secret
- Wrong name referenced

### Corrective Actions

```bash
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>
kubectl describe configmap <name> -n <namespace>
kubectl describe secret <name> -n <namespace>
```

Fix missing object or key, then restart deployment.

---

## 5.6 Pod Evicted

### Detection

```bash
kubectl get pods -n <namespace> | grep Evicted
kubectl describe pod <pod-name> -n <namespace>
kubectl describe node <node-name>
```

### Common Causes

- Node memory pressure
- Node disk pressure
- PID pressure
- Too many BestEffort pods
- Ephemeral storage exceeded

### Checks

```bash
kubectl describe node <node-name>
kubectl top node <node-name>
kubectl get events -A --field-selector type=Warning
```

### Corrective Actions

- Identify pressured node.
- Cordon node if needed:

```bash
kubectl cordon <node-name>
```

- Drain node safely:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

- Clean disk or increase capacity.
- Check resource requests/limits.
- Move critical workloads to healthier nodes.

---

## 5.7 Pod Stuck in Terminating

### Detection

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

### Common Causes

- Application not handling SIGTERM
- Finalizers blocking deletion
- Volume detach issue
- Node issue
- Container runtime issue

### Checks

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o yaml
kubectl describe node <node-name>
```

### Corrective Actions

- Wait for grace period.
- Check application logs.
- Check node health.
- Check finalizers.
- Force delete only if safe and approved:

```bash
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

Important:

> “I avoid force-deleting stateful pods or pods attached to volumes unless I understand the data impact.”

---

## 5.8 Deployment or Rollout Failure

### Detection

```bash
kubectl rollout status deploy/<deployment-name> -n <namespace>
kubectl describe deploy <deployment-name> -n <namespace>
kubectl get events -n <namespace>
```

### Common Causes

- Bad image
- Probe failure
- Resource limit too low
- Missing ConfigMap/Secret
- Application crash on startup
- Progress deadline exceeded

### Corrective Actions

Check rollout:

```bash
kubectl rollout status deploy/<deployment-name> -n <namespace>
```

Check history:

```bash
kubectl rollout history deploy/<deployment-name> -n <namespace>
```

Rollback:

```bash
kubectl rollout undo deploy/<deployment-name> -n <namespace>
```

Rollback to specific revision:

```bash
kubectl rollout undo deploy/<deployment-name> -n <namespace> --to-revision=<revision>
```

### Prevention

- Use readiness probes.
- Use startup probes.
- Use rolling update with sensible `maxUnavailable` and `maxSurge`.
- Deploy to staging first.
- Use canary or blue-green where possible.

---

## 5.9 Service Unavailable or No Traffic

### Detection

- User reports service down
- 502/503/504 errors
- Monitoring shows no endpoints
- Ingress errors

### Checks

```bash
kubectl get svc <service-name> -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>
kubectl get pods -n <namespace> --show-labels
kubectl get pods -n <namespace> -o wide
```

### Common Causes

| Cause | Check |
|---|---|
| Selector mismatch | Service selector vs pod labels |
| Readiness probe failing | Pod readiness |
| Wrong targetPort | Service YAML |
| No healthy endpoints | Endpoints object |
| DNS issue | nslookup inside cluster |
| Ingress misconfiguration | Ingress rules/backend |
| NetworkPolicy blocking | Network policies |

### Corrective Actions

- Fix service selector.
- Fix readiness probe.
- Fix targetPort.
- Restart bad pods.
- Scale deployment if no endpoints.
- Check ingress controller logs.

---

## 5.10 High CPU Utilization

### Detection

- CPU alert
- Slow response time
- HPA scaling up
- Node CPU pressure

### Checks

```bash
kubectl top nodes
kubectl top pods -n <namespace> --containers
kubectl describe node <node-name>
kubectl logs <pod-name> -n <namespace>
kubectl get hpa -n <namespace>
```

### Common Causes

- Traffic spike
- Infinite loop
- Bad query
- Too many requests to dependency
- CPU limit too low causing throttling
- HPA maxed out

### Corrective Actions

- Scale deployment if safe:

```bash
kubectl scale deploy/<deployment-name> -n <namespace> --replicas=<number>
```

- Increase HPA max if approved.
- Roll back recent release if regression.
- Rate-limit traffic if possible.
- Cordon bad node if node-level issue.

---

## 5.11 Node CPU or Memory Pressure

### Detection

```bash
kubectl describe node <node-name>
kubectl top nodes
kubectl get events -A --field-selector type=Warning
```

Conditions to check:

```text
MemoryPressure
DiskPressure
PIDPressure
Ready
```

### Corrective Actions

- Identify heavy pods.
- Cordon node if unstable:

```bash
kubectl cordon <node-name>
```

- Drain node if safe:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

- Restart kubelet if allowed:

```bash
systemctl status kubelet
systemctl restart kubelet
```

- Escalate to platform team if node/kernel/runtime issue.

---

## 5.12 Node NotReady

### Detection

```bash
kubectl get nodes
kubectl describe node <node-name>
```

### Common Causes

- Kubelet stopped
- Container runtime issue
- CNI/network plugin issue
- Node out of resources
- Certificate issue
- Time synchronization issue
- Cloud provider issue
- Node rebooted

### Checks

On node, if access allowed:

```bash
systemctl status kubelet
journalctl -u kubelet -f
systemctl status containerd
journalctl -u containerd -f
df -h
free -m
uptime
```

### Corrective Actions

- Restart kubelet if allowed.
- Check runtime status.
- Check disk and memory.
- Cordon node to prevent new scheduling.
- Drain or reboot if needed.
- Escalate to infrastructure/platform team if control plane or CNI issue.

---

# 6. Kubernetes Probes: Very Important for Interviews

## Probe Types

| Probe | Purpose |
|---|---|
| Liveness probe | Restarts container when unhealthy |
| Readiness probe | Controls traffic to pod |
| Startup probe | Gives slow-starting containers time to start |

## Common Failure Patterns

| Symptom | Likely Probe Issue |
|---|---|
| Pod restarting repeatedly | Liveness probe failing |
| Pod running but no traffic | Readiness probe failing |
| App killed during startup | Need startup probe or longer initialDelaySeconds |
| Service intermittent | Readiness flapping |

Good answer:

> “If a pod is running but not receiving traffic, I check readiness probe failures and endpoints. If a pod keeps restarting, I check liveness probe and previous logs. For slow-starting applications, I consider startup probe to avoid premature restarts.”

---

# 7. Networking Troubleshooting Notes

## Common Service Issues

### 1. Service selector mismatch

```bash
kubectl get svc <svc> -n <ns> -o yaml
kubectl get pods -n <ns> --show-labels
```

### 2. No endpoints

```bash
kubectl get endpoints <svc> -n <ns>
```

If endpoints are empty:

- Pods not ready
- Selector mismatch
- Wrong port
- Pods crashing

### 3. DNS issue

From debug pod:

```bash
nslookup <service-name>.<namespace>.svc.cluster.local
```

### 4. Port mismatch

Check:

- Service port
- Target port
- Container port

### 5. Ingress issue

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
kubectl logs -n ingress-nginx <ingress-controller-pod>
```

### 6. NetworkPolicy issue

Check whether policy blocks:

- Pod-to-pod traffic
- Namespace traffic
- Ingress controller traffic
- DNS traffic

---

# 8. Storage Troubleshooting Notes

## Common PVC Issues

```bash
kubectl get pvc -A
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get pv
kubectl describe pv <pv-name>
```

### Common Causes

- StorageClass missing
- Provisioner failed
- No available PV
- Access mode mismatch
- Volume already bound to another pod
- Storage backend issue

### Important Point

For stateful workloads:

- Be careful deleting pods.
- Be careful deleting PVCs.
- Understand data retention.
- Escalate storage backend issues.

---

# 9. Kafka Monitoring and Troubleshooting

## 9.1 Key Kafka Concepts

| Concept | Meaning |
|---|---|
| Topic | Stream of messages |
| Partition | Parallelism unit |
| Replica | Copy of partition |
| Leader | Broker serving partition |
| Follower | Broker replicating partition |
| ISR | In-sync replicas |
| Consumer group | Group of consumers sharing partitions |
| Consumer lag | Difference between produced and consumed offset |
| Controller | Broker managing cluster metadata |

---

## 9.2 Important Kafka Metrics

| Metric | Meaning |
|---|---|
| UnderReplicatedPartitions | Some replicas not catching up |
| OfflinePartitionsCount | Partitions without leader |
| ActiveControllerCount | Should be 1 |
| ISRShrinkRate | Replicas leaving ISR |
| ISRExpandRate | Replicas rejoining ISR |
| RequestLatency | Broker processing delay |
| RequestHandlerIdlePercent | Broker capacity |
| NetworkHandlerIdlePercent | Network capacity |
| ConsumerLag | Consumers falling behind |
| DiskUsage | Broker storage capacity |
| LogFlushRate | Disk write performance |

---

## 9.3 Kafka Commands

If running in Kubernetes, commands may be executed inside Kafka pod:

```bash
kubectl exec -it <kafka-pod> -n <namespace> -- bash
```

Topic check:

```bash
kafka-topics.sh --bootstrap-server <broker>:9092 --describe --topic <topic-name>
```

Consumer group check:

```bash
kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --describe --group <group-name>
```

List consumer groups:

```bash
kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --list
```

List topics:

```bash
kafka-topics.sh --bootstrap-server <broker>:9092 --list
```

---

## 9.4 Kafka Consumer Lag

### Detection

- Consumer lag alert
- Application processing delay
- Increasing backlog
- Kafka exporter dashboard

### Checks

```bash
kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --describe --group <group-name>
```

Look for:

- CURRENT-OFFSET
- LOG-END-OFFSET
- LAG

### Common Causes

- Consumer too slow
- Not enough consumers for partitions
- Slow downstream dependency
- GC pauses
- Application errors
- Rebalance loop
- Message processing timeout

### Corrective Actions

- Scale consumers up to number of partitions.
- Fix slow downstream dependency.
- Increase consumer concurrency if supported.
- Increase `max.poll.interval.ms` if processing is legitimately long.
- Tune `session.timeout.ms` and `heartbeat.interval.ms` if rebalance occurs.
- Roll back recent consumer release if bug suspected.

Important:

> “You cannot scale consumers beyond the number of partitions in a consumer group and expect parallelism to increase.”

---

## 9.5 Under-Replicated Partitions

### Detection

- Kafka alert
- Broker metrics
- Topic describe output

### Common Causes

- Broker down
- Disk issue
- Network issue
- Broker overloaded
- Long GC pauses
- ZooKeeper/KRaft issue
- Replication configuration issue

### Checks

```bash
kafka-topics.sh --bootstrap-server <broker>:9092 --describe --topic <topic-name>
```

Check broker logs and pod status:

```bash
kubectl get pods -n <kafka-namespace>
kubectl logs <kafka-broker-pod> -n <kafka-namespace>
kubectl describe pod <kafka-broker-pod> -n <kafka-namespace>
kubectl top pods -n <kafka-namespace>
```

### Corrective Actions

- Restart unhealthy broker if safe.
- Check disk capacity.
- Check node pressure.
- Check network connectivity between brokers.
- Verify controller is active.
- Escalate to Kafka/platform team if data risk.

---

## 9.6 Kafka Broker CrashLoopBackOff

### Checks

```bash
kubectl get pods -n <kafka-namespace>
kubectl describe pod <kafka-broker-pod> -n <kafka-namespace>
kubectl logs <kafka-broker-pod> -n <kafka-namespace> --previous
```

### Common Causes

- Bad configuration
- Missing storage
- PVC issue
- Corrupt log data
- ZooKeeper/KRaft issue
- OOMKilled
- Incorrect listener configuration
- TLS/auth misconfiguration

### Important

Do not delete Kafka PVCs casually. Data loss risk.

---

## 9.7 Producer Timeouts

### Common Causes

- Broker unavailable
- Network issue
- Topic does not exist
- Authentication failure
- Authorization failure
- Broker overloaded
- Producer timeout too low
- `acks=all` with low `min.insync.replicas`

### Checks

- Application logs
- Kafka broker logs
- Topic describe
- Broker pod status
- Network connectivity
- Authentication/authorization config

---

# 10. ArgoCD Notes

## 10.1 What ArgoCD Does

ArgoCD is a GitOps tool. It compares:

- Desired state in Git
- Live state in Kubernetes cluster

Then it helps sync, monitor, and rollback applications.

Key concepts:

| Concept | Meaning |
|---|---|
| Application | ArgoCD object representing deployed manifests |
| Sync | Applying desired state from Git |
| Health | Whether Kubernetes resources are healthy |
| OutOfSync | Live cluster differs from Git |
| Degraded | Resources unhealthy |
| Rollback | Returning to previous revision |
| Hook | PreSync/PostSync jobs or actions |
| Self-heal | Automatically correcting drift |

---

## 10.2 ArgoCD Application Status

| Status | Meaning |
|---|---|
| Synced | Git and cluster match |
| OutOfSync | Git and cluster differ |
| Healthy | Resources are healthy |
| Progressing | Resources still deploying |
| Degraded | Something is unhealthy |
| Suspended | Automatic sync paused |

Important interview point:

> “Sync status and health status are different. An app can be Synced but Degraded if the manifests are applied but pods are unhealthy.”

---

## 10.3 ArgoCD Troubleshooting Commands

```bash
argocd app list
argocd app get <app-name>
argocd app diff <app-name>
argocd app sync <app-name>
argocd app rollback <app-name>
argocd proj list
argocd repo list
```

Kubernetes side:

```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
kubectl get application <app-name> -n argocd -o yaml
```

---

## 10.4 Common ArgoCD Issues

| Issue | Likely Cause |
|---|---|
| OutOfSync | Manual change in cluster or unapplied Git change |
| Sync failed | Invalid manifest, missing CRD, RBAC issue |
| Degraded | Pod CrashLoop, PVC pending, deployment unavailable |
| Hook failed | PreSync/PostSync job failed |
| Missing resources | Wrong path, kustomize/helm issue |
| Permission denied | Repo credentials or RBAC issue |

---

## 10.5 How to Explain ArgoCD if You Have Hands-On Experience

Use this structure:

> “I used ArgoCD to deploy and monitor applications using GitOps. I checked application sync and health status, compared Git versus live state, investigated OutOfSync or Degraded applications, and used rollback when a bad manifest or release caused issues. I also checked failed hooks and resource-level errors.”

---

## 10.6 If You Do Not Have Real ArgoCD Experience

Do not claim false experience. Say:

> “I have not owned ArgoCD in production, but I understand the GitOps workflow: sync, diff, health, rollback, and troubleshooting application status. I am comfortable working with Kubernetes manifests and can follow runbooks while learning ArgoCD quickly.”

This is safer and more professional.

---

# 11. RCA and Incident Closure

## RCA Template

Use this in interview:

```text
Incident:
Severity:
Service impacted:
Detection time:
Detection method:
Impact:
Timeline:
Immediate mitigation:
Root cause:
Contributing factors:
Evidence:
Permanent fix:
Preventive actions:
Monitoring/alerting improvements:
Runbook/documentation updates:
Owner and target date:
```

---

## Example RCA: OOMKilled Pods

```text
Incident: Payment API pods restarting due to OOMKilled
Severity: Sev2
Impact: Intermittent 5xx errors for 25 minutes
Detection: Prometheus alert PodOOMKilled
Immediate mitigation: Rolled back to previous version and temporarily increased memory limit
Root cause: New release introduced memory leak in cache handler
Evidence: Pod events showed OOMKilled, memory usage reached limit after release
Permanent fix: Code fix and heap dump analysis
Prevention: Add memory utilization alert at 80%, canary deployment, load test before release
```

---

## Example RCA: Deployment Failure

```text
Incident: Order service rollout stuck
Severity: Sev2
Impact: New pods not becoming ready, reduced capacity
Detection: DeploymentReplicasMismatch alert
Immediate mitigation: Rollback to last stable revision
Root cause: Readiness probe path changed incorrectly
Evidence: kubectl describe pod showed readiness probe failures
Permanent fix: Correct readiness path and add manifest validation
Prevention: Add staging smoke test and probe linting in CI
```

---

## Example RCA: Kafka Consumer Lag

```text
Incident: Order processing delayed
Severity: Sev2
Impact: Orders processed late by 20 minutes
Detection: Kafka consumer lag alert
Immediate mitigation: Scaled consumer replicas and restarted stuck consumers
Root cause: Slow database query caused processing delay
Evidence: Consumer lag increased, DB query latency high
Permanent fix: Query optimization and indexing
Prevention: Add DB latency alert and consumer lag threshold alert
```

---

# 12. Scenario-Based Interview Answers

## Scenario 1: Pod OOMKilled Repeatedly

Answer:

> “First, I check the alert and impact. Then I run `kubectl describe pod` and confirm the last terminated reason is OOMKilled, usually with exit code 137. I check previous logs and memory metrics to see whether memory was gradually increasing or spiked suddenly. If it started after a release, I consider rollback. If memory usage is genuinely too close to the limit, I may temporarily increase the memory limit or scale replicas as mitigation. For Java apps, I validate JVM heap configuration. For RCA, I check whether this is a leak, traffic increase, or misconfiguration, then add alerts and right-size limits.”

---

## Scenario 2: Deployment Stuck During Rollout

Answer:

> “I check `kubectl rollout status` and describe the deployment. Then I check new pod events and logs. If new pods are failing readiness or crashing, I identify the cause: bad image, config issue, probe issue, or resource issue. If impact is high, I roll back using `kubectl rollout undo`. After service recovery, I investigate the failed revision and document RCA.”

---

## Scenario 3: Service Returning 503

Answer:

> “I check whether the service has endpoints. If endpoints are empty, I check pod labels, service selector, and readiness status. If pods are not ready, I check probe failures, logs, and events. If endpoints exist, I check ingress, DNS, network policies, and backend dependencies. My immediate goal is to restore traffic by fixing readiness, restarting bad pods, rolling back bad release, or scaling if needed.”

---

## Scenario 4: Node NotReady

Answer:

> “I check node status using `kubectl get nodes` and `kubectl describe node`. I look for Ready condition, taints, allocatable resources, and node pressure. If I have node access, I check kubelet, container runtime, disk, memory, and system logs. I cordon the node to prevent scheduling. If safe, I drain it. If the issue is control plane, CNI, or storage infrastructure, I escalate to platform/L3.”

---

## Scenario 5: Pod Pending After Deployment

Answer:

> “I describe the pod and check FailedScheduling events. If it says insufficient CPU or memory, I check node capacity and resource requests. If the HPA scaled up, I check whether cluster capacity can support new pods. I also check taints, affinity, node selectors, and PVC status. Mitigation depends on cause: adjust requests, add capacity, fix scheduling constraints, or fix PVC.”

---

## Scenario 6: HPA Not Scaling

Answer:

> “I check HPA status with `kubectl describe hpa`. If targets show unknown, I verify pods have CPU/memory requests. I also check metrics-server health. If HPA is at max replicas, I check whether the max needs to be increased or whether the app has a performance issue. I also review recent traffic patterns and alerts.”

---

## Scenario 7: Kafka Consumer Lag Increasing

Answer:

> “I describe the consumer group and check lag. Then I determine whether consumers are too few, processing is slow, or downstream dependency is slow. If consumer instances are fewer than partitions, I scale consumers. If processing is slow, I check application logs, GC, DB latency, and downstream APIs. If rebalance is happening repeatedly, I check session timeout, heartbeat, and max.poll.interval.ms.”

---

## Scenario 8: Kafka Under-Replicated Partitions

Answer:

> “I check broker status, topic describe output, and ISR. Then I check whether a broker pod is down, restarting, or has disk/network issues. I check Kafka broker logs and node resources. If a broker is unhealthy and safe to restart, I restart it. If there is risk to data or storage, I escalate to Kafka/platform team.”

---

## Scenario 9: ArgoCD App OutOfSync

Answer:

> “I check the ArgoCD application status and diff. If the difference is expected, I sync after verifying the change. If sync fails, I check manifest errors, missing CRDs, RBAC, hooks, or repository access. If the application is Degraded, I troubleshoot the underlying Kubernetes resources, not just ArgoCD.”

---

## Scenario 10: Multiple Pods Restarting Across Different Nodes

Answer:

> “If pods restart on multiple nodes, I suspect application, config, dependency, or release issue rather than node issue. I check recent deployments, ConfigMaps, Secrets, logs, and events. If it started after a release, I consider rollback. If dependency is down, I check that dependency and network connectivity.”

---

## Scenario 11: One Pod Restarting on Same Node

Answer:

> “If only one pod on one node is affected, I compare with other replicas. I check node conditions, container runtime, disk, and pod logs. It could still be application-specific, but node-level checks are important.”

---

## Scenario 12: High CPU but No Pod Restart

Answer:

> “High CPU may cause slowness, not restarts. I check pod CPU usage, HPA, node CPU, and application logs. I verify whether HPA is scaling. If HPA is maxed, I consider increasing capacity or mitigating load. If CPU limit is causing throttling, I review limits and application performance.”

---

# 13. Common Interview Questions and Strong Answers

## Q1. How do you troubleshoot a CrashLoopBackOff pod?

> “I check pod events, exit code, and previous logs. Then I identify whether the cause is application error, missing config, dependency failure, probe misconfiguration, or resource issue. If it is due to a bad deployment, I roll back. If it is config-related, I fix ConfigMap/Secret and restart. After mitigation, I document RCA and improve alerts or probes.”

---

## Q2. How do you troubleshoot OOMKilled?

> “I confirm OOMKilled from pod events and exit code 137. I compare memory usage with limits. I check whether the issue started after a release, traffic spike, or configuration change. Immediate mitigation can be restart, rollback, or temporary limit increase. Root cause may be memory leak, incorrect JVM settings, or insufficient limits.”

---

## Q3. How do you identify why a pod is Pending?

> “I describe the pod and check scheduler events. Common causes are insufficient CPU/memory, taints, affinity rules, PVC not bound, or resource requests too high. I then check node capacity, PVC status, and scheduling constraints.”

---

## Q4. How do you troubleshoot a service with no traffic?

> “I check endpoints first. If endpoints are empty, I verify service selector, pod labels, readiness probes, and pod status. If endpoints exist, I check ports, DNS, ingress, network policies, and backend dependency health.”

---

## Q5. How do you handle a failed deployment?

> “I check rollout status and new pod events. If new pods are unhealthy and impact is ongoing, I roll back. Then I investigate logs, config, probes, image, and resource settings. RCA includes preventing recurrence through staging validation and better probes.”

---

## Q6. How do you ensure high availability during node maintenance?

> “Use multiple replicas, anti-affinity or topology spread, readiness probes, PDBs, and safe drain process. Before maintenance, check PDBs and ensure critical workloads can tolerate node loss.”

---

## Q7. How do you monitor Kubernetes proactively?

> “I monitor pod restarts, OOMKills, pending pods, node conditions, deployment replica mismatch, HPA status, CPU/memory saturation, service endpoints, latency, and error rates. Alerts should include severity and runbook references.”

---

## Q8. What is the difference between liveness and readiness probe?

> “Liveness determines whether Kubernetes should restart the container. Readiness determines whether the pod should receive traffic. A pod can fail readiness without restarting.”

---

## Q9. What is the difference between CPU request and CPU limit?

> “CPU request is used for scheduling and guaranteed capacity. CPU limit caps usage. If a container reaches CPU limit, it may be throttled, but not usually killed. Memory limit breach can cause OOMKill.”

---

## Q10. What do you do if HPA shows unknown metrics?

> “Check if pods define CPU/memory requests, verify metrics-server health, check HPA configuration, and validate custom metrics adapter if custom metrics are used.”

---

## Q11. What do you check for Kafka consumer lag?

> “I describe the consumer group and check lag. Then I verify consumer count, partition count, application processing time, downstream dependency latency, rebalance logs, and GC pauses.”

---

## Q12. What is ArgoCD used for?

> “ArgoCD is used for GitOps deployment and continuous reconciliation. It compares desired state in Git with live cluster state, helps sync applications, detect drift, and rollback.”

---

# 14. Escalation Criteria for L2 Support

Escalate when:

- Control plane is unhealthy.
- etcd issue.
- Cluster-wide network failure.
- CNI/plugin failure.
- Storage backend failure.
- Node kernel or OS issue.
- Security breach suspected.
- Data loss risk.
- Production change requires approval.
- Issue exceeds L2 access or SLA.
- Kafka data integrity is at risk.
- ArgoCD/RBAC/cluster policy issue requires platform team.

Good answer:

> “As L2, I handle application, workload, configuration, basic node, monitoring, and first-level middleware issues. I escalate cluster infrastructure, storage backend, network plugin, security, or data-loss scenarios to L3/platform teams with full evidence.”

---

# 15. What Not to Do in Production

Avoid saying you will do these without checks:

- Force delete stateful pods.
- Delete PVCs.
- Delete Kafka topics.
- Drain nodes without checking PDB.
- Roll back without understanding impact.
- Modify production config without change approval.
- Disable probes to make rollout pass.
- Increase limits blindly without RCA.
- Claim ArgoCD hands-on experience if you do not have it.

---

# 16. Real Operational Story Examples

Adapt these only if similar to your actual experience.

## Story 1: Memory Leak After Release

> “In one incident, application pods started restarting after a new release. Monitoring showed memory usage climbing until the limit, and pod events showed OOMKilled with exit code 137. I checked previous logs and confirmed the issue started after deployment. I rolled back to the previous version, which restored stability. Later, heap dump analysis showed a cache object not being cleared. We added memory alerts at 80% and introduced canary deployment.”

---

## Story 2: Rollout Failure Due to Readiness Probe

> “A deployment became stuck because new pods were not ready. `kubectl rollout status` showed progress deadline exceeded. Pod events showed readiness probe failures. The release had changed the health endpoint path. I rolled back the deployment, then worked with the dev team to correct the probe path. We added staging smoke tests to catch this earlier.”

---

## Story 3: Kafka Consumer Lag

> “We received an alert for Kafka consumer lag. The consumer group describe command showed increasing lag. Application logs showed slow database queries. We scaled consumers to match partition count, but lag still increased, so we investigated the database and found a missing index. After fixing the query, lag recovered. RCA included adding DB latency alert and consumer lag threshold.”

---

## Story 4: Node Memory Pressure Causing Evictions

> “A node became unstable and some pods were evicted. `kubectl describe node` showed MemoryPressure. I checked top pods and found one application consuming more memory than expected. I cordoned the node, allowed healthy pods to reschedule, and escalated to platform team for node investigation. RCA showed resource limits were missing for one workload.”

---

## Story 5: Pending Pods Due to Insufficient CPU

> “After HPA scaled up, several pods remained Pending. Describe pod showed FailedScheduling due to insufficient CPU. I checked node allocatable resources and found requests were too high. We temporarily adjusted requests after validation and worked with platform team on cluster capacity. RCA included right-sizing requests and enabling capacity alerts.”

---

# 17. Quick Final Checklist Before Interview

Be ready to explain:

- CrashLoopBackOff
- OOMKilled
- Pending pod
- ImagePullBackOff
- Evicted pod
- Terminating pod
- Node NotReady
- Deployment rollout failure
- Service endpoint issue
- HPA not scaling
- PDB blocking drain
- Kafka consumer lag
- Kafka under-replicated partitions
- ArgoCD sync/health/rollback

Remember commands:

```bash
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl top nodes
kubectl top pods -n <ns>
kubectl rollout status deploy/<deploy> -n <ns>
kubectl rollout undo deploy/<deploy> -n <ns>
kubectl get endpoints <svc> -n <ns>
kubectl describe node <node>
kubectl get hpa -n <ns>
kubectl get pdb -A
kubectl get pvc -A
```

Remember L2 flow:

```text
Detect → Assess impact → Collect evidence → Isolate cause → Mitigate → Escalate if needed → RCA → Prevent recurrence
```

---

# 18. High-Confidence Final Answer Template

Use this for almost any troubleshooting question:

> “First, I check the alert and service impact. Then I validate the Kubernetes object state using `kubectl get`, `describe`, events, and logs. I check whether the issue is isolated to one pod, one node, one namespace, or the whole cluster. I compare recent changes, deployments, config changes, and metrics. Once I identify the likely cause, I apply the safest mitigation, such as rollback, restart, scaling, or fixing configuration. If the issue is outside L2 scope or risks data loss, I escalate with evidence. Finally, I complete RCA and improve monitoring, runbooks, or configuration to prevent recurrence.”

---

If you want, I can also give you a **one-page last-minute revision sheet** or a **mock interview with expected answers** for this same role.
