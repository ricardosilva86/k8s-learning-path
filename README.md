# Kubernetes Deep-Dive Workbook — 52 Weeks

An advanced, one-topic-per-week program for an experienced Kubernetes engineer. Every week has: why it matters in production, how it maps to the core components (API server, etcd, controller-manager, scheduler, kubelet, kube-proxy/CNI, container runtime), what to read, what to build, and questions you should not be able to answer without reading source, KEPs, or running experiments.

---

## 0. How to use this workbook

**Time budget: ~3 hours/week**
- ~60 min — the *Read* item
- ~60 min — the *Do* item (hands-on; never skip this one)
- ~30 min — write the weekly note (template at the end)
- *Deeper* items are optional, for quiet weekends

**Lab setup (do once):**
- A `kind` cluster, 3 nodes, cgroup v2 host (Linux or Docker Desktop with cgroup v2)
- A sandbox EKS account for the cloud-specific labs (EBS, LBC, IRSA/Pod Identity, Karpenter)
- Tools: `kubectl`, `etcdctl`, `crictl`, `cilium CLI`, `hubble`, `cosign`, `kube-burner`, `kwok`, `kubebuilder`, `clusterctl`

**Standing references (bookmark once):**
- KEPs: `github.com/kubernetes/enhancements` — search by number or title
- Source: `github.com/kubernetes/kubernetes` — paths below are relative to the repo root
- Design docs & conventions: `github.com/kubernetes/community` (`contributors/devel/`, `sig-*`)
- Docs: `kubernetes.io/docs/reference`
- Operators: `book.kubebuilder.io`

**Books (each covers several weeks):**
- *Kubernetes in Action, 2nd ed.* — Lukša (internals, Modules 1–3)
- *Programming Kubernetes* — Hausenblas & Schimanski (Module 7)
- *Learning eBPF* and *Container Security* — Liz Rice (Modules 3, 4, 6)
- *Hacking Kubernetes* — Martin & Hausenblas (Module 6)
- *Kubernetes Operators* — Dobies & Wood (Module 7)
- *Production Kubernetes* — Rosso et al. (Module 8)

**Buffer weeks:** 18, 27, 49 are intentionally light. Use them to catch up.

**Rules:**
1. If a week slips, drop *Deeper*, never *Do*.
2. One note per week, same template, one repo. By week 52 you have a book you wrote.
3. KEP numbers are from memory of the enhancements repo; if one does not resolve, search by title.

---

## 1. Roadmap

| Module | Weeks | Theme |
|---|---|---|
| 1 | 1–7 | Control plane internals |
| 2 | 8–14 | Scheduling and resource management |
| 3 | 15–20 | Node and runtime |
| 4 | 21–28 | Networking |
| 5 | 29–32 | Storage |
| 6 | 33–38 | Security and access control |
| 7 | 39–45 | Controllers and operator development |
| 8 | 46–52 | Operating at scale + capstone |

Each module builds on the previous one. Skim the week titles now; anything unfamiliar is a candidate for advance reading.

---

## Module 1 — Control plane internals (Weeks 1–7)

### Week 1 — The API server request pipeline
**Production:** Every "kubectl hangs" or "admission is slow" incident lives in this chain: authn → authz → mutating admission → schema validation → validating admission → storage. The order explains why a mutating webhook can bypass a validating one and why RBAC errors look different from admission errors.  
**Core:** The API server is the only component that talks to etcd; everything else is a client.  
**Read:** docs *Controlling Access to the Kubernetes API*, *Extensible Admission Controllers*; source `staging/src/k8s.io/apiserver/pkg/server/config.go` (handler chain).  
**Do:** Run `kubectl apply --v=8` and map each step; enable audit logging at `Metadata` level and read one request's lifecycle.  
**Questions:**.  
1. In which exact phase does a `ValidatingAdmissionPolicy` run relative to webhooks?  
2. When etcd is reachable but slow, which timeout fires first and what does the client see?  
3. Why can a mutating webhook see a defaulted object but not a validated one?  

### Week 2 — etcd: Raft, MVCC and revisions
**Production:** Database growth, compaction/defrag, leader elections during disk-latency spikes, and the 8 GB default quota are the usual causes of control-plane death. Managed control planes hide this; you still feel it as API latency.  
**Core:** Every `resourceVersion` is an etcd revision.  
**Read:** etcd.io docs *Learning → Data model*, *Operations → Maintenance*; the Raft paper, sections 5–6 only.  
**Do:** With `etcdctl` on the kind cluster: `get / --prefix --keys-only`, snapshot, compact, defrag; watch `etcd_disk_wal_fsync_duration_seconds`.  
**Questions:**
1. What is the difference between compaction and defragmentation, and why does one not free disk space?  
2. How does fsync latency affect Raft heartbeats and leader stability?  
3. What does the etcd key for a Pod in namespace `foo` look like, and why does `--etcd-servers-overrides` matter for Events?  

### Week 3 — List/Watch, the watch cache and informers
**Production:** Controllers that "miss events", LIST storms after control-plane restarts, `too old resource version` errors. Expensive LISTs are the number-one cause of API server memory blowups.  
**Core:** The API server's watch cache sits between etcd and clients; client-side informers mirror it.  
**Read:** KEP-2340 (consistent reads from cache), KEP-3157 (watch list); source `staging/src/k8s.io/apiserver/pkg/storage/cacher/`.  
**Deeper:** *Programming Kubernetes* ch. 3.  
**Do:** Write a ~50-line client-go informer; kill the API server mid-watch; observe resync and bookmarks with `--v=5`.  
**Questions:**
1. What do `resourceVersion=0`, `resourceVersion=""` and a specific value each mean on a LIST?  
2. How does a watch bookmark work and why does it exist?  
3. What guarantee did "consistent read from cache" change?  

