# Orange Pi 5 Plus Security Baseline and PKI Automation

Automated PKI generation, OpenSSH CA setup, and Orange Pi node security hardening using Ansible.

## Architecture and Lifespans

- **Root CA**: 10 years (`3650` days), ECC secp384r1.
- **Intermediate CA**: 5 years (`1825` days), ECC secp384r1.
- **TLS Application / Ingress Certificates**: 1 year (`365` days), ECC secp384r1.
- **SSH User Certificate**: 1 year (`+52w`), Ed25519.

## Security Baseline

- SSH relocated to port `2222`.
- Direct password authentication disabled.
- Passwordless access enforced via OpenSSH CA signed user certificates.
- Incoming SSH traffic restricted by UFW strictly to admin workstation (`192.168.1.190`).
- Default firewall policy: incoming deny, outgoing allow.

## Directory Structure

```text
k3s-ansible-opi5plus/
├── ansible.cfg
├── inventory.example.ini
├── inventory.ini           # (gitignored, copy from inventory.example.ini)
├── group_vars/
│   └── all/
│       ├── all.yml         # Generic public defaults
│       ├── vault.yml.example
│       └── vault.yml       # (gitignored, copy from vault.yml.example)
├── playbooks/
│   ├── 01_setup_pki.yml
│   ├── 01.5_setup_admin_hosts.yml
│   ├── 02_harden_nodes.yml
│   ├── 03_deploy_k3s.yml
│   ├── 04_cilium_traefik.yml
│   ├── 05_deploy_argocd.yml
│   └── issue_cert.yml
├── roles/
│   ├── pki_ca/
│   ├── ssh_ca/
│   ├── node_hardening/
│   ├── k8s_prereqs/
│   ├── k3s_cluster/
│   ├── cilium/
│   ├── traefik/
│   └── argocd/
├── cilium/
├── traefik/
├── argocd/
└── README.md
```

## Execution Workflow

### Step 0: Configure Inventory and Secrets

Copy the example templates and set your actual node IPs and cluster token:

```bash
# 1. Copy example inventory and edit your node IP addresses:
cp inventory.example.ini inventory.ini

# 2. Copy example vault template and set your secure cluster token:
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
```

### Step 1: Initialize PKI and SSH CA

Run the local setup playbook to generate the Root CA, Intermediate CA, SSH CA, and local admin certificate:

```bash
ansible-playbook playbooks/01_setup_pki.yml
```


### Step 1.5: Configure Local Admin Workstation /etc/hosts

Configure all cluster DNS mappings (`k8s.tamer.io`, `opi01..03`, `traefik.opi.tamer.io`, `argocd.opi.tamer.io`) on the admin workstation:

```bash
ansible-playbook -K playbooks/01.5_setup_admin_hosts.yml
```

### Step 2: Apply Node Hardening

Update apt repositories, upgrade system packages, install baseline utilities (iSCSI, NFS, eBPF tools), configure SSH CA trust, and enforce UFW firewall rules:

```bash
ansible-playbook playbooks/02_harden_nodes.yml
```


### Step 3: Deploy HA K3s Cluster with Kube-VIP

Prepare node prerequisites (sysctl, bpffs, swapoff, /etc/hosts) and deploy HA K3s (embedded etcd) with Kube-VIP (192.168.1.200):

```bash
ansible-playbook playbooks/03_deploy_k3s.yml
```

### Step 4: Deploy Cilium CNI and Traefik Ingress/Gateway

Deploy Cilium with eBPF kube-proxy replacement, L2 announcements, and IP pool; then deploy Traefik Gateway with 3 replicas, Intermediate CA TLS, and dashboard at `https://traefik.opi.tamer.io/`:

```bash
ansible-playbook playbooks/04_cilium_traefik.yml
```

### Step 5: Deploy ArgoCD GitOps

Deploy ArgoCD with Intermediate CA TLS certificate, Traefik IngressRoute (supporting Web UI and cleartext HTTP/2 gRPC for CLI), and access the UI at `https://argocd.opi.tamer.io/`:

```bash
ansible-playbook playbooks/05_deploy_argocd.yml
```

### Extra: Issue TLS Certificates for Applications

Issue a 1-year certificate signed by the Intermediate CA:

```bash
ansible-playbook playbooks/issue_cert.yml -e "cert_domain=example.tamer.io"
```


