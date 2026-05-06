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
A wrong upgrade procedure causes the BPF programs to be torn down without being rebuilt, making the
cluster unreachable (deadlock: Cilium needs the API server → API server needs Cilium).

### Step 1 — Update the version variables in `taskfile.yaml`

```yaml
vars:
  CILIUM_VERSION: 1.19.9 # ← new target version
  UPGRADE_COMPAT: "1.19" # ← current running MINOR version (do NOT change for patch upgrades)
```

> When upgrading across minor versions (e.g. 1.18 → 1.19), set `UPGRADE_COMPAT` to the **old** minor
> version (`"1.18"`) and read the [Cilium upgrade notes](https://docs.cilium.io/en/stable/operations/upgrade/)
> for that release **before** applying anything.

### Step 2 — Re-generate the manifest

```sh
task generate-cilium-manifest
```

This embeds `--set upgradeCompatibility=<UPGRADE_COMPAT>` into the ConfigMap, which tells Cilium to
**preserve the existing BPF map layout** during the rolling update instead of rebuilding it from scratch.
Without this flag the datapath is briefly torn down on each node, causing a full connectivity loss with
no iptables fallback.

### Step 3 — Run the preflight check

```sh
task preflight-check CILIUM_VERSION=1.19.9
```

This pre-pulls the new image on every node and validates all `CiliumNetworkPolicy` resources.
Wait until the output shows `1/1` for the preflight Deployment, then clean up:

```sh
task preflight-cleanup
```

### Step 4 — Apply the new manifest

```sh
kubectl apply -f cilium.yaml
kubectl rollout status daemonset/cilium -n kube-system --timeout=600s
```

### Step 5 — Update `UPGRADE_COMPAT` after a minor-version upgrade

Only relevant when you crossed a minor boundary (e.g. 1.18 → 1.19).
After the cluster is stable, update `UPGRADE_COMPAT` in `taskfile.yaml` to the new minor version and
re-generate + re-apply the manifest so the `upgradeCompatibility` annotation in the ConfigMap reflects
the now-current version.

---

## Why the cluster stops working without `upgradeCompatibility`

| Without `upgradeCompatibility`                    | With `upgradeCompatibility=<current>`         |
| ------------------------------------------------- | --------------------------------------------- |
| Cilium detects BPF map version mismatch           | Cilium keeps existing BPF maps                |
| Tears down all BPF programs to rebuild            | Only reloads changed programs                 |
| ~10–60 s window with no service routing           | Disruption limited to individual pod restarts |
| With kube-proxy replacement: **node unreachable** | Node stays reachable during rolling update    |

## Renovate

Renovate automatically bumps `CILIUM_VERSION` in `taskfile.yaml`.
After merging a Renovate PR you still need to manually:

1. Verify `UPGRADE_COMPAT` is still correct (it should be for patch upgrades within the same minor)
2. Run `task generate-cilium-manifest` and commit the updated `cilium.yaml`
3. Follow the upgrade procedure above
