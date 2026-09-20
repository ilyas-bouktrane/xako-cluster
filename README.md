# 🏗️ Production Distributed Kubernetes Datacenter

<div align="center">

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Talos Linux](https://img.shields.io/badge/Talos_Linux-FF7300?style=for-the-badge&logo=linux&logoColor=white)](https://www.talos.dev/)
[![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-E57000?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com/)
[![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![Cilium](https://img.shields.io/badge/Cilium-F8C517?style=for-the-badge&logo=cilium&logoColor=black)](https://cilium.io/)
[![BGP](https://img.shields.io/badge/BGP_%2F_ECMP-CC0000?style=for-the-badge&logoColor=white)](https://frrouting.org/)
[![UniFi](https://img.shields.io/badge/UniFi-0559C9?style=for-the-badge&logo=ubiquiti&logoColor=white)](https://ui.com/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://www.cloudflare.com/)
[![Argo CD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![Longhorn](https://img.shields.io/badge/Longhorn-5F259F?style=for-the-badge&logoColor=white)](https://longhorn.io/)
[![HashiCorp Vault](https://img.shields.io/badge/HashiCorp_Vault-FFEC6E?style=for-the-badge&logo=vault&logoColor=000000)](https://www.vaultproject.io/)

</div>

A production-grade, distributed Kubernetes datacenter architected from scratch — every component sourced individually — and deployed in a real 15U rack. Three Dell PowerEdge R730 hosts run Proxmox VE with Talos Linux in a redundant **3+3 topology**: one control-plane VM and one worker VM per physical host.

**This repository is the single source of truth.** A `git push` to `main` is the only deployment mechanism — there is no manual `kubectl apply` anywhere in the cluster lifecycle.

🌐 **Live services:** [gallery.xako.net](https://gallery.xako.net) · [drive.xako.net](https://drive.xako.net) · [music.xako.net](https://music.xako.net) · [vlsm.xako.net](https://vlsm.xako.net)

---

## 📸 The Rack

![15U rack — front view with the three R730 hosts, UDM Pro and switch](screenshots/rack-full.png)

---

## 🚀 About The Project

Every personal service I use — photo backup, file sync, music streaming, custom applications — runs on hardware I own, with **zero dependency on third-party providers**. That constraint drove every architectural decision in this repository.

The goal was not to make self-hosting work. It was to make it work the way production infrastructure works: declarative, reproducible, observable, and recoverable without a human typing commands into a terminal.

Four decisions define the build:

- 🧩 **Talos Linux** — an immutable, API-driven OS with no shell and no SSH. The attack surface of a node is its API, and nothing else.
- 🛰️ **BGP instead of L2 announcements** — Cilium peers with the UniFi gateway so LoadBalancer VIPs are genuinely routed with ECMP, not owned by a single elected node.
- 🔀 **Split-horizon DNS with dual certificate authorities** — one hostname, two paths, publicly trusted TLS on both.
- 🔐 **No secrets in Git, ever** — Vault HA + External Secrets Operator, verifiable across the entire commit history.

---

## 🖥️ Hardware & Edge

| Layer                   | Equipment                                                           |
| ----------------------- | ------------------------------------------------------------------- |
| **Rack**                | 15U, self-assembled                                                 |
| **Compute**             | 3 × Dell PowerEdge R730 running **Proxmox VE** in a quorate cluster |
| **Aggregate resources** | 144 vCPU · 94 GiB RAM · 3 TiB storage                               |
| **Kubernetes nodes**    | 6 × **Talos Linux** VMs — 1 control plane + 1 worker per host       |
| **Gateway**             | **UniFi UDM Pro** — zone-based firewall, IPS/IDS, FRRouting         |
| **Switching**           | **UniFi USW** — VLAN segmentation, LACP link aggregation, PoE       |
| **Edge**                | **Cloudflare** — WAF, DDoS protection, proxied DNS, Zero Trust      |
| **Remote access**       | OpenVPN and WireGuard, each on its own routed subnet                |

### Network segmentation

Seven segments, separated by function rather than convenience, with zone-based firewall policies between them:

| Segment                             | Purpose                                                                 |
| ----------------------------------- | ----------------------------------------------------------------------- |
| `192.168.10.0/24`                   | **Management** — Proxmox hypervisors, switch and gateway administration |
| `192.168.20.0/24`                   | **Deployment** — Kubernetes nodes and advertised service VIPs           |
| `192.168.30.0/24`                   | **Security** — IP surveillance                                          |
| `192.168.40.0/24`                   | **Home** — personal devices                                             |
| `192.168.50.0/24`                   | **IoT** — untrusted appliances, isolated from everything else           |
| `172.16.10.0/24` · `172.16.20.0/24` | **Remote access** — OpenVPN and WireGuard                               |

The hypervisor management plane and the workload plane are deliberately separate: the Proxmox API is reachable only from the management segment, while everything Kubernetes exposes is routed on the deployment segment. Compromising a workload does not put an attacker on the same network as the hypervisors.

The 3+3 topology means losing an entire R730 leaves a **2/3 etcd quorum** and two live workers. ECMP on the gateway withdraws the dead path automatically when the BGP hold timer expires — roughly nine seconds, with no manual intervention.

Observability spans every layer, from Proxmox host metrics and UniFi traffic flows down to a **smart plug tracking real-time power draw** of the rack.

```mermaid
flowchart TB
    NET["Internet"] --> UDM["UniFi UDM Pro<br/>zone firewall · IPS/IDS"]
    UDM --> SW["UniFi USW<br/>VLAN segmentation · LACP"]

    SW --> H1
    SW --> H2
    SW --> H3

    subgraph H1["pve-01 — Dell PowerEdge R730"]
        direction TB
        CP1["talos-cp-01<br/>control plane · .11"]
        WK1["talos-wk-01<br/>worker · .21"]
        CP1 ~~~ WK1
    end

    subgraph H2["pve-02 — Dell PowerEdge R730"]
        direction TB
        CP2["talos-cp-02<br/>control plane · .12"]
        WK2["talos-wk-02<br/>worker · .22"]
        CP2 ~~~ WK2
    end

    subgraph H3["pve-03 — Dell PowerEdge R730"]
        direction TB
        CP3["talos-cp-03<br/>control plane · .13"]
        WK3["talos-wk-03<br/>worker · .23"]
        CP3 ~~~ WK3
    end

    subgraph STATE["Shared cluster plane"]
        direction LR
        API["Kubernetes API<br/>VIP · .10"]
        ETCD["etcd quorum<br/>3 members · survives 1 loss"]
        LH["Longhorn<br/>distributed block storage"]
    end

    CP1 & CP2 & CP3 -.->|"floating VIP"| API
    CP1 & CP2 & CP3 -.->|"raft"| ETCD
    WK1 & WK2 & WK3 -.->|"volume replication"| LH
```

![Proxmox VE cluster — three quorate R730 hosts running the six Talos VMs across 144 vCPU and 94 GiB of RAM](screenshots/proxmox-cluster.png)

---

## 🗺️ Architecture

```mermaid
flowchart TB
    subgraph net["Internet"]
        CF["Cloudflare<br/>WAF · DDoS · Zero Trust"]
    end

    subgraph edge["Edge — 192.168.20.0/24"]
        UDM["UniFi UDM Pro<br/>FRR · AS 65000 · ECMP<br/>zone firewall · IPS/IDS"]
    end

    subgraph k8s["Talos Cluster — AS 65001"]
        direction TB
        GWE["External Gateway<br/>VIP 192.168.20.20<br/>Cloudflare Origin CA"]
        GWI["Internal Gateway<br/>VIP 192.168.20.99<br/>Let's Encrypt"]
        subgraph wk["Workers · BGP speakers"]
            W1["talos-wk-01<br/>.21"]
            W2["talos-wk-02<br/>.22"]
            W3["talos-wk-03<br/>.23"]
        end
        APPS["Immich · Seafile · Navidrome<br/>VLSM Calculator"]
    end

    CF -->|"HTTPS"| UDM
    UDM -->|"ECMP · 3 paths"| GWE
    UDM -->|"LAN only"| GWI
    W1 & W2 & W3 -.->|"BGP · MD5 auth<br/>VIP advertisement"| UDM
    GWE --> APPS
    GWI --> APPS
```

---

## 🛰️ Routed North-South Traffic with BGP

Workers — and **only** workers, enforced by a `nodeSelector` that excludes `node-role.kubernetes.io/control-plane` — establish BGP sessions with the UDM Pro. Cilium discovers its peer automatically, so the router's address is never hardcoded in a manifest.

```yaml
# k8s/apps/cilium/manifests/cilium.yaml
localASN: 65001 # cluster
peerASN: 65000 # UDM Pro
autoDiscovery:
  mode: "DefaultGateway" # no hardcoded peer IP
timers:
  holdTimeSeconds: 9 # failure detected in ~9s
  keepAliveTimeSeconds: 3
gracefulRestart:
  enabled: true # traffic survives a Cilium agent restart
authSecretRef: bgp-auth-secret # MD5 key injected from Vault
```

On the router side, FRR accepts nothing but the two expected VIPs and advertises nothing back into the cluster:

```text
 address-family ipv4 unicast
  maximum-paths 3                        # ECMP across all three workers
  neighbor TALOS-WORKERS route-map TALOS-IN in
  neighbor TALOS-WORKERS route-map TALOS-OUT out
!
route-map TALOS-OUT deny 10              # cluster learns zero routes
ip prefix-list TALOS-LB-VIP seq 5 permit 192.168.20.20/32
ip prefix-list TALOS-LB-VIP seq 10 permit 192.168.20.99/32
```

> **Why replace L2 announcements?** L2 mode elects a single node to own a VIP: all traffic for that service funnels through one host, and failover depends on an ARP refresh clients may cache. With BGP, all three workers advertise the same VIP, the router load-balances across them with ECMP, and failure detection becomes a protocol timer rather than a cache expiry. `maximum-paths 3` is what makes the balancing real; the restrictive route-maps are what keep the trust boundary one-directional.

![UniFi routing table — both gateway VIPs installed as BGP routes with three ECMP next-hops each](screenshots/unifi-bgp-routes.png)

_Both gateway VIPs installed as BGP routes, each load-balanced across all three workers._

---

## 🔀 Split-Horizon DNS & Dual Certificate Authorities

`gallery.xako.net` resolves differently depending on where you ask from — but presents a valid, publicly trusted certificate either way.

```mermaid
flowchart LR
    C1["External client"] -->|"public DNS"| CF["Cloudflare<br/>(proxied)"]
    CF -->|"TLS to origin"| GE["External Gateway<br/>Origin CA · ECC"]
    C2["LAN client"] -->|"internal DNS"| GI["Internal Gateway<br/>Let's Encrypt DNS-01"]
    GE --> SVC["Application service"]
    GI --> SVC
```

|            | External Gateway                               | Internal Gateway                                    |
| ---------- | ---------------------------------------------- | --------------------------------------------------- |
| **VIP**    | `192.168.20.20`                                | `192.168.20.99`                                     |
| **Issuer** | Cloudflare `ClusterOriginIssuer` (`OriginECC`) | Let's Encrypt `ClusterIssuer`, **DNS-01** challenge |
| **Path**   | Internet → Cloudflare proxy → origin           | LAN only, never leaves the switch                   |

Two concrete wins:

1. **Local traffic stays local.** A 4 GB upload to Immich from the same building hits the switch and stops there — no round trip through Cloudflare, no residential upstream bandwidth consumed.
2. **No certificate warnings internally.** A DNS-01 challenge issues a publicly trusted certificate for a service that only listens on a private IP — something HTTP-01 could never validate.

The public `A` record is kept current by `cloudflare-ddns` with `PROXIED=true`, which keeps the residential IP hidden behind Cloudflare's edge.

---

## 🚀 GitOps: App-of-Apps & Sync Waves

ArgoCD watches `k8s/root-app.yaml`, which recursively discovers every `application.yaml` under `k8s/apps/`. Adding a service means adding a directory — nothing else. Every `Application` runs with `automated` sync, `prune`, and `selfHeal`, so manual drift is reverted automatically.

Install order is enforced with **sync waves**, because the dependencies are real:

| Wave | Components                                                          | Why here                                            |
| :--: | ------------------------------------------------------------------- | --------------------------------------------------- |
| `-4` | **Cilium**                                                          | Without a CNI, no pod schedules                     |
| `-3` | **Longhorn**                                                        | Vault's raft storage needs persistent volumes       |
| `-2` | **HashiCorp Vault** (HA, raft)                                      | Source of every secret downstream                   |
| `-1` | **External Secrets**, VPA, Descheduler                              | ESO must run before any `ExternalSecret` reconciles |
| `0`  | cert-manager, CloudNativePG, MariaDB, Valkey, DDNS, VLSM Calculator | Platform services                                   |
| `1`  | **Immich**                                                          | Depends on its PostgreSQL cluster                   |
| `3`  | **Seafile**                                                         | Depends on both MariaDB and Valkey                  |

The bootstrap itself — Cilium and ArgoCD, the only two components that cannot deploy themselves — lives in `k8s/helmfile.yaml`.

![ArgoCD applications — every workload Healthy and Synced, each sourced from a path in this repository](screenshots/argocd-app-of-apps.png)

---

## 🔐 Secrets Management

**There are no secrets in this repository, and there never have been.** `.gitignore` has covered `talosconfig.yaml`, `controlplane.yaml`, and `worker.yaml` since the initial commit, and no `kind: Secret` or `stringData` block appears anywhere in the history.

```mermaid
flowchart LR
    V["Vault<br/>HA · raft"] -->|"Kubernetes auth<br/>role: eso-role"| ESO["External Secrets<br/>Operator"]
    ESO -->|"15s refresh"| S["Kubernetes Secret<br/>(in memory)"]
    S --> P["Pod"]
```

Each application declares an `ExternalSecret` pointing at the `vault-backend` `ClusterSecretStore`. What lives in Git is a **key name**, never a value:

```yaml
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  dataFrom:
    - extract:
        key: cloudflare # contents live in Vault, not here
```

Even the BGP session's MD5 key takes this path — in `frr.conf` it appears only as the placeholder `#${SECRET}#`.

---

## 📦 Workloads

| Service                     | Purpose                                                                         | Notable implementation                                                                                                                                                                  |
| --------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Immich**                  | Photo & video backup                                                            | PostgreSQL via CloudNativePG in a 3-instance cluster, with the **VectorChord** extension powering semantic search; VPA on both the server and the machine-learning deployment           |
| **Seafile**                 | File synchronization                                                            | 100 Gi on Longhorn, backed by MariaDB (three databases) with Valkey as cache                                                                                                            |
| **Navidrome**               | Music streaming                                                                 | A `seafile-client` **sidecar** syncs a Seafile library straight into the music PVC — the library is fed from any device, with no duplicated storage tier                                |
| **VLSM Calculator**         | Custom application ([repo](https://github.com/ilyas-bouktrane/vlsm-calculator)) | 3 replicas, `topologySpreadConstraints` across hosts, PDB `minAvailable: 1`, non-root with all capabilities dropped and `seccompProfile: RuntimeDefault`, liveness and readiness probes |
| **Valkey**                  | Cache layer                                                                     | 2 replicas, PDB, ACL-based authentication                                                                                                                                               |
| **CloudNativePG / MariaDB** | Stateful data                                                                   | Operator-managed clusters with pod anti-affinity and disruption budgets                                                                                                                 |
| **VPA + Descheduler**       | Scheduling                                                                      | Automatic resource right-sizing and continuous pod rebalancing across workers                                                                                                           |

![Immich pods — the three PostgreSQL instances each scheduled on a different worker node](screenshots/immich-deployment.png)

---

## 📂 Project Structure

```text
/
├── 📁 talos/                    # Per-node Talos machine config patches
│
├── 📄 frr.conf                  # BGP configuration applied on the UniFi UDM Pro
│
└── 📁 k8s/
    ├── 📄 helmfile.yaml         # Bootstrap only: Cilium + ArgoCD
    ├── 📄 root-app.yaml         # ArgoCD app-of-apps + repo credentials via ESO
    └── 📁 apps/<service>/
        ├── 📄 application.yaml  # ArgoCD Application — Helm sources + sync wave
        └── 📁 manifests/        # Cluster-specific resources (routes, PVCs, ExternalSecrets)
```

---

## 🧭 Architecture Decisions

**Talos over a general-purpose distro.** No SSH, no shell, no package manager. Debugging has to go through `talosctl`, which forces the discipline of rebuilding a node rather than patching it by hand — the same discipline that makes the cluster reproducible.

**Control planes don't speak BGP.** Excluding them from peering keeps data-plane traffic off the nodes carrying etcd.

**`longhorn-single` as the default StorageClass.** One replica with `dataLocality: strict-local` keeps volumes local to their pod: fast, with no replication traffic. It's a deliberate trade — real resilience comes from application-level replication (PostgreSQL running three instances) rather than from the block layer, and workloads needing block redundancy use the standard `longhorn` class explicitly.

**The migration is visible in the history.** The cluster first ran on a TP-Link Omada controller with MongoDB before a full move to UniFi, then a rework into the current split-horizon architecture. Debug commits weren't squashed away — the path is part of the record.

---

## 🚧 Roadmap

Honest infrastructure has a debt list. This one is mine:

- [ ] **Pin image tags** — three workloads still track `:latest`; reproducible deploys need digests
- [ ] **MariaDB Galera** — the operator runs HA, the database instance does not yet
- [ ] **Tighten Seafile grants** — currently broader than the three databases it actually needs
- [ ] **In-cluster observability** — Prometheus + Grafana with alerting on BGP session state and Longhorn volume health
- [ ] **Off-site backups** — Longhorn volume snapshots replicated to remote object storage
- [ ] **Cilium NetworkPolicies** — namespace-level isolation to complete the Zero-Trust posture

---

## 🔁 Reproducing the Cluster

```bash
# 1. Apply Talos machine configs (base secrets generated outside this repo)
talosctl apply-config --nodes <ip> --file controlplane.yaml \
  --config-patch @talos/talos-cp-01.yaml
talosctl bootstrap --nodes <ip-cp-01>

# 2. Bootstrap the only two components that can't deploy themselves
helmfile -f k8s/helmfile.yaml apply

# 3. Initialize Vault, enable Kubernetes auth, create the eso-role

# 4. Hand everything else to ArgoCD
kubectl apply -f k8s/root-app.yaml
```

From step 4 onward, the cluster converges on its own to whatever `main` describes.

---

## 👨‍💻 Contact

Built by **Ilyas Bouktrane** — Computer Science Technology student (Network Infrastructure & Security) at Collège de Bois-de-Boulogne.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/ilyas-bouktrane)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ilyas-bouktrane)

Feel free to reach out with any questions or opportunities!