### Week 4 — CRD machinery and API extension
**Production:** Every operator you run (Cilium, Kyverno, Flux) depends on this. A broken conversion webhook makes an entire resource type unreadable — including by the garbage collector.  
**Core:** CRDs are served by the same API server but stored via a generic path; aggregated APIs are proxied to another server.  
**Read:** docs *Extend the Kubernetes API with CustomResourceDefinitions* (structural schemas, CEL validation, versions and conversion); `kubernetes/community` → API conventions doc.  
**Do:** Write a CRD with two versions and a conversion webhook; break the webhook and watch what `kubectl get` and GC do.  
**Questions:**
1. What is a structural schema and what can you not do without one?  
2. How does the storage version differ from served versions, and what does the storage version migrator do?  
3. When would you choose an aggregated API server over a CRD today?  

### Week 5 — Controller-manager, workqueues and leader election
**Production:** Two active controller replicas corrupting state, or a lease that never renews after a partition, are leader-election failure modes. Workqueue rate limiting is why a bad object retries forever without flooding the API.  
**Core:** kube-controller-manager is ~40 controllers in one binary, all on the same informer/workqueue pattern.  
**Read:** `kubernetes/community` → `sig-api-machinery/controllers.md`; source `staging/src/k8s.io/client-go/util/workqueue/`, `client-go/tools/leaderelection/`.  
**Do:** Run two copies of a controller against the same Lease; inject clock skew in one and observe.  
**Questions:**
1. How does a `Lease` prevent split-brain if clocks drift?  
2. What is the default rate limiter's backoff formula and when does it reset?  
3. Why does the workqueue deduplicate keys, and what does that imply for event ordering?  

### Week 6 — API Priority and Fairness (APF)
**Production:** One misbehaving controller or a CI loop of `kubectl get pods -A` can starve kubelets. APF decides who gets a 429.  
**Core:** Replaces `--max-requests-inflight`; lives in the API server filter chain.  
**Read:** docs *API Priority and Fairness*; KEP-1040.  
**Do:** Hammer the API with a LIST loop; watch `apiserver_flowcontrol_*` metrics; write a FlowSchema that isolates the offender.  
**Questions:**
1. How do FlowSchemas match requests, and what happens when nothing matches?  
2. What are "seats", and how does a LIST consume more than one?  
3. How would you protect kubelet heartbeats from a noisy namespace?  

### Week 7 — The scheduler framework
**Production:** Pending pods with cryptic reasons, scheduling latency at scale, and why multiple schedulers rarely make sense. Karpenter does not replace the scheduler; it reads the same constraints.  
**Core:** The scheduler binds pods by writing a `Binding` subresource; the kubelet only acts after that.  
**Read:** docs *Scheduling Framework*; source `pkg/scheduler/framework/` (start with `interface.go` and the `NodeResourcesFit` plugin).  
**Do:** Use `kube-scheduler-simulator`, or a custom `KubeSchedulerConfiguration` that disables a plugin; observe the difference.  
**Questions:**
1. Name the extension points in order and which ones can reject a node.  
2. What happens to a pod rejected by a `PostFilter` plugin?  
3. What is `percentageOfNodesToScore` and why does it matter on a 1000-node cluster?  

---

## Module 2 — Scheduling and resource management (Weeks 8–14)

### Week 8 — The resource model: requests, limits, QoS and cgroups v2
**Production:** CPU throttling at low utilization, OOM at 60% usage because of page-cache accounting, and QoS class deciding who dies first.  
**Core:** The kubelet translates requests/limits into cgroup values via the CRI; the scheduler only sees requests.  
**Read:** docs *Resource Management for Pods and Containers*; KEP-2570 (Memory QoS); kernel doc `sched-bwc.rst` (CFS bandwidth control).  
**Do:** Inspect `/sys/fs/cgroup/kubepods.slice/...` for one pod of each QoS class; reproduce throttling with `stress-ng`; watch `container_cpu_cfs_throttled_periods_total`.  
**Questions:**
1. How does CFS quota throttle at low average usage, and why does the period matter?  
2. What does `memory.high` do differently from `memory.max` in cgroups v2?  
3. Why is `cpu.weight` only relevant under contention?  

### Week 9 — Eviction, OOM and pressure
**Production:** Node-level eviction and kernel OOM kill are different mechanisms with different signals and outcomes (`Evicted` vs `OOMKilled`, reschedule vs restart).  
**Core:** The kubelet's eviction manager watches node conditions; the node controller reacts by tainting.  
**Read:** docs *Node-pressure Eviction*; source `pkg/kubelet/eviction/`; KEP-1287 (in-place pod resize).  
**Do:** Fill a node's memory gradually; record which pod dies, by which mechanism, and what the event says.  
**Questions:**
1. What are soft vs hard eviction thresholds and grace periods?  
2. How is `oom_score_adj` set per QoS class? Can a Guaranteed pod be OOM-killed by the kernel?  
3. What does in-place resize change about this picture?  

### Week 10 — Topology Manager, device plugins and Dynamic Resource Allocation
**Production:** GPU and NUMA-sensitive workloads. DRA is replacing device plugins and changes how the scheduler reasons about devices.  
**Core:** Device plugins talk to the kubelet over gRPC; DRA introduces `ResourceClaim` objects handled by the scheduler.  
**Read:** docs *Device Plugins*, *Control Topology Management Policies*; KEP-693 (Topology Manager), KEP-4381 (DRA structured parameters).  
**Do:** Deploy `kubernetes-sigs/dra-example-driver` in kind.  
**Questions:**
1. Why can a pod be scheduled yet fail with `TopologyAffinityError`?  
2. What problem does DRA's structured-parameters model solve that device plugins could not?  
3. How does the CPU Manager static policy interact with Guaranteed QoS?  

