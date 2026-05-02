# Use Case 2 — GPU-Accelerated Distributed Semantic Mapping

## Overview

This use case demonstrates a multi-container robotic pipeline for distributed semantic mapping. Three containers run simultaneously on the CSAR edge server, each performing a distinct role: localization and data acquisition on the robot, GPU-accelerated object detection, and probabilistic semantic map building.

The scenario validates CSAR's ability to decompose a complex robotic application across independent execution environments while sharing GPU resources and maintaining transparent ROS 2 communication.

---

## Architecture

```
[Robot — Hunter 2.0]                  [Edge Server — uedge]
  ┌────────────────────┐              ┌────────────────────────┐
  │ Container 1        │              │ Container 2            │
  │ Localization       │──ROS 2──────▶│ Object Detection       │
  │ Data Acquisition   │              │ (Detectron2 + GPU)     │
  │ (OpenCV, ROS 2)    │              └────────────┬───────────┘
  └────────────────────┘                           │ ROS 2
                                        ┌──────────▼───────────┐
                                        │ Container 3          │
                                        │ Semantic Map         │
                                        │ (Voxeland + GPU)     │
                                        └──────────────────────┘
```

Communication among all containers uses native ROS 2 topics and services. No bridging or proxy is required because all containers are on the same CSAR infrastructure.

---

## CSAR Layers Involved

| Layer | Role in this use case |
|-------|-----------------------|
| Layer 0 | GPU drivers and CUDA available on the host; per-user network bridging containers to robot |
| Layer 1 | Three isolated containers, each with independent software environments |
| Layer 2 | ROS 2 nodes: localization, Detectron2 object detection, Voxeland map builder |

---

## Requirements

**Robot:**
- Ubuntu 22.04 with ROS 2 Humble
- LiDAR and RGB-D camera
- CSAR account or overlay network access

**Edge server (CSAR):**
- CSAR account on `uedge` (or equivalent)
- Image: `tuxlab-jazzy-ros2-humble-cuda:2.x` (containers 2 and 3)
- Image: `tuxlab-jazzy-ros2-humble:2.x` (container 1, if deployed on server)
- At least 2× NVIDIA GPU (containers 2 and 3 can share or use separate GPUs)

---

## Setup

### 1. Create the containers

```bash
csar launch <server>:tuxlab-jazzy-ros2-humble:2.1      localization-container
csar launch <server>:tuxlab-jazzy-ros2-humble-cuda:2.1 detection-container
csar launch <server>:tuxlab-jazzy-ros2-humble-cuda:2.1 mapping-container
```

### 2. Install the required packages

**In `detection-container`:**
```bash
# Install Detectron2 and ROS 2 wrapper
# Follow Detectron2 installation: https://github.com/facebookresearch/detectron2
pip install detectron2 ...
```

**In `mapping-container`:**
```bash
# Install Voxeland
# https://github.com/MAPIRlab/Voxeland
```

### 3. Set the ROS environment

In all containers:

```bash
export ROS_DOMAIN_ID=<YOUR_ROS_DOMAIN_ID>
```

If the robot is external to the server network, also set:

```bash
export ROS_DISCOVERY_SERVER=discovery.csar.<server>.mapir
```

### 4. Run the pipeline

**In `localization-container`** (or on the robot directly):
```bash
ros2 launch <your_localization_pkg> localization.launch.py
```

**In `detection-container`:**
```bash
ros2 launch <your_detection_pkg> detectron2_node.launch.py
```

**In `mapping-container`:**
```bash
ros2 launch voxeland voxeland.launch.py
```

---

## Notes on GPU Sharing

Both `detection-container` and `mapping-container` access the GPU through the CSAR infrastructure. The GPU is passed through to the containers at the LXD level; no additional configuration is required inside the containers beyond standard CUDA usage.

CSAR does not currently enforce GPU memory quotas. Coordinate with other users to avoid resource conflicts during intensive experiments.

---

## Results Summary

This use case demonstrates that CSAR supports multi-container pipelines where GPU-accelerated workloads are distributed across independent execution environments without modifying the application code. The pipeline produces probabilistic semantic maps of real indoor environments.

Quantitative results are reported in the paper (Section 5.2).
