---
title: "Running a Debian 13 VM on Fedora 42 with Bridged Networking, Podman, and Minikube"
date: "2026-05-11T05:07:39Z"
aliases: ["/projects/debian13-vm-on-fedora-bridged-networking-podman-minikube/"]
author: "Sanjay Upadhyay"
description: >
  A complete, step-by-step guide to creating a Debian 13 virtual machine on
  Fedora 42 with bridged networking, cloud-init provisioning, root password
  setup, container tooling (podman, buildah, skopeo), and a multi-node
  Kubernetes cluster via minikube.
tags: ["tech","fedora", "debian", "kvm", "libvirt", "podman", "minikube", "kubernetes", "cloud-init"]
categories: ["Projects", "virtualization", "containers"]
draft: false
---

## Why This Guide Exists

You're on Fedora 42. You want a Debian 13 VM — not through some GUI wizard,
but properly, from the command line, with real bridged networking so the VM
sits on your LAN like a first-class citizen. You want cloud-init to set up
your users and passwords. And then you want to turn that VM into a
multi-node Kubernetes cluster with podman and minikube.

Every guide I found either stopped at "install the VM" or assumed you
already knew the networking parts. This one doesn't. It holds your hand
from `dnf install` to `kubectl get nodes`.

Let's go.

---

## 1. Prerequisites

### Install the virtualization stack

```bash
sudo dnf install -y qemu-kvm libvirt virt-install libguestfs-tools \
  genisoimage virt-top bridge-utils
```

### Start libvirtd

```bash
sudo systemctl enable --now libvirtd
```

Verify it's running:

```bash
systemctl status libvirtd
```

---

## 2. Run virt-install as a Normal User (Without sudo)

By default, `virt-install` and `virsh` require root. But you don't want to
be root every time you manage your VMs. Here's how to fix that.

### Add your user to the required groups

```bash
sudo usermod -aG libvirt,wheel $(whoami)
```

