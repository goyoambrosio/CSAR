# CSAR Glossary

This glossary defines the terminology used throughout the CSAR framework documentation. Definitions are grounded in the CSAR paper and the reference deployment at the MAPIR laboratory.

---

## Framework Terms

**CSAR (Containerized System Architecture for Robotics)**
An architectural framework designed for multi-user robotics teams and the edge–cloud continuum. CSAR provides an operating model for shared robotics infrastructure by organizing computation into hardware-affine, persistent execution environments. It is not a product or platform — it is a set of architectural principles and layer definitions that can be implemented with different underlying tools.

**Infrastructure Core → [Layer 0](#layer-0--infrastructure-core)**

**Platform & Multi-User Orchestration → [Layer 1](#layer-1--platform--multi-user-orchestration)**

**Compute & Acceleration → [Layer 2](#layer-2--compute--acceleration)**

**Hardware affinity**
The explicit binding of a container or workload to specific physical hardware (e.g., a particular GPU, a PCIe device, a storage pool). CSAR treats hardware affinity as a first-class architectural concern rather than something left implicit or resolved at runtime.

**Persistent execution environment**
A container that is intended to accumulate state over time — software installations, configurations, datasets — rather than being recreated from scratch for each task. This is the primary model for user workspaces in CSAR, in contrast to the ephemeral, stateless containers typical of web-service architectures.

**Multi-user operating model**
The organizational principle by which a single physical infrastructure is shared among multiple users with isolated, independently managed execution environments. Each user has their own network segment, storage quota, DNS domain, and container project.

**Tenant**
A CSAR user. Each tenant has an isolated project within LXD, a dedicated virtual network bridge, and a personal DNS subdomain.

---

## Layers

### **Layer 0 — Infrastructure Core**
The stable, conservative substrate of the CSAR system. Responsible for host OS management, storage pools, core networking (routing, DNS, NTP), long-lived services (overlay network controller, image server, MQTT broker), and hardware driver management. Changes at this layer are infrequent and require administrator intervention. A failure at Layer 0 makes the entire multi-user ecosystem unavailable.

### **Layer 1 — Platform & Multi-User Orchestration**
The execution fabric built on LXC/LXD system containers. Provides per-user project isolation, per-user virtual bridge networks, container lifecycle management (launch, stop, snapshot, copy, move), GPU and device passthrough, cross-host container portability, and DDS discovery and routing services. Changes at this layer affect individual users but do not destabilize the infrastructure.

### **Layer 2 — Compute & Acceleration**
The dynamic workload layer where ROS 2 nodes, robotic processing pipelines, and GPU-accelerated processes execute. This layer is intentionally disposable: a broken container at Layer 2 can be deleted and recreated from a snapshot or image without affecting the layers below. The key property of Layer 2 is near-native hardware access with full isolation.

---

## Containerization

**System container**
A container that encapsulates a complete Linux environment (filesystem, processes, network, users) while sharing the host kernel. System containers are the primary abstraction in CSAR and are implemented using LXC/LXD. Unlike application containers, they are persistent, stateful, and provide a full OS environment suitable for multi-user interactive development.

**Application container**
A container that encapsulates a single process or application along with its minimal dependencies (e.g., Docker, Podman). CSAR supports application containers within hybrid LXC/Docker containers, but the primary model for user workspaces is the system container.

**LXC (Linux Containers)**
The Linux kernel-level technology (namespaces, cgroups) that underlies system containers. Provides isolation at the process, filesystem, network, and user level without hardware virtualization.

**LXD**
The system container and virtual machine manager developed by Canonical that builds on LXC. Provides the management API, image server, storage pool management, network management, and project-based multi-user isolation used in CSAR.

**LXD project**
A LXD-level isolation boundary that separates the containers, images, and storage of one CSAR user from those of others. Each CSAR user has their own LXD project, named `user-<uid>`.

**LXD profile**
A reusable configuration template that defines resource limits, device mappings (e.g., GPU passthrough), and network attachments for containers. Profiles are applied at container creation time and can be updated without recreating the container.

**Hybrid container (LXC/Docker)**
A CSAR system container that runs Docker internally. This allows users to pull and run application container images (e.g., from Docker Hub) while remaining integrated in the CSAR virtual network and retaining GPU access through the LXD layer.

**Image (CSAR image)**
A pre-built, versioned container template stored in the CSAR image server. CSAR images include a complete OS, NVIDIA drivers, SSH configuration, shared folder mounts, and optionally CUDA, ROS 2, or other software stacks. Creating a container from a CSAR image takes seconds and produces a fully configured environment.

**Snapshot**
A point-in-time capture of a container's state. Snapshots allow users to roll back to a known-good configuration after failed experiments, driver installations, or software upgrades. CSAR encourages taking snapshots before any destructive operation.

**Storage pool**
A named storage allocation managed by LXD. CSAR deployments typically use at least two pools: a fast NVMe pool for active container execution and a slower SATA pool for backups and infrequently used containers.

---

## Networking

**Virtual bridge (`lxdbr-<uid>`)**
A per-user Linux bridge created by LXD that forms an isolated Layer 2 broadcast domain for all containers belonging to that user. Each CSAR user has one virtual bridge per server, with a dedicated IP subnet.

**FQDN (Fully Qualified Domain Name)**
The full DNS name of a CSAR container, following the pattern `<container>.<username>.<server>.mapir`. For example: `c1.alice.uedge.mapir`. Resolvable within the CSAR network and from any device connected to the CSAR router.

**dnsmasq**
The DNS and DHCP server embedded in LXD. Each user's virtual bridge has its own dnsmasq instance, which resolves container FQDNs within the user's subnet and assigns IP addresses dynamically.

**Secure Network Overlay (SNO)**
A virtual private network built on top of the physical or internet infrastructure, implemented using ZeroTier running as a private controller on the CSAR router. The overlay allows geographically distributed devices (containers, laptops, robots) to communicate as if they were on the same local network, with end-to-end encryption and explicit node authorization. This is the mechanism that enables WAN-transparent ROS 2 communication in use case 1.

**ZeroTier**
The open-source peer-to-peer networking software used to implement the CSAR Secure Network Overlay. Runs as a private controller on the CSAR router, without dependency on ZeroTier's public infrastructure.

**DDS (Data Distribution Service)**
The publish-subscribe middleware standard used by ROS 2 for inter-node communication. DDS handles discovery, serialization, QoS, and transport. CSAR's networking is designed to accommodate DDS's broadcast-domain assumptions across segmented networks.

**DDS Router**
A software component (eProsima DDS Router) that bridges DDS traffic across different network domains or transports (e.g., from a local UDP domain to a WAN TCP connection). Used in CSAR to enable ROS 2 communication between nodes on the CSAR infrastructure and remote nodes connected via the secure overlay.

**Discovery Server**
A ROS 2 / DDS mechanism that replaces the default Simple Discovery Protocol (SDP) with a centralized discovery service. Required when ROS 2 nodes are distributed across multiple network segments with different broadcast domains. In CSAR, a dedicated Discovery Server container runs as a service on the primary server.

**ROS_DISCOVERY_SERVER**
An environment variable that points a ROS 2 node to the address of the Discovery Server. Set in CSAR containers when communication must cross user subnet boundaries or WAN links.

**ROS_DOMAIN_ID**
A ROS 2 environment variable (integer 0–232) that restricts DDS communication to nodes sharing the same domain. In CSAR, users are encouraged to set a non-zero domain ID to avoid unintended cross-user topic collisions. The recommended value is the last three digits of the user's UID.

---

## Hardware

**edge**
The lab-facing edge server in the MAPIR reference deployment. Hosts LXD with Docker support, physical robots (Sancho, Hunter) integrated as LXD nodes, and provides the primary point of presence for the laboratory network.

**uedge**
The high-performance server in the MAPIR reference deployment, equipped with a AMD Threadripper PRO processor, 512 GB RAM, and multiple NVIDIA RTX 6000 Ada GPUs. The primary server for GPU-accelerated workloads.

**magician**
The MikroTik router in the MAPIR reference deployment. Handles Layer 0 networking: physical routing, DNS, WiFi, and the ZeroTier-based Secure Network Overlay controller.

**GPU passthrough**
The mechanism by which LXD exposes a physical GPU directly to a container using LXC device configuration. Containers with GPU passthrough achieve near-native GPU performance, making them suitable for CUDA workloads, deep learning inference, and GPU-accelerated SLAM.

---

## ROS 2 / Robotics

**ROS 2 (Robot Operating System 2)**
The second generation of the Robot Operating System, a set of software libraries and tools for building robotic applications. CSAR is designed around ROS 2 as the communication and computation framework for robotic workloads at Layer 2.

**ROS bag**
A file format for recording and replaying ROS 2 message streams. Used in CSAR use case 1 to emulate live sensor data from the Hunter platform during the SLAM experiment.

**SLAM (Simultaneous Localization and Mapping)**
The computational problem of building a map of an environment while simultaneously tracking the agent's location within it. In CSAR use case 1, FAST-LIO is used for 3D SLAM processing on LiDAR and odometry data.

**FAST-LIO**
A computationally efficient LiDAR-inertial odometry algorithm used in CSAR use case 1 for real-time 3D mapping on the edge server.

**Detectron2**
A deep learning-based object detection and segmentation framework (Facebook AI Research). Used in CSAR use case 2 for GPU-accelerated object detection as part of the semantic mapping pipeline.

**Voxeland**
A probabilistic instance-aware semantic mapping framework developed at MAPIR. Used in CSAR use case 2 to build semantic maps of indoor environments from ROS 2 topics.

**Hunter 2.0**
A wheeled mobile robot platform used in the MAPIR laboratory. Integrated into CSAR as a physical node with LXD system containers running on board.
