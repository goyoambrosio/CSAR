# CSAR DDS Router Configuration Templates

This directory contains configuration templates for the eProsima DDS Router
as deployed in CSAR to bridge ROS 2 / DDS traffic across heterogeneous
network segments and WAN links.

## Files

| File | Role | Where it runs |
|------|------|---------------|
| `ddsrouter.service` | systemd service unit | Inside the CSAR discovery container |
| `ddsrouter_server.yaml` | DDS Router config — server side | Inside the CSAR discovery container |
| `ddsrouter_client.yaml` | DDS Router config — client side | On the remote host (laptop, robot) |

---

## Architecture

```
[Remote host]                        [CSAR edge server]
  ┌─────────────────┐                ┌──────────────────────────────────┐
  │ ddsrouter       │  TCP over WAN  │ discovery container              │
  │ (client config) │ ─────────────▶ │ ddsrouter (server config)        │
  │                 │                │   WAN repeater                   │
  │ ROS_DOMAIN_ID=N │                │   + Discovery Server (UDP + TCP) │
  └────────┬────────┘                └──────────────┬───────────────────┘
           │ DDS (local)                            │ DDS (internal network)
    [ROS 2 nodes on                         [ROS 2 nodes in
     remote host]                            CSAR containers]
```

The CSAR router (Layer 0) has a NAT rule that forwards the WAN port to the
discovery container's internal IP. Remote nodes connect to the public IP,
and the DDS Router bridges their topics transparently into the CSAR network.

---

## Server-Side Setup (Administrator)

### 1. Create the discovery container

```bash
csar launch <server>:tuxlab-noble-vulcanexus-jazzy-desktop:2.0 discovery
ssh ubuntu@discovery.csar.<server>.mapir
```

### 2. Test the DDS Router manually first

```bash
cp ddsrouter_server.yaml ~/ddsrouter_conf.yaml
# Edit ~/ddsrouter_conf.yaml and replace all placeholders
/opt/vulcanexus/jazzy/bin/ddsrouter -c ~/ddsrouter_conf.yaml
# Ctrl+C when satisfied
```

### 3. Install and enable the service

```bash
sudo cp ddsrouter.service /etc/systemd/system/ddsrouter.service
sudo systemctl daemon-reload
sudo systemctl enable --now ddsrouter.service
sudo systemctl status ddsrouter.service
```

### 4. Add the NAT rule on the CSAR router

On the MikroTik router (RouterOS), add a destination NAT rule:

```
/ip firewall nat add chain=dstnat protocol=tcp \
  dst-port=<WAN_PORT> action=dst-nat \
  to-addresses=<CONTAINER_IP> to-ports=<WAN_PORT>
```

---

## Client-Side Setup (User / Remote Host)

### 1. Install Docker

See [Docker installation docs](https://docs.docker.com/engine/install/ubuntu/).

### 2. Load the DDS Router image

```bash
docker load -i ubuntu-ddsrouter-<VERSION>.tar
```

### 3. Configure and run

```bash
mkdir -p ~/dds_router_ws
cp ddsrouter_client.yaml ~/dds_router_ws/DDS_ROUTER_CONFIGURATION.yaml
# Edit the file and replace all placeholders

docker run -it --net=host \
  -v ~/dds_router_ws/DDS_ROUTER_CONFIGURATION.yaml:/root/DDS_ROUTER_CONFIGURATION.yaml \
  ubuntu-ddsrouter:<VERSION> ddsrouter -c /root/DDS_ROUTER_CONFIGURATION.yaml
```

### 4. Set the ROS environment in a separate terminal

```bash
export ROS_DOMAIN_ID=<YOUR_DOMAIN_ID>
# Run your ROS 2 nodes normally — they will discover CSAR nodes transparently
```

---

## Port Allocation Convention

In the CSAR reference deployment, each discovery container uses a dedicated
WAN port to avoid conflicts. Use a registry like this:

| Container | WAN port | Internal IP | DS IDs |
|-----------|----------|-------------|--------|
| `discovery.csar.<server>` | `<PORT_0>` | `<IP_0>` | 1, 2 |
| `discovery-<name1>.csar.<server>` | `<PORT_1>` | `<IP_1>` | 10, 11 |
| `discovery-<name2>.csar.<server>` | `<PORT_2>` | `<IP_2>` | 20, 21 |

Keep this table in your internal CSAR administration notes.
