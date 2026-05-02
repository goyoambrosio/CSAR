# CSAR User Guide

This guide covers the day-to-day use of CSAR: connecting to the system, creating and managing containers, working with images, handling storage, and setting up ROS 2 communication.

---

## Accounts and Access

### Requesting an Account

A CSAR account must be requested through your laboratory's CSAR administrator. Depending on the available resources and your workload requirements, your account will be created on one of the CSAR servers (referred to here as `<server>`, e.g., `edge` or `uedge`).

Your account is a normal Unix user (no host-level `sudo`) but has full LXD permissions to create, manage, and delete your own containers, inside which you have full `sudo` privileges.

---

### SSH Access from Linux (Ubuntu 22.04/24.04)

Create or edit `~/.ssh/config` on your workstation:

```
Host <server>
    HostName <YOUR_SERVER_IP>
    Port 22
    User <your-csar-username>
    ForwardX11 yes
    Compression yes
    ServerAliveInterval 600
    ServerAliveCountMax 6
```

Copy the pre-generated private key from the server to your workstation:

```bash
scp <your-username>@<YOUR_SERVER_IP>:~/.ssh/roslab ~/.ssh/
```

Add the key to your SSH agent by appending to `~/.bashrc`:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/roslab
```

Then connect with:

```bash
ssh <server>
```

---

### SSH Access from Windows 10/11

Open PowerShell as administrator and enable the SSH agent service (once):

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

Copy the `roslab` private key to `C:\Users\<your-windows-user>\.ssh\` (e.g., using WinSCP), then add it:

```powershell
ssh-add $env:USERPROFILE\.ssh\roslab
```

Create `C:\Users\<your-windows-user>\.ssh\config` with the same content as the Linux example above.

Connect with:

```bash
ssh <server>
```

---

## Containers

CSAR's primary abstraction is the **system container**: a persistent, isolated Linux environment that shares the host kernel but has its own filesystem, network, and user space.

### The `lxc` and `csar` Commands

The main tools are `lxc` (standard LXD client) and `csar` (CSAR wrapper):

```bash
lxc launch     # create a container from an image
lxc list       # list your containers and their state
lxc stop       # shut down a container
lxc pause      # suspend a container
lxc start      # boot a container
lxc restart    # reboot a container
lxc delete     # delete a container (no confirmation prompt!)
lxc copy       # copy a container
lxc move       # move a container between pools, users, or servers
lxc exec       # run a command inside a container
```

```bash
csar free          # show available space in storage pools
csar containers    # show disk usage per container
csar launch        # create a container from a CSAR image
csar restart-dns   # restart the user's virtual DNS service
```

Use `lxc --help` and `csar --help` for full option lists.

---

### Creating Containers from CSAR Images

The recommended way to create containers is from CSAR pre-built images:

```bash
csar launch <server>:<image-name> <container-name>
```

For example:

```bash
csar launch <server>:tuxlab-jazzy-ros2-humble:2.1 myros2
```

The container is ready within seconds (CUDA images take slightly longer). Verify it is running:

```bash
lxc ls
```

---

### Accessing a Container

**From your CSAR account on the server:**

```bash
ssh -X ubuntu@<container>.<your-username>.<server>.mapir
```

**Directly from your workstation** (add to `~/.ssh/config`):

```
Host <container>
    HostName <container>.<your-username>.<server>.mapir
    Port 22
    User ubuntu
    ForwardX11 yes
    Compression yes
    ProxyJump <server>
    ServerAliveInterval 600
    ServerAliveCountMax 6
```

Then simply:

```bash
ssh <container>
```

---

### Updating a Container

Always update after creating a container from an image:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### Snapshots

Snapshots capture the full state of a container at a given moment.

```bash
# Create a snapshot
lxc snapshot <container> <snapshot-name>

# Restore a snapshot
lxc restore <container> <snapshot-name>

# List snapshots
lxc info <container>