### Week 11 — Placement: affinity, topology spread and descheduling
**Production:** Uneven AZ distribution after scale-downs, pods that never spread because `maxSkew` is misunderstood, and the fact that the scheduler never moves a running pod.  
**Core:** Pure scheduler plugins; the descheduler is an external controller that evicts to trigger rescheduling.  
**Read:** docs *Pod Topology Spread Constraints* (incl. `matchLabelKeys`, KEP-3243); `kubernetes-sigs/descheduler` README.  
**Do:** Create skew on purpose; run the descheduler with `RemovePodsViolatingTopologySpreadConstraint`.  
**Questions:**
1. How does scoring differ between `DoNotSchedule` and `ScheduleAnyway`?  
2. Why does topology spread not count terminating pods by default, and what does `nodeTaintsPolicy` change?  
3. How does Karpenter consolidation interact with PDBs and topology spread?  

### Week 12 — Node autoscaling: Karpenter internals
**Production:** Provisioning latency, consolidation churn, disruption budgets, spot interruption handling, and bin-packing decisions that differ from Cluster Autoscaler.  
**Core:** Karpenter watches unschedulable pods (same signal CA uses), simulates scheduling, and creates `NodeClaim` objects reconciled into instances.  
**Read:** Karpenter docs *Concepts → Scheduling* and *Disruption*; source `kubernetes-sigs/karpenter` → `pkg/controllers/disruption/`.  
**Do:** In sandbox EKS: a NodePool with disruption budgets; trigger consolidation; trace the decisions in controller logs.  
**Questions:**
1. How does Karpenter simulate the scheduler without running it, and where can the simulation diverge?  
2. What is the difference between single-node and multi-node consolidation, and when does each trigger?  
3. How do NodePool disruption budgets and PDBs combine?  

### Week 13 — Pod autoscaling: HPA, VPA, metrics pipeline, KEDA
**Production:** HPA flapping, scale-down stabilization, resource vs custom vs external metrics APIs, and VPA conflicting with HPA on the same resource.  
**Core:** metrics-server scrapes the kubelet's `/metrics/resource`; HPA runs in controller-manager and reads through the aggregated API.  
**Read:** docs *Horizontal Pod Autoscaling* (algorithm section); source `pkg/controller/podautoscaler/replica_calculator.go`; KEDA docs *Concepts*.  
**Do:** Load-test a deployment; log HPA status conditions every 15 s; explain each decision.  
**Questions:**
1. How does HPA compute desired replicas when some pods are not ready or have no metrics?  
2. What exactly does the tolerance window suppress?  
3. How does KEDA scale to zero when HPA cannot?  

### Week 14 — Priority, preemption, quotas and disruption
**Production:** System pods evicted by batch jobs, quota denials with misleading messages, and the Eviction API as the mechanism behind drains.  
**Core:** Preemption is a scheduler `PostFilter`; the Eviction API is an API server subresource that consults PDBs.  
**Read:** docs *Pod Priority and Preemption*, *Disruptions* (Eviction API, unhealthy pod eviction policy), *Resource Quotas*.  
**Do:** Drain a node with a PDB at `maxUnavailable: 0`, then with `unhealthyPodEvictionPolicy: AlwaysAllow`.  
**Questions:**
1. What is `preemptionPolicy: Never`, and what can such a pod still do?  
2. Why does the quota admission plugin sometimes reject an update that does not increase usage?  
3. What does a drain do when a PDB has `maxUnavailable: 0`, and how does the unhealthy pod eviction policy change it?  

---

## Module 3 — Node and runtime (Weeks 15–20)

### Week 15 — kubelet internals: sync loop, PLEG, probes, graceful shutdown
**Production:** `PLEG is not healthy` node flaps, probe timeouts under load, pods killed before draining connections.  
**Core:** The kubelet watches its own pods via the API server, reconciles against the CRI, and reports status back.  
**Read:** source `pkg/kubelet/kubelet.go` (`syncLoop`), `pkg/kubelet/pleg/`; KEP-3386 (evented PLEG), KEP-2000 (graceful node shutdown).  
**Deeper:** *Kubernetes in Action* 2nd ed., pod lifecycle chapter.  
**Do:** tcpdump a Service during a rolling update; measure 5xx before and after adding a preStop sleep.  
**Questions:**
1. What is the evented PLEG and what did it replace?  
2. In what order do preStop, SIGTERM and endpoint removal happen, and why does that race cause 502s?  
3. How does graceful node shutdown use systemd inhibitor locks?  

### Week 16 — CRI and containerd
**Production:** Registry pull throttling, snapshotter disk usage, shim processes outliving containers, sandbox creation failures.  
**Core:** The kubelet is a CRI client; containerd implements `RuntimeService` and `ImageService`.  
**Read:** containerd docs *Architecture*; CRI proto at `staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto`.  
**Do:** On a node: `crictl pods`, `crictl inspectp`, `ctr -n k8s.io images ls`, `ps -ef | grep shim`; match everything to pods.  
**Questions:**
1. What is a pod sandbox at the CRI level, and which container holds the network namespace?  
2. How do `containerd-shim-runc-v2` processes relate to pods?  
3. How do image pull parallelism and `serializeImagePulls` affect node startup?  

