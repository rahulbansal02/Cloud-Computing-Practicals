# ☁ OpenStack on AWS — Complete PuTTY Lab Guide
### Practical 07 | Cloud Computing Lab

---

> **⚠ FREE TIER LIMITATION — READ FIRST**
> AWS Free Tier (`t2.micro` / `t3.micro`) **cannot** run OpenStack. DevStack requires at least 4 vCPUs, 8 GB RAM, and 50 GB disk — plus nested virtualization. Free Tier instances have 1 vCPU and 1 GB RAM; the install will crash immediately.
> This guide uses a **t3.large (~$0.06/hr)**. Terminate the instance immediately after the lab. Total expected spend: **under $0.20**.

---

## Phase 0 — Architecture Overview

| OpenStack Component | AWS Equivalent | Function |
|---|---|---|
| **Nova** | EC2 | Compute & VM management |
| **Neutron** | VPC | Networking, subnets, routing |
| **Glance** | AMI | VM image storage & retrieval |
| **Cinder** | EBS | Block storage volumes |
| **Keystone** | IAM | Identity & authentication |
| **Horizon** | AWS Console | Web-based dashboard UI |

---

## Phase 1 — Provision the AWS EC2 Instance

> **💡 Cost Control Tip:** Before launching, go to CloudWatch → Billing → Create Alarm at $2. This protects you from accidentally leaving the instance running.

### Step 1.1 — Launch EC2 Instance (AWS Console)

**1. Open EC2 → Launch Instance**
Sign in to AWS Console → Services → EC2 → Launch Instance.

**2. Choose AMI**
Select **Ubuntu Server 22.04 LTS (64-bit x86)**. Do NOT use 24.04 — DevStack has compatibility issues with it as of 2025.

**3. Choose Instance Type**
Select **t3.large** (2 vCPUs, 8 GB RAM). This is the minimum that will actually complete the install without crashing.

**4. Configure Storage**
Change the Root volume from the default **8 GB → 50 GB (gp3)**. DevStack needs space for images and logs.

**5. Key Pair**
Create a new key pair → **RSA → `.ppk` format** (PuTTY-compatible). Download and save the `.ppk` file securely — you cannot re-download it.

**6. Security Group**
Create a new security group with the following inbound rules:

| Type | Port | Protocol | Source |
|---|---|---|---|
| SSH | 22 | TCP | My IP (for PuTTY access) |
| HTTP | 80 | TCP | Anywhere (Horizon dashboard) |
| Custom TCP | 8080 | TCP | Anywhere (OpenStack APIs) |
| All ICMP - IPv4 | All | ICMP | Anywhere (ping) |

**7. Launch**
Click Launch Instance. Note the **Public IPv4 address** from the EC2 dashboard — you need it for PuTTY.

---

## Phase 2 — Connect via PuTTY

> **⚠ Session Timeout Fix:** Before connecting, go to **Connection** and set **"Seconds between keepalives"** to `60`. This prevents PuTTY from disconnecting during the 30–45 minute DevStack install.

**1. Open PuTTY**
In **Host Name**, enter:
```
ubuntu@<your-ec2-public-ip>
```
Example: `ubuntu@54.234.11.22`

**2. Load your `.ppk` key**
Go to **Connection → SSH → Auth → Credentials** → Browse → select your `.ppk` file.

**3. Save the session**
Go back to **Session**, type a name (e.g. `OpenStack-Lab`), click **Save** for easy reconnection.

**4. Connect**
Click **Open**. Accept the host key fingerprint. You should see:
```
ubuntu@ip-xxx-xxx-xxx-xxx:~$
```

---

## Phase 3 — Install OpenStack (DevStack)

### Step 3.1 — Prepare the System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-pip curl
```

### Step 3.2 — Create the Stack User

DevStack must run as a dedicated non-root user with passwordless sudo:

```bash
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo su - stack
```

Your prompt will now show `stack@ip-xxx:~$` — confirm you are the `stack` user before continuing.

### Step 3.3 — Clone DevStack

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
```

### Step 3.4 — Find Your Network Interface Name

```bash
ip addr show
```

Look for the interface that has your EC2 private IP. It is typically `ens5` on Nitro-based instances or `eth0` on older types. Note the exact name.

### Step 3.5 — Create `local.conf`

Replace `ens5` with your actual interface name from the step above:

```bash
cat > local.conf <<'EOF'
[[local|localrc]]
ADMIN_PASSWORD=secret123
DATABASE_PASSWORD=secret123
RABBIT_PASSWORD=secret123
SERVICE_PASSWORD=secret123
FLOATING_RANGE=192.168.1.224/27
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=ens5          # <-- CHANGE THIS to your interface (eth0, ens5, etc.)
LOGFILE=/opt/stack/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=False
IMAGE_URLS="http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
GIT_BASE=https://github.com
EOF
```

### Step 3.6 — Run the Installer

```bash
sudo mkdir -p /opt/stack/logs && sudo chown -R stack:stack /opt/stack/logs
./stack.sh
```

