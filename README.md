# Orange Pi 5 Plus HA K3s Cluster & Zero-Trust Hardening

Automated provisioning of a production-ready, highly available **K3s Kubernetes cluster** across 3 bare-metal nodes (Orange Pi 5 Plus or any ARM64/x86 Linux boards) with **Cilium (eBPF)**, **Traefik Gateway**, **OpenSSH CA Hardening**, and **ArgoCD GitOps**.

---

## 📋 Prerequisites

Before running the playbooks, ensure you have the following ready:

### 1. Admin / Bootstrap Machine (Workstation)
* **OS:** Linux, macOS, or WSL2.
* **Tools Installed:**
  * `ansible` (>= 2.14)
  * `python3` & `python3-pip`
  * `openssl` & `ssh-keygen`
  * `git`
* **Network:** Directly connected to the same LAN as the target nodes.
* **SSH Key:** An existing local SSH key pair (e.g., `~/.ssh/id_ed25519` or `~/.ssh/id_rsa`).

### 2. 3x Target Nodes (e.g., Orange Pi 5 Plus)
* **OS:** Clean, freshly flashed **Ubuntu 24.04 LTS (Server)**.
* **Network:** Each node assigned a **static IP** (via router DHCP reservation or Netplan).
* **User:** A standard user with passwordless `sudo` rights (default: `ubuntu`).
* **SSH:** Standard OpenSSH server running on default port `22` with key or password access.
* **Connectivity:** Reachable from the bootstrap workstation (`ping` and `ssh ubuntu@<ip>` work).

---

## ⚙️ Configuration (Files to Edit)

Only **2 files** need to be customized before running the playbooks:

### Step 1: Node Inventory (`inventory.ini`)

Copy the example template and specify the static IPs of your 3 nodes:

```bash
cp inventory.example.ini inventory.ini
```

Edit `inventory.ini`:
```ini
[opi_nodes]
opi01 ansible_host=192.168.1.201
opi02 ansible_host=192.168.1.202
opi03 ansible_host=192.168.1.203
```

---

### Step 2: Secrets & Cluster Variables (`group_vars/all/vault.yml`)

Copy the secrets template (this file is `.gitignore`d and will never be pushed to Git):

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
```

Edit `group_vars/all/vault.yml` and adjust the essential values:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `base_domain` | Your root or local domain | `homelab.internal` or `example.com` |
| `admin_client_ip` | IP of your bootstrap machine (for UFW SSH whitelist) | `192.168.1.100` |
| `cluster_network_subnet` | Local LAN subnet | `192.168.1.0/24` |
| `kube_vip_address` | Static Virtual IP for Kube-VIP HA API endpoint | `192.168.1.200` |
| `k3s_token` | Secret cluster join token (generate a random string) | `openssl rand -hex 32` |
| `cilium_lb_pool_start/stop` | IP range reserved for Cilium LoadBalancers | `192.168.1.210` - `192.168.1.230` |
| `traefik_lb_ip` | Static LoadBalancer IP for Traefik Ingress | `192.168.1.210` |
| `cluster_hosts_entries` | Hostname-to-IP mappings for `/etc/hosts` | Nodes, VIP, and Admin IPs |

---

## 🚀 Execution Workflow

Run the playbooks sequentially from your bootstrap machine:

### 1. Initialize PKI & SSH CA
Generates internal Root CA, Intermediate CA, and OpenSSH CA, then signs an SSH user certificate for your local admin user:
```bash
ansible-playbook playbooks/01_setup_pki.yml
```

### 1.5. Update Local `/etc/hosts` (Bootstrap Machine)
Maps cluster domains (`k8s.<domain>`, `traefik.opi.<domain>`, `argocd.opi.<domain>`) on your local workstation:
```bash
ansible-playbook -K playbooks/01.5_setup_admin_hosts.yml
```

### 2. Harden Nodes & Enforce Zero-Trust SSH
Installs baseline packages, moves SSH to port `2222`, disables password authentication, enforces OpenSSH CA certificate validation, and sets UFW firewall to incoming deny except from your admin IP:
```bash
ansible-playbook playbooks/02_harden_nodes.yml
```

### 3. Deploy HA K3s Cluster & Kube-VIP
Prepares kernel parameters (eBPF, sysctl, swapoff) and provisions a 3-node HA K3s cluster with embedded etcd and Kube-VIP virtual IP:
```bash
ansible-playbook playbooks/03_deploy_k3s.yml
```

### 4. Deploy Cilium CNI & Traefik Ingress
Deploys Cilium with eBPF kube-proxy replacement and L2 announcement IP pool, followed by HA Traefik Ingress controller:
```bash
ansible-playbook playbooks/04_cilium_traefik.yml
```

### 5. Deploy ArgoCD GitOps Engine
Deploys ArgoCD in HA mode with Traefik IngressRoute, gRPC streaming, and TLS certificate integration:
```bash
ansible-playbook playbooks/05_deploy_argocd.yml
```

---

## 🔍 Verification & Access

### Kubernetes Cluster Access
The kubeconfig file is automatically downloaded to the project root:
```bash
export KUBECONFIG=$(pwd)/kubeconfig
kubectl get nodes -o wide
```

### Hardened SSH Access
Since port `22` and password authentication are disabled, connect using port `2222` with your CA-signed certificate:
```bash
ssh -p 2222 ubuntu@192.168.1.201
```

### Web Interfaces
* **ArgoCD:** `https://argocd.opi.<base_domain>`
* **Traefik Dashboard:** `https://traefik.opi.<base_domain>`