### Week 17 — Linux isolation primitives and Pod Security Standards
**Production:** Deciding what to actually block with Kyverno: which capabilities matter, why `hostPath` and `privileged` are escapes, what `seccomp` `RuntimeDefault` blocks.  
**Core:** The kubelet asks the runtime to apply namespaces, cgroups, seccomp, AppArmor and capabilities per container.  
**Read:** *Container Security* ch. 4–9; docs *Pod Security Standards*; KEP-127 (user namespaces).  
**Do:** Write Kyverno policies equivalent to `restricted`; then escape a pod that has only `CAP_SYS_ADMIN` to prove why it is blocked.  
**Questions:**
1. Which capability makes a container effectively root on the host even without `privileged`?  
2. What does user-namespace support change about the threat model?  
3. Why is `hostPID` dangerous even with a read-only root filesystem?  

### Week 18 — (Buffer) Sandboxed runtimes and RuntimeClass
**Production:** Multi-tenant or untrusted workloads; the performance and compatibility costs of gVisor and Kata.  
**Core:** `RuntimeClass` maps to a CRI runtime handler and carries pod overhead the scheduler accounts for.  
**Read:** gVisor docs *Architecture Guide*; docs *Runtime Class*.  
**Do (optional):** Run gVisor (`runsc`) in kind.  
**Questions:**
1. How does gVisor intercept syscalls, and which workloads break because of it?  
2. What is `overhead` in a RuntimeClass and how does the scheduler use it?  
3. Can you use eBPF-based network policy inside a Kata VM?  

### Week 19 — Node lifecycle: leases, node controller, taints, drains
**Production:** Nodes going `NotReady` and pods taking five minutes to reschedule; `node.kubernetes.io/unreachable` tolerations and taint-based eviction.  
**Core:** The kubelet renews a Lease; the node lifecycle controller marks conditions and taints.  
**Read:** source `pkg/controller/nodelifecycle/`; docs *Taints and Tolerations* (taint-based eviction); `kubernetes/node-problem-detector` README.  
**Do:** Block the kubelet's API access with a NetworkPolicy; time every state change until pods reschedule.  
**Questions:**
1. What are the `--node-monitor-grace-period` and `tolerationSeconds` defaults, and how do they combine into observed failover time?  
2. What is the difference between cordon, drain, and a NoExecute taint for a StatefulSet pod?  
3. How does Node Problem Detector feed into this?  

### Week 20 — Images, garbage collection and ephemeral storage
**Production:** Disk-pressure evictions from logs and image layers, `emptyDir` sizing, container GC vs image GC thresholds.  
**Core:** The kubelet runs both GCs; ephemeral storage is enforced by the kubelet, not the runtime.  
**Read:** docs *Garbage Collection* (kubelet section), ephemeral storage section of the resources docs; source `pkg/kubelet/images/image_gc_manager.go`.  
**Do:** Fill a pod's ephemeral storage; watch eviction vs disk-pressure ordering.  
**Questions:**
1. How is ephemeral storage usage measured and why is it expensive?  
2. What happens with image GC under disk pressure when all images are in use?  
3. How do volume-based `emptyDir` limits differ from container limits?  

---

## Module 4 — Networking (Weeks 21–28)

### Week 21 — Pod networking and the CNI specification
**Production:** IP exhaustion, overlay vs native routing performance, and the fact that the CNI plugin is called once per sandbox and nothing checks it afterwards.  
**Core:** containerd invokes the CNI plugin when creating the sandbox; the kubelet only sees the result.  
**Read:** CNI spec (`github.com/containernetworking/cni/blob/main/SPEC.md`); AWS docs on VPC CNI ENI and prefix delegation; Cilium docs *IPAM*.  
**Do:** Run a CNI ADD by hand with `cnitool`.  
**Questions:**
1. What are ADD/DEL/CHECK and who calls them?  
2. How does IPAM differ between the AWS VPC CNI and Cilium cluster-pool mode, and what does ENI mode change?  
3. Why does a CNI failure leave a pod in `ContainerCreating` rather than `Failed`?  

### Week 22 — Cilium's eBPF datapath
**Production:** kube-proxy replacement, socket-level load balancing, Maglev, debugging with `cilium monitor` and `hubble` instead of tcpdump.  
**Core:** Replaces kube-proxy entirely; attaches to tc/XDP hooks instead of netfilter.  
**Read:** Cilium docs *eBPF Datapath*, *Kube-proxy replacement*; *Learning eBPF* ch. 8.  
**Do:** `cilium monitor -v`, `cilium bpf lb list`, `hubble observe` while hitting a ClusterIP; find where DNAT happens.  
**Questions:**
1. Where is a ClusterIP translated with socket LB enabled, and why does that break some tools?  
2. What is a Cilium identity and how is it derived?  
3. What are the tradeoffs of DSR vs SNAT for NodePort traffic?  

### Week 23 — kube-proxy modes and Service semantics
**Production:** Even without kube-proxy you need the contract Cilium reimplements: conntrack, session affinity, `externalTrafficPolicy`, `internalTrafficPolicy`, EndpointSlices.  
**Core:** kube-proxy is a controller watching Services and EndpointSlices and writing dataplane rules.  
**Read:** source `pkg/proxy/` (iptables and nftables); KEP-3866 (nftables), KEP-1672 (terminating endpoints), KEP-4444 (traffic distribution).  
**Deeper:** KubeCon talk "Life of a Packet" (Michael Rubin).  
**Do:** In a kind cluster with kube-proxy, dump iptables for one Service and follow a packet.  
**Questions:**
1. Why does `externalTrafficPolicy: Local` preserve source IP, and what does it do to load distribution?  
2. How do nftables and iptables modes differ in rule-count scaling?  
3. What are terminating endpoints and why do EndpointSlices carry them?  

