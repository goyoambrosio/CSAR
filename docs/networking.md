# CSAR Networking Model

This document describes the networking architecture of CSAR: the per-user virtual networks, DNS resolution, the secure overlay (ZeroTier), and the DDS routing configuration for ROS 2 across heterogeneous networks.

---

## Physical Devices

A typical CSAR deployment involves the following physical devices:

| Role | Example hostname | Notes |
|------|-----------------|-------|
| Router / overlay controller | `<YOUR_ROUTER_HOST>` | MikroTik or equivalent; manages physical and virtual networking |
| Primary compute server | `<YOUR_UEDGE_HOST>` | High-performance server (GPU, large RAM) |
| Edge server | `<YOUR_EDGE_HOST>` | Lab-facing server |
| Mobile robots | `<YOUR_ROBOT_HOST>` | Optional; integrated via LXD |

Public-facing names are registered in the institution's DNS (e.g., `<host>.isa.uma.es`). Internal names are resolved by the CSAR router.

---

## Per-User Virtual Networks

Each user on each CSAR server has a dedicated virtual bridge (`lxdbr-<uid>`) managed by LXD. This bridge forms an isolated broadcast domain for that user's containers.

**IP addressing scheme:**

```
Server     User subnet
<server1>  10.1.<uid_last3>.0/24
<server2>  10.2.<uid_last3>.0/24
```

The user's virtual router is always at `10.<server_id>.<uid_last3>.1`.

For example, a user with UID 1003 on the second server has subnet `10.2.3.0/24`, and their containers receive addresses in the range `10.2.3.2` – `10.2.3.254` via DHCP.

---

## Container Naming (FQDN)

Every container receives a fully qualified domain name (FQDN) automatically:

```
<container-name>.<username>.<server>.mapir
```

Example: `c1.alice.uedge.mapir`

This name is resolvable from within the user's virtual network and from the CSAR router (which bridges between servers).

---

## DNS Architecture

CSAR uses a three-level DNS resolution chain:

1. **LXD dnsmasq (per-user):** Resolves container FQDNs within the user's subnet. Each user has their own dnsmasq instance. Handles DHCP and short-name resolution.

2. **Host OS DNS:** Resolves external names (e.g., `github.com`, `ubuntu.com`) via the institution's DNS servers.

3. **CSAR router DNS:** Resolves static cross-server names (e.g., `uedge.mapir` → `<YOUR_UEDGE_IP>`) and passes unknown queries to the institution's DNS. This is the critical level that allows containers on different servers to find each other by name.

---

## Secure Network Overlay

CSAR provides a **secure overlay network** service that connects distributed devices (containers, laptops, robots) over the internet as if they were on the same local network. This enables transparent ROS 2 discovery (SDP) across geographically separated nodes.

The overlay is based on [ZeroTier](https://github.com/zerotier/ZeroTierOne) running as a **private controller** on the CSAR router. No dependency on ZeroTier's public infrastructure.

**Key properties:**
- End-to-end encrypted point-to-point communication
- Each node requires explicit authorization by the controller
- IP addresses are assigned within a defined virtual subnet (e.g., `192.168.x.0/24`)
- A device can belong to multiple overlay networks simultaneously
- Local subnet routes can be advertised into the overlay (cross-network routing)

### Joining an Overlay Network

On Ubuntu (or inside a CSAR container):

```bash
sudo snap install zerotier
sudo zerotier-cli join <NETWORK_ID>
```

Then notify the CSAR administrator to authorize your node.

### Available Networks (Template)

Replace the table below with your own network IDs:

| Network ID | IP Range | Purpose |
|------------|----------|---------|
| `<NETWORK_ID_1>` | `192.168.250.0/24` | General CSAR overlay |
| `<NETWORK_ID_2>` | `192.168.249.0/24` | User/project-specific |

---

## ROS 2 / DDS Routing

### Scenario 1: Single container or same-user containers

Standard ROS 2 DDS discovery (SDP) works transparently within the user's subnet. No extra configuration needed.

### Scenario 2: Cross-user communication on the same server

Use the shared Discovery Server:

```bash
export ROS_DISCOVERY_SERVER=discovery.csar.<server>.mapir
```

Optionally set a non-zero `ROS_DOMAIN_ID` to avoid cross-user topic collisions:

```bash
export ROS_DOMAIN_ID=<last_3_digits_of_your_uid>
```

### Scenario 3: Remote nodes over WAN

For nodes outside the CSAR network (e.g., a laptop at home, a robot on 5G), deploy a DDS Router on the remote host:

**Install Docker on the remote host:**
```bash
# Standard Docker installation — see https://docs.docker.com/engine/install/ubuntu/
```

**Load the DDS Router image:**
```bash
docker load -i ubuntu-ddsrouter-v3.2.0.tar
```

**Create `DDS_ROUTER_CONFIGURATION.yaml`:**
```yaml
version: v5.0

participants:
  - name: ROS_2_LAN
    kind: local
    domain: <YOUR_ROS_DOMAIN_ID>

  - name: Router_Client
    kind: wan
    connection-addresses:
      - ip: <YOUR_CSAR_ROUTER_PUBLIC_IP>
        port: 8883
        transport: tcp
```

**Run the router:**
```bash
docker run -it --net=host \
  -v /home/ubuntu/dds_router_ws/DDS_ROUTER_CONFIGURATION.yaml:/root/DDS_ROUTER_CONFIGURATION.yaml \
  ubuntu-ddsrouter:v3.2.0 ddsrouter -c /root/DDS_ROUTER_CONFIGURATION.yaml
```

### Note on CycloneDDS (ROS 2 Humble)

If you encounter out-of-memory errors with FastDDS on ROS 2 Humble / Ubuntu 22.04:

```bash
sudo apt install ros-humble-rmw-cyclonedds-cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

---

## WiFi

The CSAR router provides WiFi in the lab on two bands. Devices connected via WiFi receive addresses in the `10.10.10.0/24` range and can communicate with containers and physical devices on the CSAR network. Contact the CSAR administrator for credentials.

---

## MQTT Broker

CSAR provides a shared MQTT broker (Mosquitto) accessible on the network at the router's address.

- Port `1883`: username/password authentication
- Port `8883`: TLS/SSL (certificates available from the broker container)

Example (unsecured):
```bash
mosquitto_sub -h <YOUR_CSAR_ROUTER_HOST> -u <MQTT_USER> -P <MQTT_PASSWORD> -t test
mosquitto_pub -h <YOUR_CSAR_ROUTER_HOST> -p 1883 -u <MQTT_USER> -P <MQTT_PASSWORD> -t test -m "hello"
```

For TLS access, copy the CA and client certificates from the broker container, then use port `8883` with `--cafile`, `--cert`, and `--key` options.
