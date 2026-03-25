# 🔴 rotom-dex-infra

> Ansible playbook to provision and configure the Rotom-Dex observability stack on any Linux VM.

---

## 🧱 Stack

| Service | Version | Role |
|---|---|---|
| 🔍 Elasticsearch | 8.12.0 | Data store |
| 📊 Kibana | 8.12.0 | Visualisation |
| 📡 APM Server | 7.17.0 | RUM data intake |
| 📂 Filebeat | 8.12.0 | Log shipping |
| 📈 Grafana | latest | Dashboards |
| 🌐 Cloudflared | latest | Secure tunnel (no open ports) |

---

## 🏗️ Architecture

```
Visitor → Cloudflare Worker → Cloudflare Tunnel (apm.yourdomain.com) → VM → APM Server → Elasticsearch
```

---

## ✅ Prerequisites

### 💻 Local machine
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html) installed (recommended via WSL on Windows)
- SSH key pair generated (`~/.ssh/id_ed25519`)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) installed (for Cloudflare Worker secrets)

### ☁️ VM (any cloud provider or VPS)
- Debian 12 (Bookworm) — tested on GCP, should work on AWS EC2, Azure, DigitalOcean, Hetzner, etc.
- At least 30GB disk
- At least 4GB RAM (2 vCPU minimum)
- Firewall: **no ports need to be open** (Cloudflare Tunnel handles all ingress) 🔒

> 💡 **Recommended providers:** GCP (free trial), AWS EC2 (free tier), DigitalOcean ($6/mo Droplet), Hetzner (cheapest for specs)

---

## 🚀 Setup Guide

### 1️⃣ Create a VM

Spin up a **Debian 12** VM on your preferred provider:

| Provider | Equivalent spec |
|---|---|
| GCP | e2-medium (2 vCPU, 4GB RAM) |
| AWS | t3.medium |
| DigitalOcean | Basic 4GB Droplet |
| Hetzner | CX22 |
| Azure | B2s |

Make note of your VM's **public IP address**.

---

### 2️⃣ Generate SSH Key Pair

If you don't already have an SSH key:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

This creates `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public).

---

### 3️⃣ Add SSH Key to Your VM

How you add the key depends on your provider:

**GCP:**
1. Go to **Compute Engine → VM Instances → Edit**
2. Under **SSH Keys**, click **Add item**
3. Paste your public key as: `YOUR_USERNAME:ssh-ed25519 AAAA...`

**AWS EC2:**
- Select your key pair during instance creation, or use `ssh-copy-id`

**DigitalOcean / Hetzner:**
- Add your public key in the provider dashboard before creating the Droplet/server

**Any provider (universal method):**
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub YOUR_USERNAME@YOUR_VM_IP
```

Verify SSH access works:
```bash
ssh YOUR_USERNAME@YOUR_VM_IP
```

---

### 4️⃣ Enable Passwordless Sudo