### Week 24 — DNS: CoreDNS, ndots and node-local caching
**Production:** The `ndots:5` latency tax, conntrack races on UDP DNS, CoreDNS autoscaling, why NodeLocal DNSCache exists.  
**Core:** The kubelet writes `/etc/resolv.conf` into the pod; CoreDNS is just a Deployment watching Services.  
**Read:** CoreDNS `kubernetes` plugin docs; docs *DNS for Services and Pods* (ndots); NodeLocal DNSCache KEP-1024.  
**Do:** `strace -e sendto` a curl from a pod; count queries; fix it with `dnsConfig`.  
**Questions:**
1. How many queries does `curl api.github.com` generate from a pod with default DNS config?  
2. What is the conntrack insert race and how does NodeLocal DNS avoid it?  
3. How does the CoreDNS `kubernetes` plugin keep its view in sync?  

### Week 25 — NetworkPolicy, CiliumNetworkPolicy and L7 policy
**Production:** Default-deny rollouts, egress policy for FQDNs, and what the standard API cannot express.  
**Core:** NetworkPolicy is just an API object; enforcement is entirely the CNI's job.  
**Read:** Cilium docs *Network Policy* (L7, FQDN, DNS proxy); AdminNetworkPolicy in `kubernetes-sigs/network-policy-api`.  
**Do:** Default-deny a namespace; write the minimal egress for a Flux-managed app including DNS.  
**Questions:**
1. Why does standard NetworkPolicy have no deny rules, and what does AdminNetworkPolicy add?  
2. How does Cilium enforce FQDN policies, and what is the DNS proxy's role?  
3. What happens to an existing connection when a blocking policy is applied?  

### Week 26 — Ingress → Gateway API and cloud load balancers
**Production:** Migrating off Ingress, AWS LBC IP vs instance targets, readiness gates, LB health checks vs pod termination.  
**Core:** Controllers watching Gateway/HTTPRoute; cloud controller manager for `type: LoadBalancer`.  
**Read:** gateway-api.sigs.k8s.io *Concepts*; AWS Load Balancer Controller docs (target types, readiness gates).  
**Do:** Migrate one Ingress to an HTTPRoute in sandbox; enable pod readiness gates and watch registration timing.  
**Questions:**
1. How does Gateway API's role separation map to platform vs app teams?  
2. What does a readiness gate for target-group registration prevent?  
3. Why do NLBs with IP targets interact badly with pod termination without a preStop sleep?  

### Week 27 — (Buffer) Service mesh data planes
**Production:** Sidecar vs ambient/sidecarless (Istio ambient, Cilium service mesh), mTLS cost, mesh interception vs CNI features.  
**Core:** Sidecars rely on init containers rewriting iptables; sidecarless designs hook the CNI or eBPF.  
**Read:** Istio docs *Ambient architecture*; KEP-753 (sidecar containers).  
**Do (optional):** Ambient mode in kind.  
**Questions:**
1. How do native sidecar containers fix mesh startup ordering?  
2. What does ztunnel do per node and how does it identify workloads?  
3. Where does Cilium mutual auth differ from mTLS?  

### Week 28 — Multi-cluster, cross-account and dual-stack networking
**Production:** Hub-and-spoke designs: Transit Gateway vs VPC peering reachability for pod IPs, cluster mesh, IPv6 as the escape from IP exhaustion.  
**Core:** Cross-cluster networking is outside Kubernetes' model; cluster mesh extends Service/identity across API servers.  
**Read:** Cilium docs *Cluster Mesh*; docs *IPv4/IPv6 dual-stack*; EKS best-practices guide, networking section on IPv6.  
**Do:** Dual-stack kind cluster; find one tool in your stack that breaks.  
**Questions:**
1. How does Cilium Cluster Mesh share identities without a shared etcd?  
2. What breaks in dual-stack when a controller assumes a single pod IP?  
3. How do you route pod CIDRs across accounts with overlay vs native routing?  

---

## Module 5 — Storage (Weeks 29–32)

### Week 29 — CSI architecture
**Production:** Volumes stuck `Attaching`, orphaned `VolumeAttachment` objects after node loss, and the sidecars that do the actual work.  
**Core:** The attach/detach controller creates `VolumeAttachment`; the kubelet's volume manager calls the node plugin to stage and publish.  
**Read:** CSI spec (`github.com/container-storage-interface/spec`); kubernetes-csi docs *Sidecar containers*; source `pkg/controller/volume/attachdetach/`.  
**Do:** Watch `VolumeAttachment` while an EBS-backed pod moves nodes; delete a node and time the force-detach.  
**Questions:**
1. What are the five CSI calls between PVC creation and a writable file in a pod, and which component issues each?  
2. Why does a volume stay attached after a node disappears, and how does force-detach timeout work?  
3. What is the external-attacher's role vs the kubelet's on EKS with EBS?  

### Week 30 — PV/PVC binding and topology-aware provisioning
**Production:** Pods pending because the volume is in another AZ; why `WaitForFirstConsumer` exists and how the scheduler participates.  
**Core:** The PV controller binds; the scheduler's `VolumeBinding` plugin delays provisioning until node selection.  
**Read:** docs *Storage Classes* (volume binding mode); source `pkg/scheduler/framework/plugins/volumebinding/`.  
**Do:** Provision with `Immediate` in a 3-AZ sandbox; watch Karpenter fail to place.  
**Questions:**
1. How does delayed binding pass topology from scheduler to provisioner?  
2. What does the `Immediate` failure mode look like under Karpenter consolidation?  
3. How is a PV's `nodeAffinity` enforced after binding?  

