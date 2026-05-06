# homelab-cilium

The Cilium configuration for my homelab Kubernetes cluster (Talos Linux, kube-proxy replacement active).

## Requirements

- Talos Linux with `localhost:7445` as the local Kubernetes API proxy
- `kubeProxyReplacement=true` — **no kube-proxy running**

## Generating the manifest

```sh
task generate-cilium-manifest
```

This runs `helm template` and writes the result to `cilium.yaml`.
The variable `UPGRADE_COMPAT` in `taskfile.yaml` must always match the **currently running** Cilium minor version
(e.g. `1.19` when you are on any `1.19.x` release).

## Upgrading Cilium (kube-proxy replacement — safe procedure)

Because Cilium replaces kube-proxy there is **no iptables fallback** during upgrades.

The root cause of cluster failures during upgrades is the default `maxUnavailable: 2` in the
Cilium DaemonSet rolling update strategy. With kube-proxy replacement active, **two nodes
simultaneously losing their Cilium agent means two nodes simultaneously have no service routing**.
In a small cluster this takes down enough of the dataplane to make the API server unreachable.

The fix is already embedded in the `helm template` command via:

```
--set updateStrategy.rollingUpdate.maxUnavailable=1
--set envoy.updateStrategy.rollingUpdate.maxUnavailable=1
```

This ensures only **one node at a time** goes through the Cilium restart.

### Upgrade procedure

### Step 1 — Update `CILIUM_VERSION` in `taskfile.yaml`

### Step 2 — Re-generate the manifest

```sh
task generate-cilium-manifest
```

### Step 3 — (Optional but recommended) Run the preflight check

Pre-pulls the new image on every node and validates all `CiliumNetworkPolicy` resources.

```sh
task preflight-check CILIUM_VERSION=<new-version>
task preflight-cleanup
```

### Step 4 — Apply the new manifest

```sh
kubectl apply -f cilium.yaml
kubectl rollout status daemonset/cilium -n kube-system --timeout=600s
```

---

## Why the cluster stops working without `maxUnavailable=1`

| `maxUnavailable: 2` (default)                        | `maxUnavailable: 1` (this repo)             |
| ---------------------------------------------------- | ------------------------------------------- |
| 2 nodes restart Cilium simultaneously                | Only 1 node restarts Cilium at a time       |
| Both nodes lose BPF service routing at once          | Remaining nodes keep routing traffic        |
| With kube-proxy replacement: **cluster unreachable** | Brief per-node disruption, cluster stays up |

> **Note:** `upgradeCompatibility` (a Helm flag sometimes mentioned in docs) has **zero effect**
> when using `helm template` + `kubectl apply`. It only works with `helm upgrade`.

## Renovate

Renovate automatically bumps `CILIUM_VERSION` in `taskfile.yaml`.
After merging a Renovate PR you still need to manually:

1. Verify `UPGRADE_COMPAT` is still correct (it should be for patch upgrades within the same minor)
2. Run `task generate-cilium-manifest` and commit the updated `cilium.yaml`
3. Follow the upgrade procedure above