> **⏳ This takes 20–45 minutes. Do NOT close PuTTY.**
> When finished you will see:
> `"This is your host IP address: <ip>"` and a DevStack component timing table.
>
> You can monitor progress in a second PuTTY tab with:
> ```bash
> tail -f /opt/stack/logs/stack.sh.log
> ```

---

## Phase 4 — Identity & Resource Setup

### Step 4.1 — Load Admin Credentials

```bash
source /opt/stack/openrc admin admin
```

### Step 4.2 — Create Project and User

```bash
openstack project create --description "Cloud Lab" --enable CloudLab-Project

openstack user create --project CloudLab-Project --password "Student@123" \
  --email student01@lab.local --enable student01

openstack role add --project CloudLab-Project --user student01 member
```

### Step 4.3 — Upload Ubuntu Image (Glance)

```bash
cd /tmp
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

openstack image create \
  --container-format bare \
  --disk-format qcow2 \
  --file /tmp/jammy-server-cloudimg-amd64.img \
  --public --min-ram 512 --min-disk 8 \
  "Ubuntu 22.04 LTS"
```

### Step 4.4 — Create a Flavor

```bash
openstack flavor create --vcpus 1 --ram 1024 --disk 10 --public lab.small
```

---

## Phase 5 — Configure Networking (Neutron)

### Step 5.1 — Create Private Network

```bash
openstack network create private-net

openstack subnet create \
  --network private-net \
  --subnet-range 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  --dns-nameserver 8.8.8.8 \
  --ip-version 4 \
  private-subnet
```

### Step 5.2 — Create and Configure Router

```bash
# Switch back to admin to set the external gateway
source /opt/stack/openrc admin admin

openstack router create lab-router
openstack router set --external-gateway public lab-router
openstack router add subnet lab-router private-subnet
```

### Step 5.3 — Create Security Group (as student01)

```bash
source /opt/stack/openrc student01 Student@123

openstack security group create lab-sg
openstack security group rule create --protocol tcp --dst-port 22 \
  --remote-ip 0.0.0.0/0 lab-sg
openstack security group rule create --protocol icmp \
  --remote-ip 0.0.0.0/0 lab-sg
```

---

## Phase 6 — Launch the Instance (Nova)

### Step 6.1 — Create SSH Keypair

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
openstack keypair create lab-keypair > ~/.ssh/lab-keypair.pem
chmod 400 ~/.ssh/lab-keypair.pem
```

### Step 6.2 — Boot the VM

```bash
NETWORK_ID=$(openstack network show private-net -f value -c id)

openstack server create \
  --flavor lab.small \
  --image "Ubuntu 22.04 LTS" \
  --network $NETWORK_ID \
  --security-group lab-sg \
  --key-name lab-keypair \
  --wait \
  my-first-instance
```

### Step 6.3 — Assign Floating IP

```bash
# Switch to admin to allocate the floating IP
source /opt/stack/openrc admin admin

openstack floating ip create public
# Copy the 'floating_ip_address' value from the output, e.g. 192.168.1.105

openstack server add floating ip my-first-instance 192.168.1.105
```

### Step 6.4 — Verify

```bash
openstack server list
openstack console log show my-first-instance
```

The server list should show **ACTIVE** status. The console log confirms the VM booted correctly.

---

## Phase 7 — Access the Horizon Dashboard

Open a browser and go to:
```
http://<your-ec2-public-ip>/dashboard
```

| Field | Value |
|---|---|
| Domain | Default |
| Username | admin |
| Password | secret123 |

> **Can't reach Horizon?** Confirm port 80 is open in your EC2 Security Group (Phase 1, Step 6). Horizon is one of the last services started — wait 5 minutes after `stack.sh` completes before trying.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `stack.sh` fails midway | Re-run `./stack.sh` — DevStack is idempotent. If it fails again: `./unstack.sh` then `./stack.sh`. |
| PuTTY disconnects during install | Set Connection → Keepalives → 60s before connecting. Reconnect and run: `tail -f /opt/stack/logs/stack.sh.log` |
| Horizon shows 503 / not found | Wait 5 minutes after install completes — Apache needs time to start. |
| "No valid host found" on server create | Nested virt may be unavailable. Try with the cirros image (lighter) or add `--property hw:cpu_mode=none` to the flavor. |
| Floating IP not reachable | Confirm ICMP and TCP 22 rules exist: `openstack security group rule list lab-sg` |
| Wrong `FLAT_INTERFACE` | Run `ip addr show`, get correct name, edit `local.conf`, then `./unstack.sh && ./stack.sh` |

---

## ⚠ Cleanup — Terminate to Avoid Charges

> A **t3.large costs ~$0.06/hour**. Leaving it running overnight = ~$0.50. A full week = ~$10.
>
> After completing the lab:
> **EC2 → Instances → select instance → Instance State → Terminate**
>
> Termination is permanent. If you want to pause instead, use **Stop** (you still pay for the EBS volume, but not compute).

**Estimated cost for a 3-hour lab session on t3.large: ~$0.18 USD.**
