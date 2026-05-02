# CSAR Administration Guide

This guide covers CSAR system administration tasks: creating and deleting user accounts, managing per-user networking, setting up monitoring, and remote power management.

> **Note:** This guide assumes you are the CSAR administrator with host-level `sudo` access on the servers. Regular CSAR users should refer to the [User Guide](user-guide.md).

---

## Creating a User Account

The following procedure applies to a server configured with LXD using per-user projects and per-user virtual bridges. Adapt the network prefix (`10.2.x.x`) and DNS domain (`uedge.mapir`) for your server.

### Step 1 — Create the Unix user

```bash
USERNAME="<new-username>"
sudo adduser $USERNAME
sudo adduser $USERNAME csar    # use 'lxd' group on some configurations
```

Record the new user's UID:

```bash
ADDUID=$(id -u "$USERNAME")
echo $ADDUID
N=$(printf "%02d" $((ADDUID % 100)))
```

### Step 2 — Initialize the LXD project

Log in once as the new user to trigger project creation, then configure it:

```bash
su $USERNAME
lxc ls
exit

lxc project set user-$ADDUID restricted.snapshots allow
```

### Step 3 — Create the user's virtual network

```bash
lxc network create lxdbr-$ADDUID
lxc network set lxdbr-$ADDUID ipv4.address 10.2.$N.1/24
lxc network set lxdbr-$ADDUID dns.domain "$USERNAME.uedge.mapir"
lxc network unset lxdbr-$ADDUID ipv6.nat
lxc network unset lxdbr-$ADDUID ipv6.address
lxc network set lxdbr-$ADDUID raw.dnsmasq "
server=/uedge.mapir/127.0.0.53
server=/mapir/<YOUR_CSAR_ROUTER_IP>
server=<YOUR_DNS_SERVER_1>
server=<YOUR_DNS_SERVER_2>
domain-needed
bogus-priv
no-hosts
no-negcache
no-poll
"
```

Review the result:
```bash
lxc network edit lxdbr-$ADDUID
```

### Step 4 — Create the DNS systemd service

Copy and adapt an existing service unit:

```bash
sudo cp /etc/systemd/system/lxd-dns-lxdbr-<EXISTING_UID>.service \
        /etc/systemd/system/lxd-dns-lxdbr-$ADDUID.service
sudo vi /etc/systemd/system/lxd-dns-lxdbr-$ADDUID.service
```

Replace all occurrences of the existing UID, IP, and username with the new values. The service template looks like this:

```ini
[Unit]
Description=LXD per-link DNS configuration for lxdbr-<ADDUID>
BindsTo=sys-subsystem-net-devices-lxdbr\x2d<ADDUID>.device
After=sys-subsystem-net-devices-lxdbr\x2d<ADDUID>.device

[Service]
Type=oneshot
ExecStart=/usr/bin/resolvectl dns lxdbr-<ADDUID> 10.2.<N>.1
ExecStart=/usr/bin/resolvectl domain lxdbr-<ADDUID> '~<USERNAME>.uedge.mapir'
ExecStopPost=/usr/bin/resolvectl revert lxdbr-<ADDUID>
RemainAfterExit=yes

[Install]
WantedBy=sys-subsystem-net-devices-lxdbr\x2d<ADDUID>.device
```

### Step 5 — Enable and start the DNS service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lxd-dns-lxdbr-$ADDUID.service
sudo systemctl status lxd-dns-lxdbr-$ADDUID
resolvectl status lxdbr-$ADDUID
```

### Step 6 — Set the default storage pool

```bash
lxc profile device set default root pool=<DEFAULT_POOL_NAME> --project user-$ADDUID
lxc profile show default --project user-$ADDUID
```

### Step 7 — Add the user to the LXD trust list

```bash
lxc config trust add
# When prompted for a name, enter: remote-user-$ADDUID
```

Log in as the new user and add the remote:

```bash
ssh $USERNAME@<server>
lxc remote add <server> https://127.0.0.1:8443
# Press y and paste the token from the previous step
exit
```

Back as admin, restrict the trust entry:

```bash
lxc config trust list
# Find the fingerprint for remote-user-$ADDUID
lxc config trust edit <fingerprint>
# Change: restricted: false  ->  restricted: true
```

### Step 8 — Verify the account

```bash
lxc launch ubuntu:22.04 c1 -t aws:t2.micro --project user-$ADDUID
lxc ls --project user-$ADDUID
ping c1.$USERNAME.<server>.mapir
```

---

## Deleting a User Account

```bash
USERNAME="<username-to-delete>"
DELUID=$(id -u "$USERNAME")