### Week 31 — Snapshots, expansion, cloning and ephemeral volumes
**Production:** Backups via `VolumeSnapshot`, online resize without restart, generic ephemeral volumes as a better `emptyDir`.  
**Core:** External controllers (snapshotter, resizer) plus kubelet node-side expansion.  
**Read:** docs *Volume Snapshots*, *Resizing PVCs*, *Ephemeral Volumes*; `kubernetes-csi/external-resizer` README.  
**Do:** Online-expand an EBS volume; inspect both phases in PVC conditions.  
**Questions:**
1. Why does expansion have two phases, and what happens if the node-side phase fails?  
2. What does a `VolumeSnapshotContent` deletion policy mean for the cloud snapshot?  
3. What is the volume populator pattern?  

### Week 32 — StatefulSets and running data on Kubernetes
**Production:** Ordinal guarantees, PVC retention, at-most-one semantics after node loss, and when an operator (CloudNativePG, Strimzi) beats a StatefulSet.  
**Core:** StatefulSet controller in controller-manager; identity depends on a headless Service plus PVC naming.  
**Read:** docs *StatefulSets* (PVC retention, KEP-1847); KEP-2268 (non-graceful node shutdown); CloudNativePG docs *Architecture* as contrast.  
**Do:** Hard-stop a node hosting a StatefulSet pod; then apply the `out-of-service` taint.  
**Questions:**
1. Why will a StatefulSet never recreate a pod on an unreachable node without intervention, and what does non-graceful shutdown change?  
2. What does `persistentVolumeClaimRetentionPolicy` do on scale-down vs delete?  
3. What is `maxUnavailable` for StatefulSets and why was it gated so long?  

---

## Module 6 — Security and access control (Weeks 33–38)

### Week 33 — Authentication: OIDC, service account tokens, IRSA and Pod Identity
**Production:** Bound/projected tokens with expiry, the `TokenRequest` API, and how a pod exchanges a Kubernetes token for AWS credentials.  
**Core:** API server authenticators run in a chain; the token controller and `TokenRequest` subresource issue tokens.  
**Read:** docs *Authenticating* (structured auth config, KEP-3221), service account token docs (KEP-1205); AWS docs on IRSA vs EKS Pod Identity.  
**Do:** Decode a projected token's JWT; call `TokenRequest` manually; exchange it with STS by hand.  
**Questions:**
1. What does the `aud` claim do, and why does IRSA require a specific audience?  
2. How does the API server validate OIDC tokens without a network call?  
3. How do the trust paths of IRSA and Pod Identity differ?  

### Week 34 — RBAC deep dive
**Production:** Escalation via `bind`/`escalate`/`impersonate`, aggregated ClusterRoles, auditing who can `exec` into pods.  
**Core:** RBAC authorizer in the API server; `SubjectAccessReview` is the query interface.  
**Read:** docs *Using RBAC Authorization* (escalation prevention); source `plugin/pkg/auth/authorizer/rbac/`; tools `rbac-tool`, `kubectl-who-can`.  
**Do:** Audit every subject able to `get secrets` cluster-wide, including via aggregation.  
**Questions:**
1. How does RBAC prevent granting permissions you do not have, and which verbs bypass that?  
2. How would you list every subject that can read Secrets, including through aggregation?  
3. Why does `kubectl auth can-i --list` lie in the presence of webhooks?  

### Week 35 — Admission: webhooks, Kyverno and CEL policies
**Production:** Webhook failure policies taking down the cluster, latency budgets, and moving rules to `ValidatingAdmissionPolicy` to leave the critical path.  
**Core:** Admission runs in the API server; webhooks are out-of-process, CEL policies in-process.  
**Read:** docs *Validating Admission Policy* (KEP-3488), *Mutating Admission Policy* (KEP-3962); Kyverno docs *Policy types*; CEL language basics.  
**Do:** Port two Kyverno validate rules to VAP; measure webhook latency reduction.  
**Questions:**
1. What can VAP not express that a Kyverno webhook can?  
2. What does `failurePolicy: Fail` with a `kube-system` exclusion actually protect?  
3. How does `MutatingAdmissionPolicy` order its mutations relative to webhooks?  

### Week 36 — Secrets management
**Production:** Encryption at rest with KMS v2, key rotation and re-encryption, External Secrets Operator vs Secrets Store CSI, and what "base64 is not encryption" really implies.  
**Core:** Encryption happens in the API server's storage layer; Secrets are mounted by the kubelet on tmpfs.  
**Read:** docs *Encrypting Confidential Data at Rest*; KEP-3299 (KMS v2); External Secrets Operator docs; Secrets Store CSI driver docs.  
**Do:** Read a Secret directly from etcd before and after enabling envelope encryption in kind.  
**Questions:**
1. What changed between KMS v1 and v2, and why was v1 a latency and availability problem?  
2. How do you prove every Secret is encrypted after enabling encryption on an existing cluster?  
3. What does a Secret look like in etcd with and without envelope encryption?  

### Week 37 — Supply chain security
**Production:** Sigstore/cosign signing, SBOM attestations, signature verification at admission, digest pinning.  
**Core:** Admission policy plus registry; the kubelet pulls what it is told, so enforcement must happen earlier.  
**Read:** docs.sigstore.dev (keyless flow); Kyverno docs *Verify images*; Flux docs *Image automation*.  
**Do:** Sign an image keyless with cosign; enforce with Kyverno; confirm digest mutation works with Flux.  
**Questions:**
1. How does keyless signing work, and what does the Fulcio/Rekor chain prove?  
2. Why is a tag-based policy unsafe even with signature verification?  
3. How does digest mutation interact with Flux image automation?  

