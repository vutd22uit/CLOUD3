# OpenStack Deployment utilizing Kolla-Ansible on Ubuntu 24.04.2

This guide provides complete instructions for setting up OpenStack using Kolla-Ansible with properly configured network interfaces. It addresses the Layer 2 nature of bridges and ensures correct connectivity between internal and external networks.

## 📋 Table of Contents
0. [Generate use and assign to groups]
1. [Understanding Network Architecture](#understanding-network-architecture)
2. [Prerequisites](#prerequisites)
3. [Network Configuration](#network-configuration)
4. [Testing Network Connectivity](#testing-network-connectivity)
5. [Kolla Installation and Setup](#kolla-installation-and-setup)
6. [Kolla Deployment](#kolla-deployment)
7. [Verification and Troubleshooting](#verification-and-troubleshooting)
8. [Making Configuration Persistent](#making-configuration-persistent)

---
## Generate use and assign to groups

This guide outlines the creation and configuration of a `deployer` user with all necessary permissions and groups required to deploy OpenStack using Kolla-Ansible.

## Step 0: Set Hostname to `aio`

Before creating users, rename your host:

```bash
sudo hostnamectl set-hostname aio
```

- Verify the hostname

```bash
hostname
ip a
#--> Example we use NIC `wlp4s0' and IP: 192.168.1.100
```
- Revise the ip for hots aio
```bash
sudo nano /etc/hosts
# add this line
127.0.0.1   localhost
192.168.1.100 aio

## Step 1: Create the `deployer` User

```bash
sudo useradd -m -s /bin/bash deployer
sudo passwd deployer
```

> Set a password for the new user when prompted.

---

## Step 2: Add `deployer` to Required Groups

Kolla-Ansible requires `sudo`, `docker`, and optionally a `kolla` group:

```bash
# add groups
sudo groupadd sudo
sudo groupadd docker
sudo groupadd kolla
# Assign deployer to groups
sudo usermod -aG sudo deployer
sudo usermod -aG docker deployer
sudo usermod -aG kolla deployer
```
Verify group memberships:

```bash
groups deployer
```

Expected output should include: `deployer sudo docker kolla`

---

## Step 3: Enable Passwordless Sudo (Recommended)

> This step avoids password prompts during automation:

```bash
echo "deployer ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/deployer
sudo chmod 0440 /etc/sudoers.d/deployer
```

---

## Step 4: Switch to the `deployer` User

```bash
su - deployer
```

---

## Step 5: Test Docker Access

> Ensure that the `deployer` user can run Docker without `sudo`:
```bash
sudo ls /etc/
# Reboot and login with deployer to deploy openstack
sudo reboot
```

## 🔄 Understanding Network Architecture

### Bridge Layer 2 vs. virtual NICs Layer 3

A critical concept to understand is that **bridges operate at Layer 2** (Data Link Layer) of the OSI model:

- Bridges forward frames based on MAC addresses, not IP addresses
- They don't inherently perform routing functions (Layer 3)
- When you assign an IP to a bridge, it's primarily for management and gateway purposes
- You cannot directly ping external addresses from a bridge interface (`ping -I br-exnat 8.8.8.8` will fail)
- You can "plugin" virtual NICs to Bridges


### Network Topology

Our setup will use:
- **Internal Network**: Virtual bridge (e.g.,'br-exnat') <--> Wifi Ethernet NIC 'wlp4s0' (192.168.1.0/24)
- **External Network**: Wifi Ethernet NIC (e.g., wlp4s0) → External network (192.168.1.0/24)
- **Connectivity**: NAT + IP Forwarding for 'br-exnat' <--> wlp4s0.
- **External Network NIC for Netron**: veth-ext<--> br-exnat <--> wlp4s0 <--> External network
- **Check your device names**:
---

## 📋 Prerequisites

- Ubuntu/Debian Linux system (22.04, 24.04)
- Administrative (sudo) privileges
- 2 Physical network interfaces (WiFi or Ethernet), we may use  1 Physical network interface + 1 Virtual bridge
- At least 16GB RAM and 4 CPU cores for Kolla
- 100GB+ free disk space
---

## 🔌 Network Configuration

### Step 1: Identify Your Interfaces

```bash
# Check available interfaces
ip addr show
```

Note your physical interface name (e.g., `wlp4s0` for WiFi or `enp2s0` for Ethernet).

### Step 2: Install Required Tools

```bash
# Install needed packages, do not upgrage ubuntu: 22.04 to 24.04
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install -y bridge-utils net-tools iptables-persistent network-manager

# Stop and disable systemd-networkd
sudo systemctl stop systemd-networkd
sudo systemctl disable systemd-networkd

# Enable and start NetworkManager
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager
```

### Step 3. Prepare Interfaces and Network Topology

1. Create a profile and manual set IP for `br-exnat` (internal)  network
2. Connect `wlp4s0` to external network (may vary depend on APs)
3. Set NAT and Routing for `br-exnat` <--> `wlp4s0 `  communications
4. Create veth  pairs  and bind into 'br-exnat`
5. Save and set  persistence on reboot

### 1.  **Create a new connection profile for wlp4s0**

Notes: Disconnect (forget) all NICs before do these following tasks

```bash
#### 1. Delete any existing conflicting connection named 'wlp4s0' (optional but clean)
sudo nmcli connection delete wlp4s0 2>/dev/null

#### 2. Add a new Wi-Fi connection profile (SSID and password required)
#--> Replace YOUR_SSID and YOUR_PASSWORD accordingly
sudo nmcli dev wifi connect YOUR_SSID password YOUR_PASSWORD ifname wlp4s0 name wlp4s0

#--> For ethernet wire use this for NIC `enp2s0`, replace to your NIC name.
sudo nmcli connection add type ethernet ifname enp2s0 con-name enp2s0

### 3. Bring the connection up wlp4s0 (or enp2s0) 
sudo nmcli connection up wlp4s0

# 4. Verify the connection
nmcli device status
# → Look for "wlp4s0 wifi connected" with correct SSID
# → or "enp2s0" is connected 
```

### 2: Create Bridge Interface

```bash
# Create bridge interface
sudo nmcli connection add type bridge autoconnect yes con-name br-exnat ifname br-exnat

# Set IP for bridge
sudo nmcli connection modify br-exnat ipv4.addresses 10.10.10.1/24 ipv4.method manual

# Enable the bridge
sudo nmcli connection up br-exnat

# Enable routing to via `br-exnat`
sudo ip route add 10.10.10.0/24 dev br-exnat

# Verify bridge creation
ip addr show br-exnat
#--> the results should show: `NO-CARRIER,BROADCAST,MULTICAST,UP; inet 10.10.10.1/24 brd 10.10.10.255`

ip route | grep 10.10.10.0
#-->  10.10.10.0/24 dev br-exnat scope link linkdown 
#--> 10.10.10.0/24 dev br-exnat proto kernel scope link src 10.10.10.1 metric 425 linkdown 
```
### 3. Enable IP forwarding rules

```bash
# 1. Edit the sysctl configuration file to enable IP forwarding:
sudo nano /etc/sysctl.conf

# 2. Add or ensure this line is present and uncommented and save:
net.ipv4.ip_forward=1

# 3. Apply the changes immediately:
sudo sysctl -p

# ✅ You should see the line:
# net.ipv4.ip_forward = 1
```

### 4: Configure NAT for `br-exnat`  <---> `wlp4s0 `

```bash
#  Optional: Clear existing NAT rules (be cautious in production, you may clean manually, not all)
sudo iptables -t nat -F POSTROUTING

# Add NAT (MASQUERADE) rule
# Replace 'wlp4s0' if your external interface is different
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o wlp4s0 -j MASQUERADE

# Allow forwarding
sudo iptables -P FORWARD ACCEPT
sudo iptables -F FORWARD

#  Allow traffic between br-exnat (internal bridge) and wlp4s0 (external)
sudo iptables -A FORWARD -i br-exnat -o wlp4s0 -j ACCEPT
sudo iptables -A FORWARD -i wlp4s0 -o br-exnat -m state --state RELATED,ESTABLISHED -j ACCEPT

#  Allow ICMP (ping)
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT
sudo iptables -A OUTPUT -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A FORWARD -p icmp -j ACCEPT

#  Save iptables rules persistently
sudo apt install -y iptables-persistent
sudo netfilter-persistent save

# (Optional) Backup rules manually too
sudo mkdir -p /etc/iptables
sudo iptables-save | sudo tee /etc/iptables/rules.v4 > /dev/null

#  Verify NAT and FORWARD chains
sudo iptables -t nat -L -n -v
# --> `0     0 MASQUERADE  0    --  *      wlp4s0  10.10.10.0/24        0.0.0.0/0`

sudo iptables -L FORWARD -n -v
# ➤ ` 0     0 ACCEPT     0    --  br-exnat wlp4s0  0.0.0.0/0            0.0.0.0/0'
# ➤  `0     0 ACCEPT     0    --  wlp4s0 br-exnat  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED`

ip route | grep 10.10.10.0
# ➤  `10.10.10.0/24 dev br-exnat proto kernel scope link src 10.10.10.1  ...`
```
### 5. Verify the static IP for `br-exnat'
```bash
#  0. List all NetworkManager connections and locate the one for `br-exnat`
nmcli connection show

#  Look for the connection associated with `br-exnat` and note its UUID or name
# Example:
# NAME                UUID                                  TYPE      DEVICE
# br-exnat              68b1e6c4-c3a6-44b7-bf35-434455be4608  bridge    br-exnat

# Check current IP assignment of br-exnat
ip a show br-exnat
ip a show wlp4s0
# Desired output should include:
# inet 10.10.10.1/24 brd 10.10.10.255 scope global br-exnat
# inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp4s0

# If `br-exnat` does NOT have a static IP assigned, use this command:
# Replace the UUID below with the correct one for your system
sudo nmcli connection modify 68b1e6c4-c3a6-44b7-bf35-434455be4608 ipv4.addresses 10.10.10.1/24 ipv4.method manual

# Bring the connection up to apply the changes
sudo nmcli connection up 68b1e6c4-c3a6-44b7-bf35-434455be4608
#  If `wlp4s0`  change the IP, you should revise the infot in 'etc/hosts'
```
### 6. Generate a Virtual NICs for `neutron_external_interface`

```
[ VM (Floating IP) ]
         │
     (Neutron)
         │
   ┌───────────────┐
   │ br-ex       │  ← (created by Kolla/Neutron agent)
   └───────────────┘
        │
   [ veth-peer ]   ← set in globals.yml: neutron_external_interface
        │
   [ veth-ext ]    ← host-end of veth pair, connects to NAT
        │
     [br-exnat]     
     (iptables MASQUERADE)
        │
   [ wlp4s0 ]      ← real external NIC (Wi-Fi) (for Interal/external interfaces)
```
We will do this in step 7 and set persistence for reboot
---
### 7. Persistent save for reboot

# 1️⃣ Create a persistent network script for interface setup 
Notes: We will add other bridges here later

```bash
sudo nano /usr/local/bin/setup-br-exnat.sh
```
Add the following content to the file (must edit `wlp4s0` to your NIC name):
```bash
#!/bin/bash

set -euo pipefail

# === Ensure br-exnat Linux bridge exists ===
if ! ip link show br-exnat &>/dev/null; then
    ip link add br-exnat type bridge
    echo "[local-setup] Created br-exnat bridge"
fi

# Assign static IP if not present
if ! ip addr show dev br-exnat | grep -q "10.10.10.1"; then
    ip addr add 10.10.10.1/24 dev br-exnat
    echo "[local-setup] Assigned IP to br-exnat"
fi

# Enable IP forwarding
if [[ "$(cat /proc/sys/net/ipv4/ip_forward)" -ne 1 ]]; then
    sysctl -w net.ipv4.ip_forward=1
    echo "[local-setup] Enabled IP forwarding"
fi

# Create veth pair if it doesn't exist
if ! ip link show veth-ext &>/dev/null; then
    ip link add name veth-ext type veth peer name veth-peer
    echo "[local-setup] Created veth pair"
fi

# Bring up both veth interfaces
ip link set veth-ext up
ip link set veth-peer up

# Attach veth-ext to br-exnat if not already
if ! bridge link show | grep -q "veth-ext"; then
    ip link set veth-ext master br-exnat
    echo "[local-setup] Attached veth-ext to br-exnat"
fi

# Final bring-up to ensure bridge gets carrier after binding
ip link set br-exnat up

# Clean previous NAT and FORWARD rules (ignore errors if not exist)
iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o wlp4s0 -j MASQUERADE 2>/dev/null || true
iptables -D FORWARD -i br-exnat -o wlp4s0 -j ACCEPT 2>/dev/null || true
iptables -D FORWARD -i wlp4s0 -o br-exnat -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || true

# Reapply NAT and FORWARD rules
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o wlp4s0 -j MASQUERADE
iptables -A FORWARD -i br-exnat -o wlp4s0 -j ACCEPT
iptables -A FORWARD -i wlp4s0 -o br-exnat -m state --state RELATED,ESTABLISHED -j ACCEPT

# Ensure route to 10.10.10.0/24 via br-exnat exists
if ! ip route show 10.10.10.0/24 | grep -q "br-exnat"; then
    ip route add 10.10.10.0/24 dev br-exnat
    echo "[local-setup] Added route for 10.10.10.0/24 via br-exnat"
fi

echo "[local-setup] Completed bridge, veth, NAT, and forwarding setup"

```
✅ This ensures that:
- `br-exnat` gets a static IP on boot
- `veth-ext` is created and ready for Neutron traffic
- NAT/forwarding rules restore connectivity to external Wi-Fi NIC (`wlp4s0`) or wired ethernet NIC
- automatic route for `br-exnat`
You can now proceed to Kolla deployment using `veth-peer` as the `neutron_external_interface`.

# 2️⃣ Save and exit the file and make the script executable
```bash
sudo chmod +x /usr/local/bin/setup-brs.sh
sudo nano /etc/systemd/system/setup-brs.service
```
- Add these line to the file and Save
```ini
[Unit]
Description=Custom br-exnat NAT Bridge Setup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-brs.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

- Enable and test it
```bash
sudo systemctl daemon-reexec    # Only needed if you updated systemd itself
sudo systemctl daemon-reload     # Always do this after editing unit files
sudo systemctl enable setup-brs.service
sudo systemctl start setup-brs.service
```

# Test manually
```bash
sudo bash -x /usr/local/bin/setup-brs.sh
#  Testing reboot behavior
sudo systemctl status setup-brs.service
```
### 8. Reboot and Verify to Ensure IP, NAT, and IP Forwarding

After completing the persistent network setup, reboot your machine:

```bash
sudo reboot
```
Once the system comes back up, verify the following:
1. Check that `br-exnat` still has its static IP:

```bash
ip a show br-exnat
# ➤ Should include: inet 10.10.10.1/24
```

2. Confirm IP forwarding is enabled:

```bash
cat /proc/sys/net/ipv4/ip_forward
# ➤ Should return: 1
```

3. Check that NAT rules are still in place:

```bash
sudo iptables -t nat -L POSTROUTING -n -v
sudo iptables -L FORWARD -n -v
# ➤ Should include rules for traffic between br-exnat and wlp4s0

```
4. Automatic rout for br-exnat
```bash
ip route | grep 10.10.10.0
# ➤ 10.10.10.0/24 dev br-exnat proto kernel scope link src 10.10.10.1 metric 425 
```

##  Setup Kolla-Ansible

### ✅ Step 1. Install Dependencies

```bash
# 1. Update system packages
sudo apt update && sudo apt upgrade -y

# 2. Install essential development and Python dependencies
sudo apt install -y \
  python3-pip python3-dev \
  libssl-dev libffi-dev \
  gcc build-essential \
  libxml2-dev libxslt1-dev \
  zlib1g-dev libdbus-1-dev libglib2.0-dev \
  python3-venv git

# 3. Check or install Python 3.10 for  Kolla (not python 3.12)
- Check python version
python3 --version
#--> The return should be Python 3.10.*. If not you have to install Python 3.10
- Install Python 3.10 
# a. Install Python 3.10 and create a virtual environment for Kolla
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.10 python3.10-venv python3.10-dev
# b. Verify
which python3.10
#--> `/usr/bin/python3.10`

# 4. Create and activate the virtual environment
sudo rm -rf ~/kolla-venv
python3.10 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate

# 5. Setup setuptools which compatibility with pbr and Python 3.10+
pip install "setuptools<69"

# 6. Upgrade pip inside the venv
pip install --upgrade pip

```

### ✅ Step 2. Cleanup Old Deployment (Inventories, Containers, and Dependencies)

> If you're working on a reused or older PC that may have prior Kolla or Docker deployments, it's important to fully clean up before starting fresh.

---

####1. Clean Docker Containers, Volumes, and Images

```bash
# Stop all containers
sudo docker ps -aq | xargs -r sudo docker stop

# Remove all containers
sudo docker ps -aq | xargs -r sudo docker rm -f

# Remove all images
sudo docker images -aq | xargs -r sudo docker rmi -f

# Remove all volumes
sudo docker volume ls -q | xargs -r sudo docker volume rm

# Clean dangling networks
sudo docker network prune -f
```
---

####2. Remove Kolla Configuration Directories

```bash
sudo rm -rf /etc/kolla
sudo rm -rf /opt/kolla

# Delete Only Kolla-related Docker Images
docker images | grep quay.io | awk '{print $3}' | xargs -r sudo docker rmi -f
sudo docker container prune -f
sudo docker image prune -f
sudo docker volume prune -f
```
---

####3. Destroy Existing Kolla Deployments- **Identify Existing Inventory Files**

> Kolla-Ansible uses inventory files (`multinode`, `all-in-one`) to manage deployments.

```bash
find ~ -type f -name "multinode" -o -name "all-in-one"
```

Example output:
```ini
/home/deployer/kolla-ansible/ansible/inventory/all-in-one
/home/deployer/kolla-ansible/ansible/inventory/multinode
/home/deployer/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one
/home/deployer/kolla-venv/share/kolla-ansible/ansible/inventory/multinode
Notes: Donot delete the new install and delete all other
/home/deployer/kolla-ansible/ansible/inventory/all-in-one
/home/deployer/kolla-ansible/ansible/inventory/multinode

- **(Optional) Destroy all old deployment**

```bash
sudo kolla-ansible destroy -i /home/deployer/kolla-ansible/ansible/inventory/all-in-one --yes-i-really-really-mean-it

kolla-ansible destroy -i /home/deployer/kolla-ansible/ansible/inventory/all-in-one --yes-i-really-really-mean-it

# Destroy multinode deployment
sudo kolla-ansible destroy -i /home/deployer/kolla-ansible/ansible/inventory/multinode --yes-i-really-really-mean-it
kolla-ansible destroy -i /home/deployer/kolla-ansible/ansible/inventory/multinode --yes-i-really-really-mean-it
```

Replace `/path/to/` with your actual inventory file path.

####4. Optional: Clear Residual Package Cache

```bash
sudo apt autoremove --purge -y
sudo apt clean
```

### ✅ Step 3. Setup Kolla-Ansible, Docker CE, and Ansible (Yoga)

> We use OpenStack Yoga because quay.io only provides container images up to the `yoga` tag.

#### 1. Install kolla-ansible

```bash
source ~/kolla-venv/bin/activate
sudo rm -rf kolla-ansible

git clone -b yoga-eol https://opendev.org/openstack/kolla-ansible.git

cd kolla-ansible

git checkout -b yoga-eol-local

pip install -r requirements.txt

pip install .

#Verify
which kolla-ansible
#--> `/home/deployer/kolla-venv/bin/kolla-ansible`
```
#### 2. Install Docker CE (clean and compatible)
# This step is Optional to download images from `download.docker.com`

```bash
#a. Core Dependencies
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

#b. Add Docker’s GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

#c. Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#d. Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin

#e. Enable Docker
sudo systemctl enable --now docker

#f. (Optional) If you plan to not download from download.docker.com again
sudo sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/docker.list

#g.  Verify
sudo cat /etc/apt/sources.list.d/docker.list
# --> Result should show: #deb ...

#h. Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker
source ~/kolla-venv/bin/activate

#h. Checking
groups

# --> Should include: docker sudo deployer kolla

docker --version
# --> Should print: Docker version ...
```
---

## Setup Ansible and Verify Default Paths 

### ✅ Step 1. Install Ansible compatible with Yoga

```bash
# ansible-core 2.12.10 = matches Yoga's tested version
# ansible>=5,<6      = provides the meta-package with community collections

pip install "ansible-core==2.12.10" "ansible>=5,<6"

# Optional: Check versions
ansible --version
docker --version
# Notes:
# - Up to now, we do NOT have ~/.ansible/collections/ installed.
# - Kolla-Ansible will expect required collections there.
# - We will manually install the necessary collections later
```

### ✅ Step 2: Set Owner and Verify Default Paths for Ansible, Kolla-Ansible, and OpenStack

This step ensures you understand where Kolla-Ansible roles expect files to be written and verifies **both path references and ownership**, especially on reused or restricted systems.

####  1. Verify Python Virtual Environment and Kolla-Ansible Role Paths

```bash
# Activate your virtual environment
source ~/kolla-venv/bin/activate

#1. Get the virtual environment path
echo "$VIRTUAL_ENV"
# ➤ Output should be something like: /home/deployer/kolla-venv

#2. Locate the Kolla-Ansible role defaults directory
ROLE_DIR=$(find "$VIRTUAL_ENV" -type d -path "*/share/kolla-ansible/ansible/roles" 2>/dev/null)

#3. Confirm the roles directory
echo "Roles dir: $ROLE_DIR"

# ➤ Output should look like: /home/deployer/kolla-venv/share/kolla-ansible/ansible/roles
# ➤  If the return = "", you forgot install ` kolla-ansible": 'pip install .
#4. Verify
ls /home/deployer/kolla-venv/share/kolla-ansible/ansible/roles
# ➤ Output should show roles for kolla-ansible

#5. Extract default path-related variables from role defaults
grep -RHE 'dest:|path:|config_dir:' \
  "$ROLE_DIR"/*/defaults/main.yml \
  | sed -E 's/^[^:]+:[[:space:]]*//' \
  | sort -u
```
- **What These Paths Tell You**?

Kolla-Ansible role defaults often reference:

- Temporary files (e.g., `/tmp/kolla_*`)
- Relative configuration paths (e.g., `openssl.cnf`)
- Optional external service endpoints
- Templated values (e.g., using `{{ }}` syntax for host/project-specific output)

#### 2. Generate and check Runtime-Critical Directories (Used During Actual Deployment)

-  **Generate and  check common Kolla/OpenStack runtime directories and ensure your user can write to them**
```bash
sudo rm -rf etc/kolla /opt/kolla  /var/lib/docker
```
```bash
for path in /etc/kolla /opt/kolla /var/lib/docker; do
  echo "Checking permissions on $path"
  sudo mkdir -p "$path"
  sudo chown -R $USER:$USER "$path"
  ls -ld "$path"
done
```
-  **These are  not defined in Ansible role defaults  but are used by**:
- Kolla configuration management
- Docker volume mounts
- Persistent service state (e.g., MariaDB data, RabbitMQ queues)


### ✅ Step 3: Copy and Configure Inventory and Defaults for Kolla-Ansible

Kolla-Ansible expects its main configuration and inventory files under `/etc/kolla`. This step prepares that directory and populates it with example files.

#### 1️⃣ Create and Take Ownership of `/etc/kolla`

```bash
source ~/kolla-venv/bin/activate

# Create required directories
sudo mkdir -p /etc/kolla
sudo mkdir -p /etc/kolla/ansible/inventory

# Give current user ownership (for editing)
sudo chown -R "$USER":"$USER" /etc/kolla
```
#### 2️⃣ Locate Example Files in Your Environment

- Find where the example configs are installed inside your venv

```bash
find ~/kolla-venv -type d -name "etc_examples"

# Example output:
# ➤ `/home/deployer/kolla-venv/share/kolla-ansible/etc_examples`
```
#### 3️⃣ Copy Example Configs and Inventories

- Copy default globals and passwords config files

```bash
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
```

- Copy sample inventory files (all-in-one, multinode)

```bash
cp -r ~/kolla-venv/share/kolla-ansible/ansible/inventory/* /etc/kolla/ansible/inventory/
```
#### 4️⃣ Verify Copy Success

- Verify base configuration files

```bash
ls /etc/kolla/
# → globals.yml  passwords.yml  ansible/

# Verify inventories
ls /etc/kolla/ansible/inventory/
# → all-in-one  multinode  registry_credentials.yml
```
### ✅ Step 4: Configure Ansible Inventory and ansible.cfg (Scoped for Kolla-Ansible)

To ensure a consistent environment for deploying Kolla-Ansible, it's best practice to define an Ansible configuration file **local to the inventory**. This avoids system-wide conflicts and ensures Ansible uses the correct Python interpreter and role paths.

#### 1️⃣ Create Inventory-Scoped Ansible Config Directory

```bash
# Activate your virtualenv (if not already)
source ~/kolla-venv/bin/activate

# Ensure the inventory directory exists
mkdir -p /etc/kolla/ansible/inventory
```
#### 2️⃣ Determine Python Interpreter Path

```bash
which python
# ➤ Output: /home/deployer/kolla-venv/bin/python
```
#### 3️⃣ Create `ansible.cfg` Inside the Inventory Directory

```bash

mkdir -p /home/deployer/kolla-venv/var/log
nano /etc/kolla/ansible/inventory/ansible.cfg
```

Add the following content (adjust paths as needed for your environment, no first blank line):

```ini
[defaults]
gather_subset = hardware,network
interpreter_python = /home/deployer/kolla-venv/bin/python
host_key_checking = False
timeout = 30
retry_files_enabled = False
pipelining = True

roles_path = /home/deployer/kolla-venv/share/kolla-ansible/ansible/roles
collections_path = /home/deployer/.ansible/collections/ansible_collections:/usr/share/ansible/collections/ansible_collections

forks = 10
inventory = /etc/kolla/ansible/inventory/all-in-one
log_path = /home/deployer/kolla-venv/var/log/ansible.log

[privilege_escalation]
become = True
become_method = sudo

[ssh_connection]
control_path = %(directory)s/%%h-%%r

```

#### 4️⃣ Export and Verify Ansible Configuration

```bash
cd /etc/kolla/ansible/inventory/
source ~/kolla-venv/bin/activate

# Clear any potentially conflicting environment variables
unset ANSIBLE_CONFIG ANSIBLE_COLLECTIONS_PATH ANSIBLE_COLLECTIONS_PATHS ANSIBLE_ROLES_PATH ANSIBLE_GATHER_SUBSET ANSIBLE_PYTHON_INTERPRETER

# Export the scoped config
export ANSIBLE_CONFIG="$PWD/ansible.cfg"

# Check interpreter and path settings
ansible-config dump | grep -E 'GATHER_SUBSET|INTERPRETER|ROLES_PATH|COLLECTIONS_PATH'
```
You should see output like:
```text
INTERPRETER_PYTHON(...) = /home/deployer/kolla-venv/bin/python

```
✅ Why not use `/etc/ansible/ansible.cfg`?
Because that’s a **system-wide config** shared by other tools (e.g., OpenStack-Ansible, Galaxy, automation scripts). Keeping `ansible.cfg` local to the Kolla inventory ensures:
- No conflicts
- Portable and isolated setup
- Reproducible behavior in multi-user environments
- This setup guarantees that `ansible-playbook` will behave consistently with your Kolla virtualenv and role layout.

### ✅ Step 5: Prepare Container Volumes for Logs and Config Injection (Kolla-Ansible Safe Setup)

This step ensures necessary directories exist and are safely owned before Kolla-Ansible deployment begins. These folders are used by Kolla for **logs**, **custom configs**, and **container runtime state**.

### 1️⃣ Create Host Directories for Logs and Custom Config Overrides

```bash
source ~/kolla-venv/bin/activate

# These are safe to pre-create
sudo rm -rf  /etc/kolla/config
sudo mkdir -p /etc/kolla/config

# Optional: only if you plan to inject config templates (e.g., config.json)
sudo mkdir -p /etc/kolla/config_files

# Set ownership for deployer
sudo chown -R "$USER":"$USER" /etc/kolla
sudo chown -R "$USER":"$USER" /etc/kolla/config /etc/kolla/config_files

# Verify permissions
ls -ld  /etc/kolla /etc/kolla/config /etc/kolla/config_files
```

### ✅ Summary
| `/etc/kolla/config` | Your override configs (e.g., nova.conf) | ✅ Yes |
| `/etc/kolla/config_files` | For injecting `config.json` templates | ✅ Optional |
⚠️ **Avoid conflicts** with other OpenStack tools by keeping these paths scoped to Kolla.


## Setup and download openstack container images

### Step 1: Configure globals.yml

Notes: You may set any existing tags for container images.

####1. Search for correct tags

```bash
sudo apt  install jq

curl -s 'https://quay.io/api/v1/repository/openstack.kolla/ubuntu-source-openvswitch-base/tag/?limit=100' \
  | jq -r '.tags[].name'

```
#--> The result should return some thing low below, you con select one to set for globals.yml:
yoga
xena
wallaby
victoria
ussuri
train

####2. Edit `globals.yml` for pull container images

```bash
sudo nano /etc/kolla/globals.yml
```
# Add/modify these settings (for ubuntu 22.04 or 24.04)
```yaml
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "yoga"
openstack_tag: "yoga"
openstack_tag_suffix: ""
docker_image_name_prefix: "ubuntu-source-"
node_custom_config: "{{ node_config }}/config"

# Network settings
kolla_internal_vip_address: "192.168.1.254"
kolla_internal_fqdn: "{{ kolla_internal_vip_address }}"

kolla_external_vip_address: "192.168.1.254"
(you may set different internal later)

# Docker options
docker_registry: "quay.io"
docker_namespace: "openstack.kolla"
kolla_container_engine: "docker"

# Neutron - Networking Options
##The bridge or device for your management/data plane
network_interface: "wlp4s0"

##The host interface to bind your external VIP on
kolla_external_vip_interface: "wlp4s0"
api_interface: "wlp4s0"
network_address_family: "ipv4"

##The physical NIC (or bridge) that Neutron uses for provider networks
## Host-level physical NIC
neutron_external_interface: "veth-peer"
neutron_plugin_agent: "openvswitch"

# Healthcheck options
enable_container_healthchecks: "yes"

# Firewall options
disable_firewall: "true"

# Region options
openstack_region_name: "RegionOne"

# OpenStack options
enable_openstack_core:  "yes"
enable_glance: "{{ enable_openstack_core | bool }}"
enable_haproxy: "yes"
enable_keepalived: "{{ enable_haproxy | bool }}"
enable_keystone: "{{ enable_openstack_core | bool }}"
enable_mariadb: "yes"
enable_memcached: "yes"
enable_neutron: "{{ enable_openstack_core | bool }}"
enable_nova: "{{ enable_openstack_core | bool }}"
enable_rabbitmq: "{{ 'yes' if om_rpc_transport == 'rabbit' or om_notify_transport == 'rabbit' else 'no' }}"
enable_horizon: "{{ enable_openstack_core | bool }}"
enable_openvswitch: "{{ enable_neutron | bool and neutron_plugin_agent != 'linuxbridge' }}"

Save and exit (Ctrl+O, Enter, Ctrl+X).

#Option
You may need save a backup file
sudo cp /etc/kolla/globals.yml /etc/kolla/globals.yml.1
```
---

### ✅ Step 2: Kolla Prepare for Deployment

#### 🔹 1. Bootstrap the Servers

This step prepares your target host (in this case, the same machine) for Kolla deployment. It configures necessary users, Docker, Python interpreter, and other base components.

- **Download colections**

```bash
# Navigate to your inventory directory
cd /etc/kolla/ansible/inventory/

# Activate Python virtual environment
source ~/kolla-venv/bin/activate

# Optional but recommended: disable AppArmor for container compatibility
sudo systemctl stop apparmor
sudo systemctl disable apparmor
sudo ln -vsf /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable

# Ensure firewall utility is available
sudo apt install ufw -y
pip install docker
# Clean up potential environment variable conflicts
unset ANSIBLE_GATHER_SUBSET ANSIBLE_PYTHON_INTERPRETER ANSIBLE_COLLECTIONS_PATH

# Point to the local ansible.cfg for Kolla
export ANSIBLE_CONFIG="$PWD/ansible.cfg"

# Verify interpreter and other settings
ansible-config dump | grep -E 'GATHER_SUBSET|INTERPRETER|ROLES_PATH|COLLECTIONS_PATH'
# → INTERPRETER_PYTHON should point to: /home/deployer/kolla-venv/bin/python

# Dowload Ansible Galaxy collection (yoga) 
sudo rm -rf  ~/.ansible/collections/ansible_collections/openstack
mkdir -p ~/.ansible/collections/ansible_collections/openstack
cd ~/.ansible/collections/ansible_collections/openstack
sudo rm -rf kolla

git clone https://opendev.org/openstack/ansible-collection-kolla.git kolla
cd kolla
git checkout tags/yoga-eol -b yoga-eol-local

##verify
ansible-galaxy collection list | grep openstack.kolla
# --> results should show: `openstack.kolla 1.0.0` 

ansible-config dump | grep -E 'GATHER_SUBSET|INTERPRETER|ROLES_PATH|COLLECTIONS_PATH'

# ---> COLLECTIONS_PATH must have /home/deployer/.ansible/collections/ansible_collections'
```

- **Bootstrap the server(s)**

- **Run bootstrap**
```bash
cd /etc/kolla/ansible/inventory
source ~/kolla-venv/bin/activate
kolla-genpwd
kolla-ansible bootstrap-servers -i /etc/kolla/ansible/inventory/all-in-one
```
- **Debugs: You may need revise docker_sdk tasks if bootstrap-servers False in Ubuntu 24.04**
```bash
### Revise docker_sdk/tasks/main.yml
nano ~/.ansible/collections/ansible_collections/openstack/kolla/roles/docker_sdk/tasks/main.yml

#revise the 'pip3' line and save
- name: Install docker SDK for python
....
executable: "{{ virtualenv is none | ternary('pip3', omit) }}"
## -->
- name: Install docker SDK for python
---
executable: "{{ virtualenv is none | ternary('/home/deployer/kolla-venv/bin/pip3', omit) }}

# Re  bootstrap-servers
kolla-ansible bootstrap-servers -i /etc/kolla/ansible/inventory/all-in-one

```

## ✅ Expected Results

- The `kolla` user will be created on the local machine.
- Docker and Python dependencies will be installed.
- SSH key exchange and Ansible setup will be validated.

### ✅ Step 3:  Precheck Kolla Preparation

- **Clean all old docker images**
```bash
source ~/kolla-venv/bin/activate
## Stop every container that was started by Kolla (they all carry the 'kolla_version' label)
docker ps -q --filter "label=kolla_version" | xargs -r docker stop

## (Options) Delete all old dicker images (if ones need)
sudo docker stop $(sudo docker ps -aq)
sudo docker rm -f $(sudo docker ps -aq)
sudo docker rmi -f $(sudo docker images -aq)
sudo docker volume rm $(sudo docker volume ls -q)
sudo docker system prune -a --volumes -f

## Stop libvirtd if one need
sudo systemctl stop libvirtd
sudo systemctl disable libvirtd
sudo pkill -f libvirtd
sudo rm -f /var/run/libvirt/libvirt-sock

## Check NICs
sudo apt install -y libdbus-1-dev libglib2.0-dev
pip3 install dbus-python
ip a

##--> The resuls should show:
##--> local NIC (10.10.10.*/24) and external NIC (192.168.1.*/24)

## Ping check (use your real IP)
ping -c 3 10.10.10.1
ping -c 3 8.8.8.8

## Detele the .254 ips in ones avalable
# Verify IP for NICs
ip a
# You may need delete some IP
sudo ip addr del 10.10.10.254/32 dev br-exnat
sudo ip addr del 192.168.1.254/32 dev wlp4s0
```
#### Revise checking tasks (only for ubuntu 24.04)
```bash
nano /home/deployer/kolla-venv/share/kolla-ansible/ansible/roles/prechecks/tasks/host_os_checks.yml
add a line below the line: `- ansible_facts.distribution_version != '24.04'
```ini
  when:
    - ansible_facts.distribution_release not in host_os_distributions[ansible_facts.distribution]
    - ansible_facts.distribution_version not in host_os_distributions[ansible_facts.distribution]
    - ansible_facts.distribution_major_version not in host_os_distributions[ansible_facts.distribution]
    - ansible_facts.distribution_version != '24.04'
```

####  Run kolla-ansible precheck
- **Check IP/hotsname for your local host **
```bash
sudo docker ps -q | xargs -r sudo docker stop
sudo ip neigh flush all
ip a show
hostname -f
#---> eg. IP Results show: 192.168.1.100
#--->  hostname show: aio
```
- **Setting IP for your local hosts**
```bash
sudo nano /etc/hosts
# Double check the line to hosts (use your host name)
192.168.1.100 aio
```
- **Run checking**
```bash
cd /etc/kolla/ansible/inventory
source ~/kolla-venv/bin/activate
kolla-ansible prechecks -i /etc/kolla/ansible/inventory/all-in-one
```
-**Debug**
You may need revise rabbitmq roles if it raise errors. If not let it as original one.
-**Revise rabbitmq roles**
```bash
nano /home/deployer/kolla-venv/share/kolla-ansible/ansible/roles/rabbitmq/tasks/precheck.yml

#Revise 1:
- name: Check if all rabbit hostnames are resolvable
....
  command: "getent {{ nss_database }} {{ hostvars[item].ansible_facts.hostname }}"

--->
- name: Check if all rabbit hostnames are resolvable
....
  shell: |
    getent {{ nss_database }} {{ hostvars[item].ansible_facts.hostname }} | grep STREAM | awk '{print $1}' | sort -u
 
#Revise 2:
  when:
     - not item.1 is match('^'+('api' | kolla_address(item.0.item))+'\\b')
     
  --->
    when: item.1 != hostvars[item.0.item]['ansible_' + api_interface]['ipv4']['address']

```

### ✅ Step 4: Generate configs
```bash
cd /etc/kolla/ansible/inventory/

# Activate Python virtual environment
source ~/kolla-venv/bin/activate
sudo docker ps -q | xargs -r sudo docker stop
kolla-ansible genconfig -i /etc/kolla/ansible/inventory/all-in-one
```
### ✅ Step 5: Pull or Build Docker Images Locally
# ✅ Step 5: Pull or Build Docker Images Locally

## 1. Pull or Build Docker Images Locally

### Option A: Pull from Official Registry

1. **Activate the virtual environment**:

   ```bash
   cd /etc/kolla/ansible/inventory/
   source ~/kolla-venv/bin/activate
   ```

2. **Verify Python interpreter settings**:
   Ensure the Python interpreter is correctly set to use your virtual environment.

   ```bash
   unset ANSIBLE_GATHER_SUBSET ANSIBLE_PYTHON_INTERPRETER ANSIBLE_COLLECTIONS_PATH
   export ANSIBLE_CONFIG="$PWD/ansible.cfg"
   
   ansible-config dump | grep -E 'GATHER_SUBSET|INTERPRETER|ROLES_PATH|COLLECTIONS_PATH'
   ```
   - You should see something like:
     ```bash
     INTERPRETER_PYTHON(/etc/kolla/ansible/inventory/ansible.cfg) = /home/deployer/kolla-venv/bin/python
     ```

3. **Pull the Docker images**:

   ```bash
   sudo systemctl restart docker
    sudo rm -rf /var/lib/docker/tmp/
    sudo mkdir /var/lib/docker/tmp
    sudo chown -R "$USER":"$USER"  /var/lib/docker/
   kolla-ansible pull -i /etc/kolla/ansible/inventory/all-in-one
   ```

4. **Check the pulled images**:

   After pulling the images, verify that they are available:

   ```bash
   sudo docker image ls
   ```

   - You should see containers with the `tag=yoga`.

---

### Option B: Build Locally (e.g., if official image is broken)

1. **Build the Docker image locally**:
 ## see more at https://static.opendev.org/docs/kolla/latest/admin/image-building.html
   If the official image is broken or unavailable, you can build it locally. This example for  kolla-toolbox.

   ```bash
kolla-build -n quay.io/openstack.kolla --threads 8 --skip-existing --base ubuntu --tag yoga kolla-toolbox
   ```

2. **Verify the build**:

   After building the image, verify that it exists:

   ```bash
   docker images | grep kolla-toolbox
   ```

   - The expected output should be:
     ```bash
     kolla/ubuntu-source-kolla-toolbox   yoga   <image_id>
     ```

---

#### Step 2 **Re-tag (if needed)**:

   If Kolla expects the `quay.io/...` format for the image, you can re-tag it:
   Usage:  docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

   ```bash
   docker tag quay.io/openstack.kolla/ubuntu-source-kolla-toolbox:yoga kolla/ubuntu-source-kolla-toolbox:yoga
   ```
   Or alternatively, modify Kolla to use the `kolla/` namespace instead (depending on your preference or setup).
  
#### `retag_images.sh`
sudo nano retag_images.sh
#### add these lines to the file and save
```ini
#!/bin/bash

# Loop through all images with the 'yoga' tag and retag them to 'yoga-eol' in the kolla namespace
for image in $(docker images --format "{{.Repository}}:{{.Tag}}" | grep 'quay.io/openstack.kolla/ubuntu-source'); do
    # Retag the image to kolla namespace
    new_image="kolla/$(echo $image | cut -d/ -f2-)"
    docker tag $image $new_image
    echo "Retagged $image to $new_image"
done

# Verify the new tags
echo "Verifying the new tags..."
docker images | grep kolla/ubuntu-source

# Remove old images from quay.io/openstack.kolla namespace (duplicates)
for old_image in $(docker images --format "{{.Repository}}:{{.Tag}}" | grep 'quay.io/openstack.kolla/ubuntu-source'); do
    docker rmi -f $old_image
    echo "Removed old image: $old_image"
done

```
###Run the Script
```bash
sudo chmod +x retag_images.sh
./retag_images.sh
```

Verify:
```bash
   sudo docker image ls
#--> Results should show only "kolla/..." 
```
#### 🔍 3. Check and Modify Image (Optional)

1. **Enter the image to inspect**:

   You can enter the built Docker image for inspection:

   ```bash
   docker run -it kolla/ubuntu-source-kolla-toolbox:yoga bash
   ```

2. **Inspect the image**:

   Run commands to inspect the image:

   ```bash
   ls -l /usr/bin/sudo
   getent passwd ansible
   getent group kolla
   ```

## Deploy Openstach use Kolla-ansible

### Step 1. Re-configure `/etc/kolla/globals.yml` to Use Local Images

Edit the file:

```bash
sudo cp /etc/kolla/globals.yml /etc/kolla/globals.yml.1
sudo nano /etc/kolla/globals.yml
```

### ✅ Add or modify these entries

```yaml

docker_registry: ""
docker_registry_insecure: "no"
docker_namespace: "kolla"
```

---

## 🚀 4. Deploy with Kolla-Ansible Using Local Images

### ✅ Deploy

```bash
cd /etc/kolla/ansible/inventory/
source ~/kolla-venv/bin/activate
   
# Verify setting
ansible localhost -m setup | grep ansible_interfaces -A10

# Deploy openstack
sudo systemctl restart docker
sudo docker ps -q | xargs -r sudo docker stop
kolla-ansible deploy -i /etc/kolla/ansible/inventory/all-in-one
```
##Debugs if some false
ansible-playbook \
  -i /etc/kolla/ansible/inventory/all-in-one \
  -e @/etc/kolla/globals.yml \
  -e @/etc/kolla/passwords.yml \
  -e kolla_action=deploy \
  /home/deployer/kolla-venv/share/kolla-ansible/ansible/site.yml -vvvv
###

### ✅ Post-Deployment

```bash
# Post-deploy 
kolla-ansible post-deploy -i /etc/kolla/ansible/inventory/all-in-one

# Generate openrc file
sudo cp /etc/kolla/admin-openrc.sh ~/
sudo chmod +r ~/admin-openrc.sh
source ~/admin-openrc.sh
```

### ✅ Verification and Troubleshooting

### Verify OpenStack Services

```bash
# Install OpenStack client
pip install python-openstackclient

# Source credentials
sudo chmod +r /etc/kolla/admin-openrc.sh
source /etc/kolla/admin-openrc.sh

# List services
openstack service list

# --> results
(kolla-venv) deployer@aio:/etc/kolla/ansible/inventory$ openstack service list
+----------------------------------+-------------+----------------+
| ID                               | Name        | Type           |
+----------------------------------+-------------+----------------+
| 150ee4d8d76a445d85a22a26d9984881 | heat        | orchestration  |
| 18be4a3e682e4985b863a75c1ca6a130 | heat-cfn    | cloudformation |
| 4112cb741289482ca287f2c8d7f19788 | neutron     | network        |
| 6d8f41fd5696498b8abdddeacd316b7c | nova        | compute        |
| 911add4883d24034a89074003bca342d | glance      | image          |
| c78bcff440ba411fbc4d98bfa59be0d3 | placement   | placement      |
| e96fdf7df40d471989473118d58782b2 | nova_legacy | compute_legacy |
| ee9a7cae5a3d4be99b266ee34538f21f | keystone    | identity       |
+----------------------------------+-------------+----------------+

# List endpoints
openstack endpoint list

# --> results
+---------------+-----------+--------------+---------------+---------+-----------+---------------+
| ID            | Region    | Service Name | Service Type  | Enabled | Interface | URL           |
+---------------+-----------+--------------+---------------+---------+-----------+---------------+
| 00fa628a956d4 | RegionOne | heat-cfn     | cloudformatio | True    | public    | http://192.16 |
| d95892214270f |           |              | n             |         |           | 8.1.254:8000/ |
| 9a4ba2        |           |              |               |         |           | v1            |
| 04922466afc84 | RegionOne | neutron      | network       | True    | internal  | http://192.16 |
| 1498e6506095b |           |              |               |         |           | 8.1.254:9696  |
| 0361e2        |           |              |               |         |           |               |
| 35a77986f50c4 | RegionOne | heat-cfn     | cloudformatio | True    | internal  | http://192.16 |
| a4f81745908ef |           |              | n             |         |           | 8.1.254:8000/ |
| 9b15cf        |           |              |               |         |           | v1            |
| 47a4a7edfa824 | RegionOne | nova         | compute       | True    | public    | http://192.16 |
| c3c8224dcb180 |           |              |               |         |           | 8.1.254:8774/ |
| 2524d4        |           |              |               |         |           | v2.1          |
| 51f5b824abcc4 | RegionOne | glance       | image         | True    | internal  | http://192.16 |
| a2fa96efc2f6d |           |              |               |         |           | 8.1.254:9292  |
| fa35f0        |           |              |               |         |           |               |
| 63647a1a55604 | RegionOne | placement    | placement     | True    | public    | http://192.16 |
| ff695b6221466 |           |              |               |         |           | 8.1.254:8780  |
| 17dc73        |           |              |               |         |           |               |
| 646fd30acc164 | RegionOne | glance       | image         | True    | public    | http://192.16 |
| c31906d5a598a |           |              |               |         |           | 8.1.254:9292  |
| 9655f3        |           |              |               |         |           |               |
| 68bae4e1686a4 | RegionOne | nova_legacy  | compute_legac | True    | internal  | http://192.16 |
| 4b3a73c5ffa07 |           |              | y             |         |           | 8.1.254:8774/ |
| ed9c9f        |           |              |               |         |           | v2/%(tenant_i |
|               |           |              |               |         |           | d)s           |
| 7cb1957087b64 | RegionOne | keystone     | identity      | True    | internal  | http://192.16 |
| 30db45574eed8 |           |              |               |         |           | 8.1.254:5000  |
| d08011        |           |              |               |         |           |               |
| 9c7a76ab19ca4 | RegionOne | neutron      | network       | True    | public    | http://192.16 |
| 596969ae12c57 |           |              |               |         |           | 8.1.254:9696  |
| ebb663        |           |              |               |         |           |               |
| 9d556c671aeb4 | RegionOne | keystone     | identity      | True    | admin     | http://192.16 |
| e3f98d897d95b |           |              |               |         |           | 8.1.254:35357 |
| c454e0        |           |              |               |         |           |               |
| 9e0351453a204 | RegionOne | keystone     | identity      | True    | public    | http://192.16 |
| 2f7a405383a5e |           |              |               |         |           | 8.1.254:5000  |
| d1836c        |           |              |               |         |           |               |
| 9fcb8e105a464 | RegionOne | placement    | placement     | True    | internal  | http://192.16 |
| 97a80219f99f5 |           |              |               |         |           | 8.1.254:8780  |
| bdfb95        |           |              |               |         |           |               |
| b589279a1e354 | RegionOne | nova         | compute       | True    | internal  | http://192.16 |
| 8c9a1273a7044 |           |              |               |         |           | 8.1.254:8774/ |
| 475045        |           |              |               |         |           | v2.1          |
| c17892bd56ae4 | RegionOne | heat         | orchestration | True    | public    | http://192.16 |
| 36595d6f66a3e |           |              |               |         |           | 8.1.254:8004/ |
| 76e74c        |           |              |               |         |           | v1/%(tenant_i |
|               |           |              |               |         |           | d)s           |
| f2c80bff02cc4 | RegionOne | nova_legacy  | compute_legac | True    | public    | http://192.16 |
| 63e807b4c05aa |           |              | y             |         |           | 8.1.254:8774/ |
| 12aed5        |           |              |               |         |           | v2/%(tenant_i |
|               |           |              |               |         |           | d)s           |
| facd5c81ea124 | RegionOne | heat         | orchestration | True    | internal  | http://192.16 |
| ade86ecaca033 |           |              |               |         |           | 8.1.254:8004/ |
| 2b9ae5        |           |              |               |         |           | v1/%(tenant_i |
|               |           |              |               |         |           | d)s           |
+---------------+-----------+--------------+---------------+---------+-----------+---------------+

### ✅ Dashboard login
http://192.168.1.254
name: admin
#Check all password in `/etc/kolla/cat passwords.yml`
# Example for keystone
grep keystone_admin_password /etc/kolla/passwords.yml
# --> keystone_admin_password: y6pn3SnyvhsFilQyOrTAIuvZzzQR4MHesGuXJSNl

### Troubleshooting Network Issues

If you encounter network connectivity issues:

1. **Check Bridge Status**:

   ```bash
   ip link show br-exnat
   bridge link show
   ```

2. **Verify IP Forwarding**:

   ```bash
   cat /proc/sys/net/ipv4/ip_forward  # Should be 1
   ```

3. **Check NAT Rules**:

   ```bash
   sudo iptables -t nat -L -n -v
   ```

4. **Verify Routing**:

   ```bash
   ip route show
   ```

5. **Test with Traceroute**:

   ```bash
   # Install traceroute if needed
   sudo apt install traceroute -y
   
   # Test from namespace
   sudo ip netns exec testns traceroute 8.8.8.8
   ```

6. **Check Docker/Container Networking**:

   ```bash
   sudo docker network ls
   sudo docker ps
   ```

---

## 🔄 Making Configuration Persistent

### Network Configuration

```bash
# Create startup script
sudo nano /etc/network/if-up.d/bridge-setup
```

# Copy to `/etc/network/if-up.d/bridge-setup`

```ini
#!/bin/bash
# Script to setup br-exnat bridge after reboot, create veth, assign VIP, enable forwarding and NAT

set -e

# Only run if br-exnat exists
if ip link show br-exnat >/dev/null 2>&1; then
    echo "Setting up br-exnat bridge..."

    # Check if veth pair already exists
    if ! ip link show veth0 >/dev/null 2>&1; then
        ip link add veth0 type veth peer name veth1
    fi

    # Attach veth1 to br-exnat (ignore error if already attached)
    brctl addif br-exnat veth1 2>/dev/null || true

    # Bring up interfaces
    ip link set veth0 up
    ip link set veth1 up
    ip link set br-exnat up

    # Assign VIP if not already assigned
    
    echo "br-exnat bridge setup complete."

    echo "Setting up IP forwarding and NAT..."

    # Enable IP forwarding
    sysctl -w net.ipv4.ip_forward=1

    # Setup NAT (prevent duplicates by cleaning old rules first)
    iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o wlp4s0 -j MASQUERADE 2>/dev/null || true
    iptables -D FORWARD -i br-exnat -o wlp4s0 -j ACCEPT 2>/dev/null || true
    iptables -D FORWARD -i wlp4s0 -o br-exnat -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || true

    iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o wlp4s0 -j MASQUERADE
    iptables -P FORWARD ACCEPT
    iptables -A FORWARD -i br-exnat -o wlp4s0 -j ACCEPT
    iptables -A FORWARD -i wlp4s0 -o br-exnat -m state --state RELATED,ESTABLISHED -j ACCEPT

    echo "IP forwarding and NAT setup complete."

else
    echo "br-exnat does not exist yet. Skipping bridge setup."
fi
```

# Set execution

sudo chmod +x /etc/network/if-up.d/bridge-setup

# Verify: Reboot the verify

```bash
ip a
iptables -t nat -L -n -v
iptables -L FORWARD -n -v

#---> the result should show br-exnat is UP and confirm NAT and FORWARD rules exist after reboot.

---

## 📝 Important Notes

1. **Bridge Layer 2 Nature**: Remember that bridges operate at Layer 2. You cannot directly ping external addresses from the bridge interface itself. Always test connectivity from a host (or namespace) connected to the bridge.

2. **Interface Names**: Replace `wlp4s0` and other interface names with your actual interface names throughout this guide.

3. **IP Addresses**: Adjust IP addresses according to your network setup. Ensure the external VIP (`kolla_external_vip_address`) is an unused IP on your external network.

4. **OpenStack Version**: The guide uses "yoga" as the OpenStack release. Adjust this to your preferred version.

5. **All-in-One vs. Multinode**: This guide uses the all-in-one inventory for simplicity. For production deployments, consider using a multinode setup.

6. *Turn of all dotker running*
```bash
sudo docker ps -q | xargs -r sudo docker stop
```

7. Restart docker

```bash
sudo systemctl restart docker
```

8. Build openvswitch-db-server if deploy raise errors

```bash
sudo pip3 install kolla
sudo kolla-build openvswitch-db-server
sudo docker image rm quay.io/openstack.kolla/ubuntu-source-openvswitch-db-server

#Re deploy
kolla-ansible deploy -i ./all-in-one
```