# Remove Unix user
sudo deluser $USERNAME --remove-home

# Clean up LXD resources
lxc project switch user-$DELUID
lxc list
lxc delete <instance> --force    # repeat for all instances
lxc image list
lxc image delete <fingerprint>   # repeat for all images
lxc project delete user-$DELUID

# Remove the virtual network
lxc network list
lxc network delete lxdbr-$DELUID

# Remove from trust list
lxc config trust list
lxc config trust remove <fingerprint>

# Remove LXD user state
sudo rm -rf /var/snap/lxd/common/lxd-user/users/$DELUID

# Remove DNS service
sudo rm /etc/systemd/system/lxd-dns-lxdbr-${DELUID}.service
sudo systemctl daemon-reload
```

---

## Monitoring Stack

CSAR uses a monitoring container with Prometheus (LXD metrics), Loki (log aggregation), and Grafana (dashboards).

### LXD Metrics Endpoint

Enable the LXD metrics API on the host:

```bash
lxc config set core.metrics_address :8444
```

Create a metrics container and generate certificates:

```bash
lxc launch ubuntu:22.04 metrics
lxc exec metrics -- bash -c "snap install prometheus"
lxc exec metrics -- bash -c "
  openssl req -x509 -newkey ec \
    -pkeyopt ec_paramgen_curve:secp384r1 -sha384 \
    -keyout metrics.key -nodes -out metrics.crt -days 3650 \
    -subj '/CN=metrics.local'
"
```

Add the Prometheus certificate to the LXD trust list:

```bash
lxc file pull metrics/root/metrics.crt - | \
  lxc config trust add metrics.crt --type=metrics --name prometheus -
```

### Prometheus Configuration

Inside the metrics container, append to `/var/snap/prometheus/current/prometheus.yml`:

```yaml
- job_name: lxd
  metrics_path: '/1.0/metrics'
  scheme: 'https'
  static_configs:
    - targets: ['<YOUR_EDGE_HOST>:8444']
  tls_config:
    insecure_skip_verify: true
    ca_file: 'tls/server.crt'
    cert_file: 'tls/metrics.crt'
    key_file: 'tls/metrics.key'
```

Restart Prometheus:

```bash
snap restart prometheus
```

### Loki

```bash
# Inside the metrics container:
apt-get update
apt-get install loki promtail

# On the LXD host:
lxc config set loki.api.url=http://metrics.<your-user>.<server>.mapir:3100
```

### Grafana

```bash
# Inside the metrics container:
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
  sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Access at `http://metrics.<your-user>.<server>.mapir:3000` (or via SSH tunnel from a remote machine).

---

## Remote Power Management

If your server supports Wake-on-LAN, you can power it on remotely from another machine on the same network:

```bash
wakeonlan <SERVER_MAC_ADDRESS>
```

From RouterOS (MikroTik):

```
/tool wol interface=<INTERFACE> mac=<SERVER_MAC_ADDRESS>
```

If your server has an IPMI/BMC interface, you can open a tunnel and access the web interface:

```bash
ssh -f -N -L 54443:<YOUR_BMC_IP>:54443 <YOUR_ROUTER_USER>@<YOUR_ROUTER_IP>
# Then open https://localhost:54443 in your browser
```

---

## CSAR `csar` Command Reference

The `csar` command is a CSAR-specific wrapper around `lxc`. Key subcommands:

| Command | Description |
|---------|-------------|
| `csar launch <server>:<image> <name>` | Create a container from a CSAR image |
| `csar free` | Show available space per storage pool |
| `csar containers` | Show disk usage per container |
| `csar restart-dns` | Restart the user's virtual DNS service |

For a full list: `csar --help`
