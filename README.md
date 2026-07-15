# 🚀 SUSE Harvester & Rancher Prime — Automated Bare-Metal Deployment

![Ansible](https://img.shields.io/badge/Ansible-%E2%89%A5%202.14-EE0000?logo=ansible&logoColor=white)
![Harvester](https://img.shields.io/badge/Harvester-v1.7.1-30BA78?logo=suse&logoColor=white)
![Rancher Prime](https://img.shields.io/badge/Rancher%20Prime-2.13.3-0075A8?logo=rancher&logoColor=white)
![RKE2](https://img.shields.io/badge/RKE2-v1.33.6+rke2r1-blue)
![License](https://img.shields.io/badge/License-Apache--2.0-lightgrey)

Ansible automation that installs a **3-node SUSE Harvester HCI cluster on bare metal via PXE boot** — fully unattended — and then deploys a highly available **Rancher Prime management cluster** (3 × RKE2 VMs, built with the SUSE Edge Image Builder) on top of it.

Everything runs from a single openSUSE Leap 15.6 admin host. Node identity is anchored to **MAC addresses** end to end: from the iPXE boot menu, through the Harvester install configs, down to the static IPs of the Rancher VMs.

```
┌─────────────────────────────────────────────────────────────────┐
│  Admin / PXE host (openSUSE Leap 15.6)                          │
│  dnsmasq · TFTP · nginx · NFS · Edge Image Builder · Ansible    │
└───────────────┬─────────────────────────────────────────────────┘
                │ PXE boot + fully automated installation
┌───────────────▼─────────────────────────────────────────────────┐
│  Harvester HCI cluster (3 bare-metal nodes, VIP)                │
│  Longhorn storage · cluster networks · VM images · backups      │
│   ┌──────────────────────────────────────────────┐              │
│   │  Rancher Prime cluster (3 RKE2 VMs, LB VIP)  │              │
│   │  cert-manager · Rancher · Harvester plugin   │              │
│   └──────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📑 Table of Contents

- [Features](#-features)
- [Requirements](#-requirements)
- [Quick Start](#-quick-start)
- [Deployment Workflow](#-deployment-workflow)
- [Configuration](#%EF%B8%8F-configuration)
- [Secrets Handling](#-secrets-handling)
- [Repository Layout](#-repository-layout)
- [How It Works](#-how-it-works)
- [Troubleshooting](#-troubleshooting)
- [License](#-license)

---

## ✨ Features

- **Zero-touch bare-metal install** — dnsmasq proxy DHCP, TFTP and iPXE chain-loading; per-node boot scripts and Harvester create/join configs selected by MAC address
- **VLAN-aware** — untagged and VLAN-tagged management networks, with the correct kernel VLAN syntax for Harvester `< 1.7` and `>= 1.7`
- **Bootable Rancher image** — the SUSE Edge Image Builder produces a self-installing SL Micro ISO containing RKE2, cert-manager and Rancher Prime; the 3-node cluster comes up without any manual step
- **Static IPs without DHCP reservations** — nmstate profiles are matched to each VM by MAC address at first boot
- **Full day-1 configuration via API** — storage classes, disk tagging, cluster networks, dedicated storage network, VM images, NFS backup target, IP pool, L4 load balancer, Rancher UI plugin and Harvester cluster import
- **Idempotent** — every playbook can be re-run safely; existing resources are detected and skipped
- **Sanitized for public use** — no credentials, keys or personal data; every environment-specific value is a variable

## 📋 Requirements

| Component | Requirement |
|---|---|
| **Admin host** | openSUSE Leap 15.6, ≥ 2 vCPU, ≥ 8 GB RAM, ≥ 120 GB free disk (enforced by assertions) |
| **Harvester nodes** | 3 × bare metal, PXE-bootable NIC, known MAC addresses |
| **Ansible** | ≥ 2.14 with `community.general` and `kubernetes.core` |
| **Network** | One shared L2 segment; free static IPs for all nodes + 2 VIPs (Harvester, Rancher) |
| **Licensing** | SUSE Customer Center registration code (SL Micro), Rancher Prime access |
| **TLS** | Certificate, key and root CA for the Rancher FQDN in `./cert/` |

## ⚡ Quick Start

```bash
# 1. Clone and install the required collections
git clone https://github.com/<your-org>/harvester-deployment.git
cd harvester-deployment
ansible-galaxy collection install -r requirements.yml

# 2. Adapt the configuration — every value marked CHANGE_ME must be set
grep -rn "CHANGE_ME" vars/

# 3. Place your TLS material
cp /path/to/tls.crt /path/to/tls.key /path/to/cacerts.pem ./cert/

# 4. Run the deployment (see the workflow below)
ansible-playbook 01-playbook-prepare.yaml
```

## 🔁 Deployment Workflow

| # | Playbook | Purpose |
|---|---|---|
| 01 | `01-playbook-prepare.yaml` | Turn the admin host into a PXE server, download all boot artifacts, render iPXE scripts + node configs, install kubectl |
| 02 | `02-playbook-edge-image-builder.yaml` | Build the self-installing Rancher ISO with the Edge Image Builder |
| 04 | `04-playbook-nfs-server.yaml` | Provide an NFS export as the Harvester backup target |
| 11 | `11-playbook-pxe-onsite-validate-setup.yaml` | Validate the PXE setup on site (services, files, firewall, HTTP) — report only |
| ⏻ | **Power on the bare-metal nodes** | Node 1 creates the cluster and claims the VIP, nodes 2 + 3 join automatically |
| 12 | `12-playbook-harvester-config.yaml` | Configure Harvester: SSH key, backup target, disks, `ssd` storage class, images, networks |
| 13 | `13-playbook-rancher.yaml` | Create the 3 Rancher VMs, the IP pool and the 80/443 load balancer |
| 15 | `15-playbook-rancher-config.yaml` | Install the Harvester UI plugin and import Harvester into Rancher |

```bash
ansible-playbook 01-playbook-prepare.yaml
ansible-playbook 02-playbook-edge-image-builder.yaml
ansible-playbook 04-playbook-nfs-server.yaml
ansible-playbook 11-playbook-pxe-onsite-validate-setup.yaml
# → power on the Harvester nodes and wait for the installation to finish
# → download the kubeconfig from the Harvester UI → ./harvester_kubeconfig.yaml
ansible-playbook 12-playbook-harvester-config.yaml
ansible-playbook 13-playbook-rancher.yaml
# → download the kubeconfig from the Rancher UI → ./rancher_kubeconfig.yaml
ansible-playbook 15-playbook-rancher-config.yaml
```

## ⚙️ Configuration

All environment-specific values live in `vars/` — nothing is hard-coded anywhere else.

| File | Contents |
|---|---|
| `vars/network.yml` | Subnet (`network_prefix`), gateway, DNS, domain, VIPs, node IPs, VLANs, MTU |
| `vars/infrastructure.yml` | Versions, node names + MAC addresses, credentials (placeholders), image names, TLS paths |
| `vars/general.yml` | Package list, directory layout, Harvester add-ons, image upload list |

Key variables at a glance:

```yaml
network_prefix: 192.168.100          # first three octets of your network
pxe_ip: "{{ network_prefix }}.45"    # the admin host itself
harvester_vip: "{{ network_prefix }}.40"
rancher_prime_vip: "{{ network_prefix }}.50"
harvester_node1_mac1: "AA:BB:CC:00:00:01"   # decides create vs. join
vlan_id_harvester: "0"               # "0" = untagged, >0 = VLAN templates
```

## 🔐 Secrets Handling

This repository ships **without any real credentials**. Before running:

1. Replace every `CHANGE_ME` (cluster tokens, passwords, SCC code, SSH public key).
2. Generate the encrypted OS password for the image: `openssl passwd -6 'YourPassword'`
3. Prefer **Ansible Vault** for all secrets:
   ```bash
   ansible-vault encrypt_string 'MySecret' --name 'harvester_cluster_token'
   ```

Kubeconfigs, certificates and keys are excluded via `.gitignore`; sanitized `*.example` files document the expected structure. **Never** remove those ignore rules.

## 📁 Repository Layout

```
.
├── 01-playbook-prepare.yaml                 # PXE server setup + downloads
├── 02-playbook-edge-image-builder.yaml      # Rancher image build (EIB)
├── 04-playbook-nfs-server.yaml              # NFS backup target
├── 11-playbook-pxe-onsite-validate-setup.yaml   # on-site validation
├── 12-playbook-harvester-config.yaml        # Harvester day-1 config
├── 13-playbook-rancher.yaml                 # Rancher VMs + IP pool + LB
├── 15-playbook-rancher-config.yaml          # Rancher config + cluster import
├── requirements.yml                         # Ansible collections
├── vars/                                    # ← all environment settings
├── cert/                                    # TLS material (Git-ignored)
├── *.kubeconfig.yaml.example                # kubeconfig templates
└── roles/
    ├── pxe_*          # PXE stack, boot artifacts, iPXE + install configs
    ├── eib_*          # Edge Image Builder build tree (OS, network, k8s)
    ├── harvester_*    # Harvester API: storage, networks, images, VMs, LB
    └── rancher_*      # Rancher API: chart repos, UI plugin, import
```

Every playbook, role and template carries an English header comment explaining what it does and why.

## 🔍 How It Works

1. **PXE stage** — dnsmasq answers PXE requests and serves `undionly.kpxe` (BIOS) or `ipxe.efi` (UEFI) via TFTP. iPXE then chain-loads `boot.ipxe` over HTTP, which matches the node's MAC address and jumps to its node-specific script.
2. **Harvester install** — the node script boots the Harvester kernel with `harvester.install.automatic=true` and a node-specific config URL. Node 1 uses *create* mode (claims the VIP), nodes 2 + 3 use *join* mode.
3. **Image build** — the Edge Image Builder consumes a declarative `definition.yaml` plus nmstate network profiles, RKE2 configs, Helm values and TLS manifests, and emits one ISO that installs the entire Rancher stack.
4. **VM stage** — Ansible creates block PVCs (ISO + OS disk) and KubeVirt VMs with fixed MACs on Harvester. A Harvester `IPPool` + `LoadBalancer` expose Rancher on its VIP (80/443).
5. **Integration** — a `ClusterRegistrationToken` is created in Rancher, and the resulting import URL is written into Harvester's `cluster-registration-url` setting → Harvester appears in Rancher's Virtualization Management.

## 🛠 Troubleshooting

| Symptom | Fix |
|---|---|
| Node does not PXE boot | Run playbook 11; watch `journalctl -u dnsmasq -f`; verify the node's MAC in `vars/infrastructure.yml` |
| Harvester installer hangs | Check the node console; test `curl http://<pxe_ip>/harvester/boot.ipxe` |
| Image upload stays at 0 % | nginx on the admin host must still be running (role `pxe_onsite_on`) |
| Rancher VMs get no IP | VM MACs (playbook 13) must exactly match the EIB network profiles (playbook 02) |
| Playbook 13 fails on the image ID | Check `vm_image_id` / `vm_disk0_storage_class` against the names in the Harvester UI → Images |
| Cluster import missing | Playbook 15 needs **both** kubeconfigs; inspect `kubectl get settings -n harvester` |

## 📄 License

Apache-2.0 — adjust as needed.

*SUSE, Harvester, Rancher and SUSE Linux Micro are trademarks of SUSE LLC. This is a community automation project and not an official SUSE product.*
