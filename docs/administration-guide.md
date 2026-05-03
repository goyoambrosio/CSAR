# CSAR Administration Guide

**Containerized System Architecture for Robotics**

Gregorio Ambrosio Cestero (`gambrosio@uma.es`)

> This guide covers CSAR system administration tasks. It assumes you are
> the CSAR administrator with host-level `sudo` access on the servers.
> Regular CSAR users should refer to the
> [Reference Guide](reference-guide.md) instead.

---

## Table of Contents

1. [Creating a User Account](#1-creating-a-user-account)
2. [Deleting a User Account](#2-deleting-a-user-account)

---

## 1. Creating a User Account

The following procedure applies to `uedge`. Adapt the network prefix
(`10.2.x.x`) and DNS domain (`uedge.mapir`) for `edge` where noted.

### Step 1 — Create the Unix user account

```bash
USERNAME="<new-username>"

sudo adduser $USERNAME
sudo adduser $USERNAME csar    # use 'lxd' instead of 'csar' on edge
```

Record the new user's UID and compute the subnet index `N`:

```bash
ADDUID=$(id -u "$USERNAME")
echo $ADDUID

N=$(printf "%02d" $((ADDUID % 100)))
echo $N
```

Log in once as the new user to trigger LXD project initialization,
then exit immediately:

```bash
su $USERNAME
lxc ls
exit
```

### Step 2 — Configure the LXD project

```bash
lxc project set user-$ADDUID restricted.snapshots allow
lxc project show user-$ADDUID    # review — no changes needed here
```

### Step 3 — Create the user's virtual network and DNS domain

```bash
lxc network create lxdbr-$ADDUID
lxc network set lxdbr-$ADDUID ipv4.address 10.2.$N.1/24
# On edge use: 10.1.$N.1/24

lxc network set lxdbr-$ADDUID dns.domain "$USERNAME.uedge.mapir"
# On edge use: "$USERNAME.edge.mapir"

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
# On edge, replace the first server line with:
# server=/edge.mapir/127.0.0.53
```

Review the resulting network configuration and fix the description
if it has a doubled prefix (e.g., `user-user-` → `user-`):

```bash
lxc network edit lxdbr-$ADDUID
```

### Step 4 — Create the systemd DNS service for the user

Copy an existing service file and edit it for the new user.
**Important:** the substitution commands below must use literal values,
not shell variables — `systemd` unit files do not expand them.

```bash
sudo cp /etc/systemd/system/lxd-dns-lxdbr-<EXISTING_UID>.service \
        /etc/systemd/system/lxd-dns-lxdbr-$ADDUID.service

sudo vi /etc/systemd/system/lxd-dns-lxdbr-$ADDUID.service
```

Inside `vi`, replace the old values with the new ones:

```
:%s/<EXISTING_UID>/<NEW_ADDUID>/g
:%s/10.2.<EXISTING_N>.1/10.2.<NEW_N>.1/g
:%s/<OLD_USERNAME>.uedge.mapir/<NEW_USERNAME>.uedge.mapir/g
:wq
```

The resulting service file should look like this:

```ini
[Unit]
Description=LXD per-link DNS configuration for lxdbr-<ADDUID>

BindsTo=sys-subsystem-net-devices-lxdbr\x2d<ADDUID>.device
After=sys-subsystem-net-devices-lxdbr\x2d<ADDUID>.device

[Service]
Type=oneshot
ExecStart=/usr/bin/resolvectl dns    lxdbr-<ADDUID> 10.2.<N>.1
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

# If you edit the service file later, restart with:
# sudo systemctl restart lxd-dns-lxdbr-$ADDUID.service

sudo systemctl status lxd-dns-lxdbr-$ADDUID
resolvectl status lxdbr-$ADDUID
```

Expected output from `resolvectl`:

```
Link N (lxdbr-<ADDUID>)
    Current Scopes: DNS
         Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.2.<N>.1
       DNS Servers: 10.2.<N>.1
        DNS Domain: ~<USERNAME>.uedge.mapir
```

### Step 6 — Set the default storage pool

```bash
lxc profile device set default root pool=<DEFAULT_POOL_NAME> \
    --project user-$ADDUID

lxc profile show default --project user-$ADDUID    # verify
```

### Step 7 — Add the user to the LXD trust list

As the CSAR administrator, generate a trust token:

```bash
lxc config trust add
# When prompted for a name, enter: remote-user-<ADDUID>
# Copy the token that is printed.
```

Log in as the new user and add the remote:

```bash
ssh $USERNAME@<server>
lxc remote add <server> https://127.0.0.1:8443
# Press y when prompted, then paste the token from the previous step.
exit
```

Back as the CSAR administrator, restrict the trust entry:

```bash
lxc config trust list
# Find the fingerprint for remote-user-<ADDUID>

lxc config trust edit <fingerprint>
# Change: restricted: false
# To:     restricted: true
```

### Step 8 — Verify the account

Run a quick sanity check from the administrator account:

```bash
lxc launch ubuntu:22.04 c1 -t aws:t2.micro --project user-$ADDUID
lxc ls --project user-$ADDUID
ping c1.$USERNAME.uedge.mapir
```

Then log in as the new user and run a final check:

```bash
ssh $USERNAME@<server>

lxc remote list
lxc image list <server>:
lxc ls
csar launch <server>:tuxlab-jazzy:1.1 t1
ssh -X ubuntu@t1.$(whoami).uedge.mapir
ls ~/shares
ls ~/uploads
exit
```

If all steps produce the expected output, the account is ready.

---

## 2. Deleting a User Account

Log in as the CSAR administrator before starting.

```bash
USERNAME="<username-to-delete>"
DELUID=$(id -u "$USERNAME")
```

### Step 1 — Remove the Unix user account

```bash
sudo deluser $USERNAME --remove-home
```

### Step 2 — Delete all LXD resources in the user's project

```bash
lxc project switch user-$DELUID

lxc list
lxc delete <instance> --force    # repeat for every instance in the list

lxc image list
lxc image delete <FINGERPRINT>   # repeat for every image in the list

lxc project delete user-$DELUID
```

### Step 3 — Delete the user's virtual network

```bash
lxc network list
lxc network delete lxdbr-$DELUID
```

### Step 4 — Remove from the LXD trust list

```bash
lxc config trust list
# Find the fingerprint for remote-user-<DELUID>

lxc config trust remove <FINGERPRINT>
```

### Step 5 — Remove LXD user state

```bash
sudo ls -al /var/snap/lxd/common/lxd-user/users/
sudo rm -rf /var/snap/lxd/common/lxd-user/users/$DELUID
```

### Step 6 — Remove the DNS systemd service

```bash
sudo rm /etc/systemd/system/lxd-dns-lxdbr-${DELUID}.service
sudo systemctl daemon-reload
```

The user and all their associated resources have been fully removed.

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

