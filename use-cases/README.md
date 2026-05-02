# Use Cases

This directory contains documentation and configuration for the two use cases described in the CSAR paper.

Both use cases were evaluated on the MAPIR laboratory deployment of CSAR at the University of Málaga.

---

## Use Case 1 — Edge-Offloaded 3D SLAM over 5G

**Directory:** [`uc1-slam-5g/`](uc1-slam-5g/)

A ROS bag dataset is replayed on a remote laptop connected to the CSAR infrastructure via 5G and a secure overlay network. The computationally demanding SLAM processing (FAST-LIO) is offloaded to an edge server container. Visualization (RViz) runs in a separate container on the same server. The researcher controls the experiment remotely via SSH with X11 forwarding.

**Key CSAR capabilities demonstrated:**
- Layer 0: secure overlay network enabling WAN-transparent DDS communication
- Layer 1: isolated containers for acquisition, processing, and visualization
- Layer 2: edge-offloaded SLAM with GPU access

---

## Use Case 2 — GPU-Accelerated Distributed Semantic Mapping

**Directory:** [`uc2-semantic-mapping/`](uc2-semantic-mapping/)

A robot performs localization and data acquisition. Object detection (Detectron2) runs in a GPU-accelerated container on the edge server. A third container builds a probabilistic semantic map (Voxeland). Communication among all three components uses native ROS 2 topics and services.

**Key CSAR capabilities demonstrated:**
- Decomposition of a robotic pipeline across multiple containers
- Controlled GPU sharing across independent execution environments
- Reproducible multi-container deployment via CSAR images
