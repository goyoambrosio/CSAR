# CSAR Architecture

This document describes the three-layer architecture of CSAR and the design rationale behind each layer.

## Overview

CSAR organizes a shared robotics infrastructure as a layered multi-user system. Rather than treating a laboratory server as a collection of personal machines, CSAR structures it so that computation is placed explicitly, hardware affinity is a first-class concern, and failures are contained without compromising overall system stability.

The framework distinguishes three functional layers:

```
┌─────────────────────────────────────────────────────┐
│  Layer 2 — Compute & Acceleration                   │
│  ROS 2 nodes · GPU workloads · experimental pipelines│
├─────────────────────────────────────────────────────┤
│  Layer 1 — Platform & Multi-User Orchestration      │
│  LXC/LXD containers · isolation · DNS · portability │
├─────────────────────────────────────────────────────┤
│  Layer 0 — Infrastructure Core                      │
│  Host OS · storage pools · physical networking      │
└─────────────────────────────────────────────────────┘
```

---

## Layer 0 — Infrastructure Core

Layer 0 is the stable, deliberately conservative substrate of the system. Its role is to provide a predictable and long-lived foundation on which all other layers depend.

**Responsibilities:**
- Host OS management (Ubuntu LTS, kernel updates)
- Storage pool management (NVMe primary pools, SATA backup pools)
- Physical networking (uplink, routing, DNS forwarding)
- Long-lived services: overlay network controller, MQTT broker, image server
- Hardware driver management (NVIDIA drivers, CUDA)

**Design principle:** Layer 0 changes infrequently. Stability takes priority over novelty. Experimental work should never require changes at this layer.

---

## Layer 1 — Platform & Multi-User Orchestration

Layer 1 is the execution fabric. It is built on Linux system containers (LXC/LXD) and provides each user with an isolated, persistent, portable workspace.

**Responsibilities:**
- Per-user project isolation (LXD projects)
- Per-user virtual bridge networks (`lxdbr-<uid>`)
- Per-user DNS domains (`<container>.<user>.<server>.mapir`)
- Container lifecycle management (launch, stop, snapshot, copy, move)
- Cross-host container portability (image publishing and sharing)
- GPU and device passthrough to containers
- Hybrid LXC/Docker containers for application container workflows

**Key design decision:** Containers in CSAR are treated as **persistent execution environments**, not as ephemeral packaging artifacts. A user's container is their workstation — it persists across sessions, accumulates configuration, and can be snapshotted, backed up, or shared.

**Per-user network model:**

Each user on each server has a dedicated subnet:

```
Server    CIDR
edge      10.1.<uid_last3>.0/24
uedge     10.2.<uid_last3>.0/24
```

Container FQDNs follow the pattern:
```
<container>.<username>.<server>.mapir
```

For example: `c1.alice.uedge.mapir`

---

## Layer 2 — Compute & Acceleration

Layer 2 is the dynamic workload layer. This is where ROS 2 nodes, robotic processing pipelines, and GPU-accelerated processes run.

**Responsibilities:**
- ROS 2 graph execution (nodes, topics, services, actions)
- DDS discovery and communication (within a user domain, across users, across WAN)
- GPU-accelerated workloads (SLAM, object detection, semantic mapping)
- Experimental pipelines, algorithm tuning, dataset processing

**Key design decision:** Layer 2 is intentionally **disposable**. Containers at this layer can be created, destroyed, snapshotted, and replaced without affecting the layers below. This enables safe prototyping: a user can break a container without affecting other users or the infrastructure.

---

## ROS 2 / DDS Communication Model

CSAR supports four communication scenarios with increasing scope:

| Scenario | Scope | Mechanism |
|----------|-------|-----------|
| Single container | Intra-host | Standard ROS 2 / DDS (SDP) |
| Multiple containers, same user | Same broadcast domain | Standard ROS 2 / DDS (SDP) |
| Multiple users on same server | Cross-subnet | Discovery Server (`discovery.csar.<server>.mapir`) |
| Remote nodes (WAN) | Cross-network | DDS Router over secure overlay |

For cross-user communication, set:
```bash
export ROS_DISCOVERY_SERVER=discovery.csar.<server>.mapir
```

For WAN communication, a DDS Router container bridges the remote node to the CSAR domain. See [`deployment/networking/dds/`](../deployment/networking/dds/) for configuration templates.

---

## Reference Images

CSAR provides pre-built LXD images that users can instantiate in seconds:

| Image | Base OS | Contents |
|-------|---------|----------|
| `tuxlab-jazzy:1.1` | Ubuntu 22.04 | NVIDIA drivers, XFCE, utilities |
| `tuxlab-jazzy-cuda:1.2` | Ubuntu 22.04 | + CUDA, cuDNN, PyTorch, OpenCV |
| `tuxlab-jazzy-ros2-humble:2.1` | Ubuntu 22.04 | + ROS 2 Humble full desktop |
| `tuxlab-jazzy-ros2-humble-cuda:2.1` | Ubuntu 22.04 | + ROS 2 Humble + CUDA |
| `discovery-jazzy-ros2-humble:1.1` | Ubuntu 22.04 | Discovery Server as a service |
| `dockerlab-jazzy:1.1` | Ubuntu 22.04 | Hybrid LXC/Docker with GPU access |
| `mosquitto-jazzy:1.0` | Ubuntu 22.04 | MQTT broker |
| `tuxlab-noble:2.0` | Ubuntu 24.04 | Base development image |
| `tuxlab-noble-vulcanexus-jazzy-desktop:2.0` | Ubuntu 24.04 | Vulcanexus (eProsima ROS 2) |

To create a container from an image:
```bash
csar launch <server>:<image-name> <container-name>
```
