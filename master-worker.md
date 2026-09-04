# K3s Kubernetes Cluster Setup Guide (Master & Worker Nodes)

This guide provides the exact step-by-step instructions to turn any **two Ubuntu/Linux VMs** (or AWS EC2 instances) into a production-ready **K3s Master (Control Plane)** and **Worker Node** cluster.

---

##  Environment & Prerequisites

### 1. VM Allocation
| Role | Recommended Specs | Open Firewall Inbound Ports |
| :--- | :--- | :--- |
| **Master Node (Control Plane)** | 2+ vCPU, 4GB+ RAM, 30GB+ Disk | `22` (SSH), `6443` (K3s API), `80` (HTTP), `443` (HTTPS), `10250` (Kubelet metrics) |
| **Worker Node** | 2+ vCPU, 4GB+ RAM, 30GB+ Disk | `22` (SSH), `10250` (Kubelet metrics), `80` (HTTP), `443` (HTTPS) |

> **Important:** Ensure internal traffic between Master and Worker is allowed on **UDP Port 8472** (Flannel Overlay Network) and **TCP Port 10250** (Kubelet metrics).

---

##  Step 1: System Preparation (Run on BOTH VMs)

Log into both VMs via SSH and prepare the OS environment:

```bash
# Update package list and system dependencies
sudo apt update && sudo apt upgrade -y

# Install curl and iptables (required for K3s networking)
sudo apt install -y curl iptables tar
```

*(Optional)* Set clean hostnames for clarity:
- On **Master VM**:
  ```bash
  sudo hostnamectl set-hostname k3s-master
  ```
- On **Worker VM**:
  ```bash
  sudo hostnamectl set-hostname k3s-worker
  ```

---

##  Step 2: Install & Initialize Master Node (Control Plane)

SSH into your **Master VM**:

```bash
ssh -i <YOUR_KEY>.pem ubuntu@<MASTER_IP>
```

### 1. Install K3s Server
Run the official single-command K3s installer:

```bash
curl -sfL https://get.k3s.io | sh -
```

### 2. Verify K3s Server Status
Check that the K3s systemd service is active and running:

```bash
sudo systemctl status k3s
```

### 3. Extract the Node Join Token
K3s generates a secure cluster token. Copy this token (you will need it on the worker node):

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
*(Example output: `K10a1b2c3d4e5f6...::server:7890abcdef...`)*

### 4. Verify Master Node Status locally
```bash
sudo kubectl get nodes
```

---

##  Step 3: Join Worker Node to the Cluster

SSH into your **Worker VM**:

```bash
ssh -i <YOUR_KEY>.pem ubuntu@<WORKER_IP>
```

### 1. Join Cluster as Agent Node
Run the K3s installation script pointing to the **Master VM's IP** and using the **Token** extracted from Step 2:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<MASTER_IP_OR_PRIVATE_IP>:6443 \
  K3S_TOKEN=<TOKEN_FROM_MASTER> \
  sh -
```

> **Tip:** If Master and Worker are inside the same VPC / local subnet, use the **Master Private IP** for `K3S_URL`. If across public networks, use the **Master Public IP** and ensure port `6443` is open on the Master's firewall/Security Group.

### 2. Verify Worker Agent Status
Check that the agent service is running:

```bash
sudo systemctl status k3s-agent
```

---

##  Step 4: Verify Cluster Status & Configure Remote Access

### 1. Verify Nodes from Master
Return to the **Master VM** (or run via kubectl):

```bash
sudo kubectl get nodes -o wide
```

**Expected Output:**
```text
NAME         STATUS   ROLES                  AGE   VERSION
k3s-master   Ready    control-plane,master   3m    v1.36.3+k3s1
k3s-worker   Ready    <none>                 1m    v1.36.3+k3s1
```

### 2. Setup `kubeconfig` for Local Machine Access (Optional)

On the **Master VM**, retrieve the `/etc/rancher/k3s/k3s.yaml` file:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```

1. Copy the YAML text.
2. Save it locally on your development machine (e.g., `C:\Users\username\.kube\config` or `c:\Users\username\Downloads\k3s.yaml`).
3. Edit line `18` of the file, replacing `https://127.0.0.1:6443` with `https://<MASTER_PUBLIC_IP>:6443`.
4. Test access from your local terminal:
   ```powershell
   kubectl get nodes --kubeconfig="c:\Users\username\Downloads\k3s.yaml"
   ```

---

##  Step 5: Label Nodes & Organize Workloads

*(Optional)* Label the worker node so workloads can target it explicitly:

```bash
# Mark node as worker
sudo kubectl label node k3s-worker node-role.kubernetes.io/worker=worker
```

---

##  Step 6: Management & Uninstall Commands

### Restart Services
- **Master:** `sudo systemctl restart k3s`
- **Worker:** `sudo systemctl restart k3s-agent`

### Complete Uninstall / Reset
- **To reset Master:**
  ```bash
  /usr/local/bin/k3s-uninstall.sh
  ```
- **To reset Worker:**
  ```bash
  /usr/local/bin/k3s-agent-uninstall.sh
  ```
