# Use Case 1 — Edge-Offloaded 3D SLAM over 5G

## Overview

This use case demonstrates edge-offloaded 3D SLAM using FAST-LIO, where the data source (LiDAR + odometry replayed from a ROS bag) is physically separated from the processing infrastructure and connected only through 5G and a secure overlay network.

The scenario validates CSAR's ability to sustain a distributed robotic workload in which sensing, processing, and visualization execute in independent containers across heterogeneous network segments, with ROS 2 communication handled transparently by the lower layers of the framework.

---

## Architecture

```
[Remote laptop]             [5G / Secure Overlay]        [Edge Server — uedge]
  ┌──────────────┐                                       ┌───────────────────────┐
  │ Container:   │  ──── ROS 2 (DDS over overlay) ────▶ │ Container:            │
  │ ROS bag      │                                       │ FAST-LIO SLAM         │
  │ replay       │                                       │ (GPU access)          │
  └──────────────┘                                       └────────┬──────────────┘
                                                                  │ ROS 2 topics
                                                         ┌────────▼──────────────┐
                                                         │ Container:            │
                                                         │ RViz visualization    │
                                                         │ (X11 → researcher)    │
                                                         └───────────────────────┘
```

The researcher controls the experiment via SSH from the remote laptop and receives the graphical output through X11 forwarding.

---

## CSAR Layers Involved

| Layer | Role in this use case |
|-------|-----------------------|
| Layer 0 | Secure overlay network (ZeroTier) connecting 5G laptop to CSAR; DDS Router configuration |
| Layer 1 | Three isolated containers: replay, SLAM processing, visualization |
| Layer 2 | FAST-LIO node with GPU access; RViz with X11 forwarding |

---

## Requirements

**Remote host (laptop):**
- Ubuntu 22.04 or 24.04
- Docker (for DDS Router)
- ROS 2 Humble or later
- 5G modem or equivalent WAN connection
- ZeroTier client

**Edge server (CSAR):**
- CSAR account on `uedge` (or equivalent)
- Image: `tuxlab-jazzy-ros2-humble-cuda:2.1` (for SLAM container)
- Image: `tuxlab-jazzy-ros2-humble:2.1` (for visualization container)
- GPU available (any NVIDIA with CUDA 12.x)

---

## Setup

### 1. Join the overlay network

On the remote laptop:

```bash
sudo snap install zerotier
sudo zerotier-cli join <YOUR_OVERLAY_NETWORK_ID>
# Ask the CSAR admin to authorize this node
```

### 2. Configure the DDS Router on the remote laptop

Create `~/dds_router_ws/DDS_ROUTER_CONFIGURATION.yaml` (see [`deployment/networking/dds/ddsrouter_client.yaml`](../../deployment/networking/dds/ddsrouter_client.yaml)):

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

Run the DDS Router:

```bash
docker run -it --net=host \
  -v ~/dds_router_ws/DDS_ROUTER_CONFIGURATION.yaml:/root/DDS_ROUTER_CONFIGURATION.yaml \
  ubuntu-ddsrouter:<VERSION> ddsrouter -c /root/DDS_ROUTER_CONFIGURATION.yaml
```

### 3. Create the CSAR containers

On the CSAR server:

```bash
csar launch <server>:tuxlab-jazzy-ros2-humble-cuda:2.1 slam-container
csar launch <server>:tuxlab-jazzy-ros2-humble:2.1      rviz-container
```

### 4. Set the ROS environment

In all containers and on the remote laptop:

```bash
export ROS_DOMAIN_ID=<YOUR_ROS_DOMAIN_ID>
export ROS_DISCOVERY_SERVER=discovery.csar.<server>.mapir
```

### 5. Run the experiment

**On the remote laptop** (replay container or directly):
```bash
ros2 bag play <YOUR_ROSBAG_PATH>
```

**In the SLAM container** on the edge server:
```bash
ros2 launch fast_lio mapping.launch.py
```

**In the RViz container** on the edge server:
```bash
rviz2 -d <YOUR_RVIZ_CONFIG>
```

The researcher accesses RViz output via X11 forwarding over the SSH session.

---

## Dataset

The dataset used in the paper consists of LiDAR and odometry recordings from the Hunter 2.0 mobile platform, equipped with an Ouster OS1-32 LiDAR. It covers approximately 1 km of trajectory around the Computer Science School at the University of Málaga and is non-trivial both in spatial scale and data volume, representative of bandwidth-demanding field deployments.

The dataset is publicly available on Zenodo:

> Anaya Palacios, F., Galindo, C., González-Jiménez, J. (2025).
> *Mobile Robot Dataset with Ouster OS1-32 LiDAR at the University of Málaga.*
> Zenodo. https://doi.org/10.5281/zenodo.15301791

---

## Results Summary

The main result of this use case is architectural: CSAR sustains a distributed robotic workload in which (i) sensing, processing, and visualization execute in separate containers, (ii) communication spans heterogeneous network segments, and (iii) ROS 2-based applications require no modification to operate across the WAN.

Quantitative results are reported in the paper (Section 5.1).