### Week 38 — Audit, runtime security and the cluster-compromise threat model
**Production:** Audit stages and levels, shipping audit logs, Tetragon/Falco for runtime detection, walking the attacker path from pod to cluster-admin.  
**Core:** Audit is an API server filter; runtime detection is node-level eBPF.  
**Read:** docs *Auditing*; *Hacking Kubernetes* (pod-to-cluster escalation chapter); Tetragon docs *Tracing policies*.  
**Do:** Write the escalation path from a pod with `create pods` in `kube-system`; detect it with Tetragon.  
**Questions:**
1. What does `RequestResponse` level cost in log volume, and how would you scope it?  
2. Given a service account that can `create pods` in `kube-system`, write the escalation path.  
3. How does Tetragon enforce (not just observe) a policy?  

---

## Module 7 — Controllers and operator development (Weeks 39–45)

### Week 39 — Reconciliation theory
**Production:** Controllers that work in demos and break under event loss, partial failures or restarts. Level-triggered design is why Kubernetes survives missed events.  
**Core:** Every core controller does this: observe desired + actual, diff, act, requeue.  
**Read:** *Programming Kubernetes* ch. 1–2; `sig-api-machinery/controllers.md`; API conventions on conditions and `observedGeneration`.  
**Do:** Write a reconciler on paper for "ensure an S3 bucket exists"; list every failure mode.  
**Questions:**
1. What is the difference between edge- and level-triggered reconciliation, and why does Kubernetes choose the latter?  
2. What is `observedGeneration` protecting against?  
3. Why should a reconciler never trust that its last action succeeded?  

### Week 40 — controller-runtime: managers, caches, predicates, owner references
**Production:** Memory bloat from caching every Secret, missed reconciles from aggressive predicates, cascading deletion.  
**Core:** Owner references are consumed by the garbage collector in controller-manager, not your controller.  
**Read:** kubebuilder book *Architecture Concept Diagram*, *Watching Resources*; controller-runtime `pkg/builder`, `pkg/cache` godoc; source `pkg/controller/garbagecollector/`.  
**Do:** Build the kubebuilder book's CronJob example; add a cache label selector and observe what breaks.  
**Questions:**
1. How does `Owns()` differ from `Watches()` with `EnqueueRequestForOwner`?  
2. What does GC do with an owner reference to a resource in another namespace?  
3. How do you restrict the cache to labeled objects, and what breaks?  

### Week 41 — Building an operator: CRD design and versioning
**Production:** Spec/status separation, condition conventions, defaulting, shipping `v1beta1` → `v1` without downtime.  
**Core:** Kubebuilder markers generate OpenAPI schemas the API server enforces; webhooks run inside your operator.  
**Read:** kubebuilder book *Multi-Version API*, *Webhooks*; *Kubernetes Operators* (maturity model).  
**Do:** Add `v2` to your CronJob operator with conversion.  
**Questions:**
1. Why should `status` be a subresource, and what does that change about RBAC and updates?  
2. What must a conversion webhook guarantee about round-tripping?  
3. How do you design conditions so that `kubectl wait` works?  

### Week 42 — Server-side apply, field managers and concurrency
**Production:** Flux, Kyverno mutations and HPA fighting over the same field. SSA field ownership resolves the fight; conflict errors are a feature.  
**Core:** The API server tracks `managedFields`; client-side apply uses the last-applied annotation.  
**Read:** docs *Server-Side Apply*; source `staging/src/k8s.io/apimachinery/pkg/util/managedfields/`; Flux docs *Kustomization → drift detection*; source `pkg/controller/replicaset/` (expectations).  
**Do:** Create a conflict between Flux and a manual `kubectl apply --server-side`; inspect `managedFields`.  
**Questions:**
1. What happens when two managers apply different values with `force: false`?  
2. How does Flux handle a field it previously managed being removed from Git?  
3. What does the ReplicaSet controller's expectations pattern solve that SSA does not?  

### Week 43 — Testing operators
**Production:** Unit vs `envtest` (real API server, no kubelet) vs e2e on kind; chaos tests that kill the operator mid-reconcile.  
**Core:** envtest runs a real API server and etcd, so admission and schema validation behave like production.  
**Read:** kubebuilder book *Writing tests* (envtest); `kubernetes-sigs/e2e-framework` README.  
**Do:** envtest for a finalizer whose external resource is already gone; `kill -9` the operator mid-reconcile.  
**Questions:**
1. What does envtest not simulate, and which bugs only show up with a real kubelet?  
2. How do you test a finalizer when the external resource is already gone?  
3. How do you make a reconcile deterministic enough to assert on?  

### Week 44 — Workload controller internals
**Production:** Deployment rollout math (`maxSurge`/`maxUnavailable` rounding), ReplicaSet hash collisions, DaemonSet update strategies, Job backoff and Indexed completion, JobSet.  
**Core:** All in controller-manager; the Deployment controller never touches pods, only ReplicaSets.  
**Read:** source `pkg/controller/deployment/rolling.go`, `pkg/controller/job/`; KEP-3329 (podFailurePolicy); `kubernetes-sigs/jobset` docs.  
**Do:** Step through a rollout with `maxSurge: 25%` on 3 replicas; predict every ReplicaSet scale before it happens.  
**Questions:**
1. How does the Deployment controller pick which old ReplicaSet to scale down first?  
2. What is the pod-template-hash and when does a collision occur?  
3. How does `podFailurePolicy` distinguish a preemption from an application crash?  