# Delete a snapshot
lxc delete <container>/<snapshot-name>
```

Snapshots are especially useful before installing new software or drivers, when you want a safe fallback.

---

### Copying Containers to Other Users

To share a container with another user, first ask the CSAR administrator to grant cross-project copy permissions. Then:

```bash
lxc cp <my-container> <server>:<copy-name> \
    --target-project <target-user-project>
```

Each user's project name follows the pattern `user-<uid>`. You can check your own with:

```bash
echo user-$(id -u)
```

---

### Creating and Sharing Images

To publish a container as a reusable image:

```bash
lxc publish <container> --alias <image-name> --public --force
```

To copy the image to another user or to the shared server repository:

```bash
lxc image copy <image-name> <server>: \
    --target-project <default|user-<uid>> \
    --alias <new-image-name>
```

Use `--target-project default` to make the image available to all users on the server.

---

## Storage

Each CSAR server has multiple storage pools. The primary pool (NVMe, fast) is the default. A secondary pool (SATA, slower) is available for backups and low-priority containers.

```bash
csar free             # check available space in each pool
csar containers       # check disk usage per container

# Create a container in a specific pool
lxc launch <image> <container> -s <pool-name>

# Move a container to another pool (stop first)
lxc stop <container>
lxc move <container> -s <pool-name>

# Copy as backup to slow pool
lxc stop <container>
lxc cp <container> <container>-bak -s <backup-pool-name>
```

Containers in any pool are fully operational; those on the SATA pool simply run at lower I/O throughput.

---

## Shared Datasets

CSAR provides a shared storage area (`~/shares/datasets`) accessible from any container or user account. This avoids duplicating large datasets across individual user spaces.

Containers created from current `tuxlab` images have `~/shares` and `~/uploads` already mounted on login.

For older containers, mount the shared folder manually:

```bash
sudo apt install sshfs

# Add to ~/.bashrc:
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/roslab
mkdir -p ~/shares
sshfs ubuntu@shares.csar.<server>.mapir:/mnt/shares ~/shares
```

If the mount becomes stale after a disconnection:

```bash
fusermount -u ~/shares
fusermount -u ~/uploads
```

Then re-login or run `source ~/.bashrc`.

---

## Graphical Environment

All CSAR servers and containers built from CSAR images include an XFCE graphical desktop.

**From Linux:** Any graphical application launched from an SSH session with `ForwardX11 yes` opens directly on your local display. No additional setup needed.

**From Linux (full desktop via Remmina):**

```bash
sudo apt install remmina
```

Create an RDP connection to `<YOUR_SERVER_IP>` or to the container's FQDN.

**From Windows:** After establishing an SSH tunnel with `LocalForward 3388 localhost:3389`, open Remote Desktop to `localhost:3388`.

---

## Hybrid LXC/Docker Containers

For workflows that require Docker (e.g., pulling pre-built application images), CSAR provides hybrid containers that run Docker inside LXD with GPU access:

```bash
csar launch <server>:dockerlab-jazzy:1.1 <container-name>
lxc config set <container-name> security.nesting true
lxc restart <container-name>
```

Docker is available inside the container. Portainer (web UI) is accessible at:

```
https://<container>.<username>.<server>.mapir:9443
```

For remote access, add a port forward to `~/.ssh/config`:

```
LocalForward 9443 localhost:9443
```

To verify GPU access from Docker inside the container:

```bash
docker run --rm --gpus all nvidia/cuda:<version>-base-ubuntu<os-version> nvidia-smi
```

---

## Monitoring

CSAR servers include a monitoring stack (Prometheus + Loki + Grafana or NetData) running in a dedicated container.

Access from within the lab network:

```
http://metrics.<your-username>.<server>.mapir:3000    # Grafana
http://<monitor-container>.<server>.mapir/netdata     # NetData (uedge)
```

For remote access, add a local port forward to your SSH config:

```
LocalForward 3000 metrics.<your-username>.<server>.mapir:3000
```

Then open `http://localhost:3000` in your browser while the SSH session is active.