> **What these groups do:**
>
> | Group | Purpose |
> |-------|---------|
> | `libvirt` | Grants access to the system libvirt daemon (`qemu:///system`). Without this, you can only use the session daemon (`qemu:///session`), which has limitations. |
> | `wheel` | Allows `sudo` access (Fedora's default admin group). You'll still need `sudo` for some host-level operations, but VM management itself won't require it. |

### Verify group membership

Log out and log back in (or open a new terminal), then:

```bash
groups
```

You should see `libvirt` and `wheel` in the output. If you don't want to
log out, you can temporarily activate the groups:

```bash
newgrp libvirt
```

### Test that virsh works without sudo

```bash
virsh list --all
```

If you see output (even an empty list) instead of a permission error, you're
good. If you get "access denied", see the [Troubleshooting](#troubleshooting)
section.

### A note on session vs. system mode

You now have two ways to run VMs:

| Mode | URI | What it means |
|------|-----|---------------|
| **System** | `qemu:///system` | VMs are owned by root, survive reboots, can use bridges. **This is what we want.** |
| **Session** | `qemu:///session` | VMs are owned by your user, lost on reboot, limited networking. Avoid this. |

Commands like `virsh` and `virt-install` default to system mode when you're
in the `libvirt` group. If you ever need to be explicit:

```bash
virsh --connect qemu:///system list --all
```

---

## 3. Create a Network Bridge (br0)

Without a bridge, your VM would be stuck behind NAT — unable to get its own
LAN IP address. A bridge makes your VM a peer on the network, just like your
host machine.

### 3.1 Find your Ethernet device

```bash
nmcli device status
```

Look for the row where TYPE is `ethernet` and STATE is `connected`. Note the
**DEVICE** column (e.g., `enp130s0`, `eno1`, `enp3s0`). Also note the
**CONNECTION** column — that's the NM profile name (e.g., "Wired connection 1").

### 3.2 Check your current IP settings

You'll need these if you use a static IP. Even if you use DHCP, it's good
to know what you're working with:

```bash
nmcli -t -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns \
  connection show "Wired connection 1"
```

### 3.3 Create the bridge

**If DHCP (most common on home networks):**

```bash
sudo nmcli connection add type bridge ifname br0 con-name br0 \
  ipv4.method auto ipv6.method auto
```

**If static IP (fill in your actual values):**

```bash
sudo nmcli connection add type bridge ifname br0 con-name br0 \
  ipv4.method manual \
  ipv4.addresses 192.168.2.112/24 \
  ipv4.gateway 192.168.2.1 \
  ipv4.dns '192.168.2.1,1.1.1.1' \
  ipv6.method disabled
```

### 3.4 Enslave the Ethernet device under the bridge

**This is the step most people miss — and without it, the bridge has no
ports, shows NO-CARRIER, and never gets an IP.**

```bash
sudo nmcli connection add type bridge-slave ifname enp130s0 master br0 \
  con-name br0-slave-enp130s0
```

Replace `enp130s0` with your actual device name from step 3.1.

### 3.5 Prevent the old connection from conflicting

When `enp130s0` becomes free, NetworkManager might re-activate your old
"Wired connection 1" profile instead of the bridge slave. The cleanest fix
is to delete it — you no longer need it (the bridge + slave replaces it):

```bash
sudo nmcli connection delete "Wired connection 1"
```

If you're nervous about this, you can instead just disable autoconnect:

```bash
sudo nmcli connection modify "Wired connection 1" connection.autoconnect no
```

### 3.6 Switch over

> ⚠️ **Warning:** If you are running these commands over SSH on that same
> interface, your session will drop. Either run from a local console, or
> combine both commands on one line so NM transitions quickly:

```bash
sudo nmcli connection down "Wired connection 1" && sudo nmcli connection up br0
```

### 3.7 Verify the bridge

```bash
ip -br addr show br0
bridge link show br0
ip route list
```

You should see:

- `br0` is `UP` with an IP address (not `NO-CARRIER`)
- `enp130s0` is listed as a bridge port
- A default route through `br0`

### 3.8 Allow the bridge through firewalld

```bash
sudo firewall-cmd --add-interface=br0 --zone=public --permanent
sudo firewall-cmd --reload
```

---

## 4. What If You Don't Have an Ethernet NIC?

Bridging works cleanly with Ethernet. **WiFi bridging on Linux does not work
reliably** — most WiFi APs reject frames from MAC addresses that aren't
associated, and bridging would make the VM appear with a different MAC. So
if your host only has WiFi, or you're on a laptop, you need alternatives.

### Option A: Use libvirt's default NAT network (virbr0)

This is the easiest option and works out of the box. Your VM gets a private
IP (typically `192.168.122.x`), and the host NATs traffic to the outside
world. The VM can reach the internet, but external machines can't reach the
VM directly.

```bash
# virbr0 is already created by libvirt. Verify it:
virsh net-list --all

# If it's not active:
sudo virsh net-start default
sudo virsh net-autostart default
```

Then in your `virt-install` command, use:

```bash
--network network=default,model=virtio
```

instead of `--network bridge=br0,model=virtio`.

**Pros:** Zero setup. Works on WiFi. **Cons:** VM is behind NAT, no direct
LAN access, port forwarding requires extra configuration.

### Option B: Create a custom libvirt NAT network

If you want a dedicated NAT network (e.g., a specific subnet):

```bash
cat << 'EOF' | sudo tee /etc/libvirt/qemu/networks/nat-net.xml
<network>
  <name>nat-net</name>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip address='10.100.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.100.0.100' end='10.100.0.200'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /etc/libvirt/qemu/networks/nat-net.xml
sudo virsh net-start nat-net
sudo virsh net-autostart nat-net
```

Then use `--network network=nat-net,model=virtio` in `virt-install`.

### Option C: Use macvtap (Ethernet or WiFi in some cases)

Macvtap gives the VM its own virtual NIC with a unique MAC on the physical
network. It works with WiFi in **some** configurations (if your WiFi driver
and AP support it), but it's unreliable and the host and VM **cannot
communicate with each other** over the same interface.

```bash
# No bridge setup needed. Just use macvtap directly in virt-install:
--network type=direct,source=enp130s0,source_mode=vepa,model=virtio
```

**This is not recommended for WiFi.** It's listed here for completeness.

### Option D: SSH tunnels for accessing VM services

With NAT networking, you can't reach the VM directly from the host. Use SSH
tunnels:

```bash
ssh -L 8080:localhost:8080 -L 8888:localhost:8888 -N sanjayu@<VM_IP>
```

Then access services on `localhost:8080` and `localhost:8888` from your host
browser.

### Summary — Which networking to use

| Scenario | Recommended | Command |
|----------|-------------|---------|
| Desktop with Ethernet | Bridge (`br0`) | `--network bridge=br0,model=virtio` |
| Laptop with WiFi only | libvirt NAT (`virbr0`) | `--network network=default,model=virtio` |
| Need custom subnet | Custom NAT network | `--network network=nat-net,model=virtio` |
| Advanced, both on Ethernet | macvtap | `--network type=direct,source=eth0,model=virtio` |

---

## 5. Download the Debian 13 NoCloud Image

Grab the official Debian 13 (Trixie) NoCloud qcow2 image. The NoCloud variant
supports cloud-init without requiring a cloud platform.

```bash
cd ~
wget https://cloud.debian.org/images/cloud/trixie/daily/latest/debian-13-nocloud-amd64-daily.qcow2
```

> The exact filename will vary by date. Adjust accordingly. The NoCloud
> image is important — the regular cloud image expects OpenStack/AWS
> metadata, which we don't have.

---

## 6. Create the VM Disk (qcow2 Overlay)

We create a **copy-on-write overlay** backed by the original image. This means
the downloaded image stays pristine — all writes go to the overlay. You can
always create fresh VMs from the same base image.

```bash
qemu-img create -f qcow2 \
  -b /home/sanjayu/debian-13-nocloud-amd64-daily.qcow2 \
  -F qcow2 \
  /home/sanjayu/debian13-vm0.qcow2 \
  40G
```

> **Why 40G?** The virtual disk size is 40 GB, but the actual file on disk
> starts tiny and grows only as you write data. The base image might be
> ~2 GB, and the overlay starts at a few KB.

Verify:

```bash
qemu-img info /home/sanjayu/debian13-vm0.qcow2
```

You should see a virtual size of 40 GiB and the backing file path.

---

## 7. Create the Cloud-Init Seed ISO

The NoCloud image expects a CDROM labelled `cidata` with `meta-data` and
`user-data` files. This is how we set the hostname, create users, and set
passwords without ever logging into the VM manually.

### 7.1 Write the meta-data

```bash
mkdir -p /tmp/cidata

cat > /tmp/cidata/meta-data << 'EOF'
instance-id: debian13-vm0
local-hostname: debian13-vm0
EOF
```

### 7.2 Write the user-data

This is the important one. It creates your user, sets passwords, installs
packages, enables SSH, and — critically — grows the root partition to fill
the 40 GB disk (without this, your root filesystem stays at ~2 GB and
you'll run out of space as soon as you install anything).

```bash
cat > /tmp/cidata/user-data << 'EOF'
#cloud-config
hostname: debian13-vm0
manage_etc_hosts: true

# Automatically grow the root partition to fill the disk
growpart:
  mode: auto
  devices: ['/']

resize_rootfs: true

# Create a sudo user
users:
  - name: sanjayu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    groups: sudo, adm

# Set passwords for root and sanjayu
chpasswd:
  list: |
    root:adc087c9
    sanjayu:adc087c9
  expire: false

# Enable SSH password authentication and root login
ssh_pwauth: true
disable_root: false

# Install essential packages
packages:
  - openssh-server
  - sudo
  - cloud-guest-utils
  - qemu-guest-agent

# Post-boot commands
runcmd:
  - sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
  - systemctl enable sshd
  - systemctl restart sshd
  - systemctl enable qemu-guest-agent
  - systemctl start qemu-guest-agent
EOF
```

> **Security note:** Change `adc087c9` to a strong password. For production
> use, replace password auth with SSH keys using `ssh_authorized_keys:` under
> the `users` section.

### 7.3 Generate the seed ISO

```bash
genisoimage -output /home/sanjayu/debian13-vm0-seed.iso \
  -volid cidata -joliet -rock \
  /tmp/cidata/user-data /tmp/cidata/meta-data
```

> If `genisoimage` isn't found, install it: `sudo dnf install genisoimage`.
> Alternatively, use `mkisofs` from the `xorriso` package.

---

## 8. Allow QEMU to Use the Bridge

When running VMs as an unprivileged user (session mode or via the bridge
helper), QEMU needs explicit permission to attach to your bridge. Without
this, you'll get `access denied by acl file`.

```bash
echo "allow br0" | sudo tee -a /etc/qemu-kvm/bridge.conf
```

Verify:

```bash
cat /etc/qemu-kvm/bridge.conf
```

Should show:

```
allow virbr0
allow br0
```

Also make sure the bridge helper is setuid (it normally is on Fedora):

```bash
ls -l /usr/libexec/qemu-bridge-helper
# Should show: -rwsr-xr-x
```

---

## 9. Create the VM with virt-install

Now the moment of truth. Since you're in the `libvirt` group, this works
without `sudo`:

```bash
virt-install \
  --name debian13-vm0 \
  --memory 8192 \
  --vcpus 8 \
  --disk path=/home/sanjayu/debian13-vm0.qcow2,bus=virtio,format=qcow2 \
  --disk path=/home/sanjayu/debian13-vm0-seed.iso,device=cdrom,bus=sata \
  --network bridge=br0,model=virtio \
  --osinfo require=off \
  --import \
  --noautoconsole
```

> **If using NAT networking instead of a bridge**, replace the `--network`
> line with:
> ```bash
> --network network=default,model=virtio
> ```

What each flag does:

| Flag | Purpose |
|------|---------|
| `--name` | VM name as shown by `virsh list` |
| `--memory 8192` | 8 GiB RAM |
| `--vcpus 8` | 8 virtual CPUs |
| 1st `--disk` | The 40 GB overlay disk on virtio bus |
| 2nd `--disk` | The cloud-init seed ISO as a SATA CDROM |
| `--network bridge=br0` | Bridged to your physical LAN |
| `--osinfo require=off` | Skips OS validation (Debian 13 may not be in osinfo-db yet) |
| `--import` | The disk already has an OS — don't run an installer |
| `--noautoconsole` | Don't attach to the serial console automatically |

Wait about 30 seconds, then verify:

```bash
virsh list --all
```

The VM should be in the `running` state.

---

## 10. Connect and Verify

### Get the VM's IP address

```bash
# If using bridged networking, the VM gets a DHCP lease from your LAN:
virsh domifaddr debian13-vm0

# Or use the QEMU guest agent (more reliable):
virsh domifaddr --source agent debian13-vm0
```

### SSH in

```bash
ssh sanjayu@<VM_IP>
# password: adc087c9 (or whatever you set in cloud-init)
```

### Verify the disk was grown

```bash
df -h /
# Should show ~40 GB, NOT ~2 GB
```

### Verify the user and sudo

```bash
whoami        # sanjayu
sudo whoami   # root
```

---

## 11. Install Container Tools

Once you're inside the VM:

```bash
sudo apt-get update
sudo apt-get install -y podman buildah skopeo podman-compose
```

Verify everything installed correctly:

```bash
podman --version       # e.g., podman version 5.x
buildah --version      # e.g., buildah version 1.x
skopeo --version       # e.g., skopeo version 1.x
podman-compose --version
```

Quick smoke test:

```bash
podman run --rm -it docker.io/library/alpine echo "Podman works!"
```

---

## 12. Install Minikube and kubectl

```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

minikube version
kubectl version --client
```

---

## 13. Prepare the Kernel for Kubernetes

```bash
# Load required kernel modules
sudo modprobe br_netfilter overlay

# Persist them across reboots
cat << 'EOF' | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF

# Sysctl settings required by Kubernetes
cat << 'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## 14. Create a Multi-Node Kubernetes Cluster

```bash
sudo minikube start \
  --driver=podman \
  --nodes=3 \
  --cni=calico \
  --cpus=2 \
  --memory=2g \
  --disk-size=10g \
  -p k8s-cluster
```

> **Why `sudo`?** Minikube's podman driver is more reliable in rootful mode.
> Rootless podman with minikube is experimental and has known issues with
> cgroups and networking.

### Copy the kubeconfig to your user

```bash
mkdir -p ~/.kube
sudo cp /root/.kube/config ~/.kube/config
sudo chown -R $(id -u):$(id -g) ~/.kube
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

### Verify the cluster

```bash
kubectl get nodes -o wide
```

You should see three nodes — one control-plane and two workers — all
`Ready`.

---

## 15. Deploy Demo Workloads

### A simple Nginx deployment

```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide
```

### A full voting app (vote → redis → worker → postgres → result)

```bash
kubectl create namespace vote

kubectl apply -n vote -f - <<'EOF'
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
    - port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  ports:
    - port: 5432
  selector:
    app: db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:16-alpine
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
            - name: POSTGRES_DB
              value: votes
          ports:
            - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: vote
spec:
  type: NodePort
  ports:
    - port: 80
  selector:
    app: vote
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
        - name: vote
          image: dockersamples/examplevotingapp_vote
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: result
spec:
  type: NodePort
  ports:
    - port: 80
  selector:
    app: result
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result
  template:
    metadata:
      labels:
        app: result
    spec:
      containers:
        - name: result
          image: dockersamples/examplevotingapp_result
          ports:
            - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
        - name: worker
          image: dockersamples/examplevotingapp_worker
EOF
```

---

## 16. Access the Services from Your Host

The NodePort IPs (like `192.168.49.x`) exist only inside the VM's podman
network. Your Fedora host can't reach them directly. Here are three ways
to access them, from simplest to most feature-rich.

### Option A: kubectl port-forward + SSH tunnel (recommended)

From inside the VM:

```bash
kubectl port-forward -n vote svc/vote 8080:80 &
kubectl port-forward -n vote svc/result 8888:80 &
```

From your Fedora host:

```bash
ssh -L 8080:localhost:8080 -L 8888:localhost:8888 -N sanjayu@<VM_IP>
```

Then open in your host's browser:

- 🗳️ Vote: `http://localhost:8080`
- 📊 Result: `http://localhost:8888`

### Option B: Firefox via SSH X11 forwarding

Install Firefox on the VM:

```bash
sudo apt-get install -y firefox-esr xauth
```

From your Fedora host (must be running a graphical session):

```bash
ssh -X sanjayu@<VM_IP>
firefox-esr --no-remote &
```

> If Firefox doesn't render, try:
> ```bash
> MOZ_DISABLE_CONTENT_SANDBOX=1 GDK_BACKEND=x11 \
>   firefox-esr --no-remote http://<NODE_IP>:<VOTE_PORT> &
> ```

### Option C: VNC server on the VM

If X11 forwarding is too slow or unreliable:

```bash
# Inside the VM:
sudo apt-get install -y tigervnc-standalone-server firefox-esr
vncpasswd
vncserver :1 -geometry 1920x1080
```

From the Fedora host, SSH-tunnel the VNC port:

```bash
ssh -L 5901:localhost:5901 -N sanjayu@<VM_IP>
```

Then connect with any VNC client (Remmina, TigerVNC) to `localhost:5901`.

---

## 17. Build and Deploy a Custom Image with Podman

This ties together podman, buildah, and your Kubernetes cluster.

```bash
# Create a small app
mkdir -p ~/demo-app && cd ~/demo-app

cat > app.py << 'PYEOF'
from http.server import HTTPServer, SimpleHTTPRequestHandler
import os, socket
class Handler(SimpleHTTPRequestHandler):
    def do_GET(self):
        msg = f"Hello from podman-built image!\nPod: {os.environ.get('HOSTNAME','?')}\nNode: {socket.gethostname()}\n"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(msg.encode())
if __name__ == "__main__":
    HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
PYEOF

cat > Dockerfile << 'DEOF'
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
DEOF
```

### Build with podman (or buildah)

```bash
sudo podman build -t demo-app:latest ~/demo-app
```

### Push into minikube

```bash
sudo podman save demo-app:latest | sudo minikube image load -p k8s-cluster -
```

### Deploy

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: demo-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
spec:
  type: NodePort
  ports:
    - port: 8080
  selector:
    app: demo-app
EOF
```

---

## Troubleshooting {#troubleshooting}

### `virsh: access denied` — Permission issues

**Symptom:** Running `virsh list` gives a permission error.

**Fix:**

```bash
# Check group membership
groups

# If 'libvirt' is missing, add it and re-login:
sudo usermod -aG libvirt $(whoami)
# Log out and back in, or:
newgrp libvirt

# Check libvirt socket permissions
ls -l /var/run/libvirt/libvirt-sock
# Should be: srwxrwx--- 1 root libvirt

# Verify POLICYKIT auth
sudo cat /etc/polkit-1/rules.d/50-libvirt.rules
# If missing, create it:
sudo tee /etc/polkit-1/rules.d/50-libvirt.rules << 'EOF'
polkit.addRule(function(action, subject) {
    if (action.id == "org.libvirt.unix.manage" &&
        subject.isInGroup("libvirt")) {
        return polkit.Result.YES;
    }
});
EOF
```

### Bridge shows `NO-CARRIER` / no IP / no routes

**Symptom:** `ip addr show br0` shows `NO-CARRIER` and no IP address.

**Root cause:** The bridge has no ports attached. The bridge-slave
connection was either never created or never activated.

**Fix:**

```bash
# 1. Check if a bridge-slave connection exists
nmcli con show | grep slave

# 2. If missing, create it (replace enp130s0 with your device):
sudo nmcli connection add type bridge-slave ifname enp130s0 master br0 \
  con-name br0-slave-enp130s0

# 3. Make sure the old wired connection doesn't conflict:
sudo nmcli connection delete "Wired connection 1"
# Or disable its autoconnect:
# sudo nmcli connection modify "Wired connection 1" connection.autoconnect no

# 4. Restart the bridge:
sudo nmcli connection down br0
sudo nmcli connection up br0
```

### `access denied by acl file` — QEMU bridge helper

**Symptom:** `virt-install` fails with `access denied by acl file`.

**Root cause:** The QEMU bridge helper doesn't have `br0` in its ACL.

**Fix:**

```bash
echo "allow br0" | sudo tee -a /etc/qemu-kvm/bridge.conf
# Also check /etc/qemu/bridge.conf — add it there too if it exists:
echo "allow br0" | sudo tee -a /etc/qemu/bridge.conf
```

### `no space left on device` inside the VM

**Symptom:** `df -h /` shows only ~2 GB even though you created a 40 GB disk.
Minikube or apt-get fails with "no space left on device".

**Root cause:** The NoCloud image has a small root partition. The cloud-init
`growpart` module should expand it, but if it didn't run (or wasn't
configured), you're stuck at ~2 GB.

**Fix (manual):**

```bash
# Check the partition layout
lsblk

# Install growpart
sudo apt-get install -y cloud-guest-utils

# Grow partition 1 (adjust the partition number from lsblk output)
sudo growpart /dev/vda 1

# Grow the filesystem
sudo resize2fs /dev/vda1    # for ext4
# sudo xfs_growfs /          # for XFS

# Verify
df -h /
```

**Fix (cloud-init — prevent it next time):** Make sure your `user-data`
includes:

```yaml
growpart:
  mode: auto
  devices: ['/']
resize_rootfs: true
packages:
  - cloud-guest-utils
```

### Minikube fails to start with podman driver

**Symptom:** `minikube start --driver=podman` fails with various errors.

**Common fixes:**

```bash
# 1. Make sure cgroups v2 is active
stat -fc %T /sys/fs/cgroup   # should print 'cgroup2fs'

# 2. Make sure kernel modules are loaded
sudo modprobe br_netfilter overlay
sudo sysctl --system

# 3. Try with explicit rootful mode
sudo minikube start --driver=podman --force

# 4. If pods keep crashing, check podman status
sudo podman ps -a
sudo podman logs <container-id>

# 5. Nuclear option — delete and recreate
sudo minikube delete -p k8s-cluster
sudo minikube start --driver=podman --nodes=3 --cni=calico \
  --cpus=2 --memory=2g --disk-size=10g -p k8s-cluster
```

### Can't reach NodePort services from the host

**Symptom:** `curl http://<VM_IP>:<NodePort>` times out from the Fedora host.

**Root cause:** The NodePort is listening on the podman network
(192.168.49.x), not on the VM's LAN IP.

**Fix:** Use `kubectl port-forward` + SSH tunnel (see Section 16 above).

### `virt-install` fails with "Transport endpoint is not connected"

**Symptom:** VM creation fails immediately.

**Root cause:** Usually a bridge or network misconfiguration.

**Fix:**

```bash
# Check libvirt network status
virsh net-list --all

# If using a bridge, verify it's up
ip link show br0

# Check libvirt logs
sudo journalctl -u libvirtd --since "5 minutes ago"

# Try with the default NAT network to isolate the issue:
virt-install --network network=default ...
```

### Cloud-init doesn't run / password not set

**Symptom:** Can't SSH in, password doesn't work.

**Fix:**

```bash
# 1. Verify the seed ISO has the right volume label
isoinfo -d -i /home/sanjayu/debian13-vm0-seed.iso | grep "Volume id"
# Should print: Volume id: cidata

# 2. Connect to the serial console to see what's happening
virsh console debian13-vm0

# 3. Check cloud-init logs (once you get in):
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log

# 4. Re-run cloud-init manually:
sudo cloud-init clean
sudo cloud-init init
sudo cloud-init modules --mode=config
sudo cloud-init modules --mode=final
```

### SSH connection drops when creating the bridge

**Symptom:** You lose your SSH session when running `nmcli connection down`.

**Prevention:**

```bash
# Combine both commands on one line so NM transitions quickly:
sudo nmcli connection down "Wired connection 1" && sudo nmcli connection up br0

# Or use a local console / out-of-band access

# After reconnection, verify:
ip addr show br0
ping -c 3 8.8.8.8
```

### VM is stuck at "Booting from ROM"

**Symptom:** `virsh console debian13-vm0` shows nothing after GRUB.

**Fix:**

```bash
# The image might need a serial console configured.
# Try connecting with VNC instead:
virsh vncdisplay debian13-vm0

# Or add a serial console and re-provision:
virt-install ... --serial pty --graphics vnc,listen=0.0.0.0
```

---

## Cleanup Commands

```bash
# Inside the VM — delete all K8s resources
kubectl delete deploy,svc --all --all-namespaces
sudo minikube delete -p k8s-cluster

# On the host — remove the VM
virsh destroy debian13-vm0
virsh undefine debian13-vm0
rm -f /home/sanjayu/debian13-vm0.qcow2 /home/sanjayu/debian13-vm0-seed.iso

# Remove the bridge (if you want to go back to direct Ethernet)
sudo nmcli connection down br0
sudo nmcli connection delete br0
sudo nmcli connection delete br0-slave-enp130s0
sudo nmcli connection add type ethernet ifname enp130s0 con-name "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

---

## Command Cheat Sheet

| What | Command |
|------|---------|
| List VMs | `virsh list --all` |
| Start VM | `virsh start debian13-vm0` |
| Stop VM | `virsh shutdown debian13-vm0` |
| Force stop | `virsh destroy debian13-vm0` |
| Delete VM | `virsh undefine debian13-vm0` |
| VM console | `virsh console debian13-vm0` |
| VM IP address | `virsh domifaddr debian13-vm0` |
| Minikube status | `minikube status -p k8s-cluster` |
| Minikube stop | `sudo minikube stop -p k8s-cluster` |
| Minikube delete | `sudo minikube delete -p k8s-cluster` |
| K8s nodes | `kubectl get nodes -o wide` |
| All pods | `kubectl get pods -A -o wide` |
| Pod logs | `kubectl logs <pod>` |
| Shell into pod | `kubectl exec -it <pod> -- /bin/sh` |
| Port forward | `kubectl port-forward svc/<name> 8080:80` |
| Podman test | `podman run --rm -it alpine echo hello` |
| Buildah build | `buildah from docker.io/library/alpine` |
| Skopeo inspect | `skopeo inspect docker://docker.io/library/nginx` |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                  Fedora 42 Host                           │
│                                                          │
│   ┌──────────────────────────────────────────────────┐   │
│   │          Debian 13 VM (debian13-vm0)              │   │
│   │                                                    │   │
│   │  ┌────────────────────────────────────────────┐  │   │
│   │  │     Minikube (podman driver, 3 nodes)       │  │   │
│   │  │                                              │  │   │
│   │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │  │   │
│   │  │  │ control- │ │ worker-  │ │ worker-  │    │  │   │
│   │  │  │  plane   │ │   m02    │ │   m03    │    │  │   │
│   │  │  └──────────┘ └──────────┘ └──────────┘    │  │   │
│   │  └────────────────────────────────────────────┘  │   │
│   │                                                    │   │
│   │  podman  buildah  skopeo  podman-compose  kubectl  │   │
│   └──────────────────────────────────────────────────┘   │
│                                                          │
│   br0 ──── enp130s0 ──── physical LAN                     │
└──────────────────────────────────────────────────────────┘
```

---

*Happy virtualizing. If something breaks, check the troubleshooting section.
If it's not there, check `journalctl`. If it's not there either, well —
that's what makes us all sysadmins.*
```

{{< comments >}}