### Week 45 — GitOps engines as controllers: Flux internals
**Production:** Source/kustomize/helm controllers as separate reconcilers, `dependsOn`, health checks, drift detection, and who wins when a human `kubectl edit`s.  
**Core:** Flux is controllers with CRDs; it applies via SSA and detects drift by diffing managed fields.  
**Read:** Flux docs *Kustomize controller → health checks, dependsOn, pruning*, *Helm controller → remediation*; source `fluxcd/pkg/ssa`.  
**Do:** Make a HelmRelease fail at an upgrade hook; observe remediation decisions in status.  
**Questions:**
1. How does kustomize-controller decide a `Kustomization` is healthy, and what does it check for a CRD or a Job?  
2. When a HelmRelease upgrade fails midway, how does remediation choose between rollback and uninstall?  
3. How does Flux prune, and what prevents it from deleting something it did not create?  

---

## Module 8 — Operating at scale (Weeks 46–52)

### Week 46 — Observability internals
**Production:** Metric cardinality from pod labels, control-plane metrics worth alerting on (API latency SLIs, etcd fsync, scheduler queue depth), kube-state-metrics as a controller, Datadog cluster agent vs node agent.  
**Core:** Every component exposes `/metrics`; container metrics originate at the kubelet's cAdvisor endpoint.  
**Read:** docs *Kubernetes Metrics Reference* (focus on `apiserver_request_duration_seconds`, `etcd_request_duration_seconds`, `scheduler_pending_pods`); kube-state-metrics docs on sharding; Datadog docs *Cluster Agent*.  
**Do:** Build one dashboard that separates etcd slowness from admission slowness.  
**Questions:**
1. Which API server metrics distinguish etcd slowness from admission slowness?  
2. Why does kube-state-metrics need sharding on large clusters?  
3. What is the cost of `kubectl top` on a 5000-pod cluster?  

### Week 47 — Upgrades and version skew
**Production:** Skew policy between API server, kubelet and clients; API removals breaking Helm charts; node rotation with Karpenter drift.  
**Core:** Control plane first, always; the kubelet may lag up to three minors.  
**Read:** kubernetes.io/releases/version-skew-policy; *Deprecated API Migration Guide*; Karpenter docs *Drift*; EKS docs on control-plane upgrades. Tools: `pluto`, `kubent`.  
**Do:** Run `pluto` against your Flux repos; throttle Karpenter drift with a NodePool budget.  
**Questions:**
1. What happens to a controller using a removed API version after a control-plane upgrade, and how do you find those first?  
2. How does Karpenter drift trigger node replacement, and how do you throttle it?  
3. What does an EKS control-plane upgrade do to in-flight watches?  

### Week 48 — Multi-tenancy models
**Production:** Namespaces-as-tenants vs vcluster/virtual control planes vs cluster-per-tenant; the isolation each actually provides.  
**Core:** Namespaces exist only at the API server; nothing at node or network level isolates them.  
**Read:** docs *Multi-tenancy*; vcluster docs *Architecture*; `kubernetes-sigs/hierarchical-namespaces` README.  
**Do:** Stand up a vcluster in kind; list what it shares with the host.  
**Questions:**
1. What does a vcluster share with its host and what does it not?  
2. Which Kyverno policies make one namespace tenant unable to affect another?  
3. How does hierarchical namespace propagation work?  

### Week 49 — (Buffer) Cluster API and fleet management
**Production:** Managing many clusters declaratively: the management-cluster model, bootstrap/control-plane/infrastructure providers, vs EKS via Terraform.  
**Core:** Operators reconciling clusters as CRDs — Kubernetes managing Kubernetes.  
**Read:** cluster-api.sigs.k8s.io *Concepts*; CAPA docs on EKS.  
**Do (optional):** `clusterctl` in kind with the Docker provider.  
**Questions:**
1. How does a `MachineDeployment` rolling update differ from a Deployment's?  
2. What does the EKS infrastructure provider actually create?  
3. How do you move a cluster between management clusters?  

### Week 50 — Disaster recovery
**Production:** etcd snapshot/restore semantics (restore changes cluster identity), Velero for resources + volumes, "rebuild from Git" and its gaps (PVs, Secrets, cloud resources).  
**Core:** etcd is the state; everything else can be regenerated from it — except external state.  
**Read:** etcd docs *Disaster recovery*; Velero docs *Restore reference* (ordering, hooks); Flux docs *Bootstrap*.  
**Do:** Restore an old etcd snapshot in kind and watch kubelets fight the API; then rebuild a sandbox cluster from Git alone and list what was missing.  
**Questions:**
1. After an etcd restore to an older snapshot, what happens to kubelets holding pods that no longer exist?  
2. What does Velero's restore order look like, and why?  
3. What state in your cluster is in neither Git nor etcd?  

### Week 51 — Performance and scale limits
**Production:** Scalability envelopes (nodes, pods, Services, endpoints per Service), load testing, API server tuning, which controllers fall over first.  
**Core:** Scalability SLIs per component: API latency, pod startup latency, in-cluster DNS latency.  
**Read:** `kubernetes/community` → `sig-scalability/configs-and-limits/thresholds.md`, `sig-scalability/slos/`; kube-burner docs; kwok README.  
**Do:** 500 fake nodes with kwok; run a kube-burner cluster-density test; watch APF and watch-cache metrics.  
**Questions:**
1. Why do Services with thousands of endpoints hurt every node, and how do EndpointSlices mitigate it?  
2. Which single config change most reduces API server memory on a large cluster?  
3. What is the pod startup latency SLO, and where does the time go?  

### Week 52 — Capstone: trace, break, document
**Task:** Trace a `kubectl apply` of a Deployment with a PVC and a Service end to end through every component in this workbook, with real timings. Then design three failure injections — etcd latency, webhook timeout, AZ loss — and predict behavior before running them.  
**Materials:** Everything above plus your 51 notes.  
**Questions:**
1. Where were your predictions wrong?  
2. Which week's topic explained it?  
3. Which component had the least observability for what you needed?  
