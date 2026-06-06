# Runbook: DNS Resolution Failure

## Symptoms

- Init containers (e.g., `git-init-0`) fail with DNS resolution errors
- Error messages contain: `Could not resolve host`, `Name or service not known`, `SERVFAIL`
- Pods stuck in `Init:Error` or `Init:CrashLoopBackOff`
- External network access works from the host but not from pods

## Diagnosis Steps

### Step 1: Test DNS from Inside a Pod

```bash
# Run a debug pod
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- /bin/sh

# Inside the pod:
nslookup github.com
nslookup host.k3s.internal
nslookup kubernetes.default.svc
```

**Expected**: All should resolve to valid IPs.
**If `host.k3s.internal` resolves to a wrong IP**: See Resolution B.

### Step 2: Check CoreDNS ConfigMap

```bash
kubectl get configmap -n kube-system coredns-custom -o yaml
```

Look for hardcoded IPs that may be stale.

### Step 3: Check Node IP

```bash
# On the host (not in a pod)
ip addr show
# or
hostname -I
```

Compare with the IP in `coredns-custom` ConfigMap.

### Step 4: Check CoreDNS Logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

Look for:
- `SERVFAIL` responses
- Forward errors to upstream DNS
- Plugin errors

## Resolution

### Scenario A: General DNS Failure

If all DNS queries fail:

```bash
# Restart CoreDNS
kubectl rollout restart deployment -n kube-system coredns

# Wait for it to be ready
kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s

# Test again
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup github.com
```

### Scenario B: Stale host.k3s.internal Record (Known Issue)

If `host.k3s.internal` resolves to an old IP:

```bash
# 1. Get current host IP
HOST_IP=$(hostname -I | awk '{print $1}')
echo "Current host IP: $HOST_IP"

# 2. Update CoreDNS ConfigMap
kubectl patch configmap -n kube-system coredns-custom --type merge -p "{\"
data\":{\"host.k3s.internal.server\":\"host.k3s.internal {\\n  hosts {\\n    $HOST_IP host.k3s.internal\\n    fallthrough\\n  }\\n}\\n\"}}"

# 3. Restart CoreDNS
kubectl rollout restart deployment -n kube-system coredns

# 4. Delete stuck pods so they retry
kubectl delete pods -n kubeopencode-agent --all
```

### Scenario C: Upstream DNS Unreachable

If CoreDNS can resolve cluster-internal names but not external:

```bash
# Check CoreDNS ConfigMap for upstream DNS
kubectl get configmap -n kube-system coredns -o yaml

# Typical upstream is /etc/resolv.conf on the node
# Ensure the node can resolve external names:
nslookup github.com

# If node's DNS is broken, fix node's /etc/resolv.conf
# Then restart CoreDNS
```

## Verification

```bash
# Test from a new pod
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- /bin/sh
nslookup github.com
nslookup host.k3s.internal
wget -qO- https://github.com -O /dev/null && echo "External HTTPS: OK"

# Check agent pods are starting
kubectl get pods -n kubeopencode-agent -w
```

## Prevention

1. **Use static IP for K3s node** or configure DHCP reservation on router
2. **Monitor CoreDNS** with Prometheus alert `CoreDNSSERVFAIL`
3. **Document host IP** in cluster docs and update procedure
4. **Consider host network mode** for critical init containers if DNS is unreliable