Connect to your VM (via SSH or your provider's browser console) and run:

```bash
sudo visudo
```

Add this line at the end:
```
YOUR_USERNAME ALL=(ALL) NOPASSWD: ALL
```

Save and exit (`Ctrl+X`, `Y`, `Enter`).

---

### 5️⃣ Set Up Cloudflare Tunnel 🌐

On your VM, install and authenticate cloudflared:

```bash
# Download and install cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb

# Login (opens browser URL — authorize with your Cloudflare account)
cloudflared tunnel login

# Create named tunnel
cloudflared tunnel create rotom-dex

# Create DNS routes (requires a domain on Cloudflare)
cloudflared tunnel route dns rotom-dex apm.yourdomain.com
cloudflared tunnel route dns rotom-dex es.yourdomain.com
```

Copy the credentials file locally for Ansible:
```bash
# On VM — expose credentials temporarily
sudo cp ~/.cloudflared/<tunnel-id>.json ~/credentials.json
sudo chmod 644 ~/credentials.json

# On your local machine
scp YOUR_USERNAME@YOUR_VM_IP:~/credentials.json files/cloudflared-credentials.json

# Clean up on VM
rm ~/credentials.json
```

---

### 6️⃣ Clone This Repository

```bash
git clone https://github.com/bobbyricardy/rotom-dex-infra.git
cd rotom-dex-infra
```

---

### 7️⃣ Configure Inventory

```bash
cp inventory.ini.example inventory.ini
```

Edit `inventory.ini` with your VM details:
```ini
[vm]
rotom-dex ansible_host=YOUR_VM_IP ansible_user=YOUR_USERNAME ansible_ssh_private_key_file=~/.ssh/id_ed25519 ansible_become=true ansible_become_method=sudo ansible_become_flags='-n'
```

---

### 8️⃣ Configure Variables

Edit `vars/main.yml` with your settings:
```yaml
elk_version: "8.12.0"
apm_server_version: "7.17.0"
grafana_version: "latest"
cloudflared_tunnel_id: "YOUR_TUNNEL_ID"
cloudflared_hostname_apm: "apm.yourdomain.com"
cloudflared_hostname_es: "es.yourdomain.com"
elk_stack_dir: "/home/{{ ansible_user }}/elk-stack"
```

---

### 9️⃣ Configure Secrets (Vault) 🔐

Create `vars/vault.yml` with your passwords:
```yaml
elastic_password: "your_elastic_password"
kibana_password: "your_kibana_password"
filebeat_password: "your_filebeat_password"
grafana_password: "your_grafana_password"
```

Encrypt it:
```bash
ansible-vault encrypt vars/vault.yml
```

> ⚠️ **Never commit** `vars/vault.yml` or `files/cloudflared-credentials.json` — they are in `.gitignore`.

---

### 🔟 Test Connection

Add your SSH key to the agent:
```bash
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519
```

Test Ansible connectivity:
```bash
ansible vm -i inventory.ini -m ping
```

You should see `pong` 🏓

---

### Run the Playbook 🎉

Dry run first (no changes made):
```bash
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass --check
```

Run for real:
```bash
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

---

## 🖥️ Accessing Services

All services are bound to `localhost` on the VM. Use SSH tunnelling to access them locally:

```bash
ssh -L 5601:localhost:5601 \
    -L 3000:localhost:3000 \
    -L 9200:localhost:9200 \
    YOUR_USERNAME@YOUR_VM_IP
```

Then open:

| Service | URL |
|---|---|
| 📊 Kibana | http://localhost:5601 |
| 📈 Grafana | http://localhost:3000 |
| 🔍 Elasticsearch | http://localhost:9200 |

---

## ☁️ Cloudflare Worker Setup

After provisioning, set the Worker secrets:

```bash
wrangler secret put APM_SERVER_URL
# Enter: https://apm.yourdomain.com

wrangler secret put ES_API_KEY
# Enter: your_elasticsearch_api_key
```

Create a read-only Elasticsearch API key:
```bash
source ~/elk-stack/.env
curl -u elastic:${ELASTIC_PASSWORD} -X POST http://localhost:9200/_security/api_key \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rotom-dex-readonly",
    "role_descriptors": {
      "readonly": {
        "cluster": ["monitor"],
        "indices": [{ "names": ["apm-*"], "privileges": ["read"] }]
      }
    }
  }'
```

---

## 📁 File Structure

```
rotom-dex-infra/
├── inventory.ini.example        # VM connection template
├── inventory.ini                # your VM config (gitignored)
├── playbook.yml                 # main Ansible playbook
├── vars/
│   ├── main.yml                 # non-secret variables
│   └── vault.yml                # 🔐 encrypted secrets (gitignored)
├── templates/
│   ├── docker-compose.yml.j2
│   ├── apm-server.yml.j2
│   ├── filebeat.yml.j2
│   ├── cloudflared-config.yml.j2
│   └── env.j2
├── files/
│   └── cloudflared-credentials.json  # 🔐 tunnel credentials (gitignored)
└── README.md
```
