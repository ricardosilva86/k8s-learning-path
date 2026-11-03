Here's a copy-paste setup. It assumes Linux or macOS with Homebrew and Docker (or Podman) running. Check the cgroup line first — everything else is forgiving.

**1. Host check (cgroup v2 is required for weeks 8–9)**

```bash
docker info --format '{{.CgroupVersion}}'   # must print 2
# Docker Desktop: Settings → General → "Use cgroup v2" if it prints 1
```

**2. Tools**

```bash
brew install kind kubectl helm kustomize etcd cilium-cli hubble \
             kubebuilder cosign kwok stern k9s

# Not in Homebrew (or stale there) — grab the release binaries:
#   kube-burner        github.com/kube-burner/kube-burner/releases
#   crictl             github.com/kubernetes-sigs/cri-tools/releases
#   clusterctl         github.com/kubernetes-sigs/cluster-api/releases  (week 49 only)
#   pluto / kubent     brew install FairwindsOps/tap/pluto ; brew install kubent
#   rbac-tool          brew install alcideio/tap/rbac-tool
#   kubectl-who-can    kubectl krew install who-can
```

**3. kind cluster config — `lab-kind.yaml`**

Three nodes, no default CNI (Cilium goes in instead, so the lab matches what you run at work), kube-proxy kept for week 23, and a few feature gates you'll need later.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lab
networking:
  disableDefaultCNI: true
  kubeProxyMode: iptables        # switch to "nftables" in week 23
  ipFamily: ipv4                 # set to "dual" in week 28
featureGates:
  InPlacePodVerticalScaling: true   # week 9
  UserNamespacesSupport: true       # week 17
  DynamicResourceAllocation: true   # week 10
runtimeConfig:
  resource.k8s.io/v1beta1: true     # DRA API group; adjust to the version your kind node image serves
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        audit-log-path: /var/log/kubernetes/audit.log
        audit-policy-file: /etc/kubernetes/audit-policy.yaml
      extraVolumes:
        - name: audit
          hostPath: /etc/kubernetes/audit-policy.yaml
          mountPath: /etc/kubernetes/audit-policy.yaml
          readOnly: true
          pathType: File
        - name: audit-log
          hostPath: /var/log/kubernetes
          mountPath: /var/log/kubernetes
          pathType: DirectoryOrCreate
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: ./audit-policy.yaml
        containerPath: /etc/kubernetes/audit-policy.yaml
  - role: worker
  - role: worker
  - role: worker
```

Minimal `audit-policy.yaml` next to it (week 1 and 38 will expand it):

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
```

**4. Create the cluster and install the base stack**

```bash
kind create cluster --config lab-kind.yaml

# Cilium with kube-proxy left in place (so week 23 is possible). Flip
# kubeProxyReplacement to true after week 23 to study the replacement path.
cilium install \
  --set kubeProxyReplacement=false \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
cilium status --wait

# metrics-server (week 13); kind nodes use self-signed kubelet certs
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server \
  -n kube-system --set args="{--kubelet-insecure-tls}"

# Kyverno (weeks 17, 35, 37)
helm repo add kyverno https://kyverno.github.io/kyverno/
helm upgrade --install kyverno kyverno/kyverno -n kyverno --create-namespace

# Flux (weeks 42, 45) — bootstrap against a throwaway repo later; for now:
brew install fluxcd/tap/flux
flux install

kubectl get nodes -o wide && kubectl get pods -A
```

**5. Getting at the internals (you'll use these every week)**

```bash
# shell on a node
docker exec -it lab-worker bash
#   inside: crictl pods ; ctr -n k8s.io images ls ; cat /sys/fs/cgroup/kubepods.slice/...

# etcd on the control plane
docker exec -it lab-control-plane sh -c '
  ETCDCTL_API=3 etcdctl \
    --endpoints https://127.0.0.1:2379 \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --cert   /etc/kubernetes/pki/etcd/server.crt \
    --key    /etc/kubernetes/pki/etcd/server.key \
    get / --prefix --keys-only | head'

# audit log
docker exec lab-control-plane tail -f /var/log/kubernetes/audit.log | jq .

# component metrics (week 46)
kubectl get --raw /metrics | grep apiserver_request_duration_seconds | head
kubectl get --raw /metrics | grep apiserver_flowcontrol | head
```

**6. Tear down / reset**

```bash
kind delete cluster --name lab      # recreate takes ~2 min; don't be precious about it
```

**7. Sandbox EKS (only for the cloud-specific weeks: 12, 26, 29–31, 33)**

You already have Terraform modules for this, so reuse the smallest variant you own. If you want a throwaway that is independent of work code:

```bash
brew install eksctl
eksctl create cluster --name k8s-lab --region eu-central-1 \
  --version <current> --nodes 2 --node-type t3.medium \
  --with-oidc --managed
# then: Karpenter via its Helm chart, AWS LBC via Helm, EBS CSI as an EKS add-on
eksctl delete cluster --name k8s-lab --region eu-central-1   # the moment you're done
```

Two notes: kind's node image Kubernetes version determines which feature gates and API versions exist, so run `kind version` and `kubectl version` once and keep the DRA `runtimeConfig` line in step with what that version serves. And keep the EKS cluster strictly ephemeral — create it the evening you need it, delete it the same night.
