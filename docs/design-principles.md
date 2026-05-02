# CSAR Design Principles

This document describes the core conceptual principles behind CSAR. Understanding these principles helps when adapting the framework to new environments or extending it beyond the reference deployment.

---

## 1. Containers as Persistent Environments, Not Packaging Artifacts

The dominant use of containers in software engineering treats them as ephemeral, stateless units: a container is created, runs a process, and is discarded. This model works well for web services and CI/CD pipelines.

CSAR rejects this model for robotics infrastructure. A CSAR container is a **persistent execution environment** — the equivalent of a personal workstation, but one that lives on shared infrastructure. It accumulates software, configuration, data, and state over time. It can be snapshotted, backed up, cloned, and transferred, but it is not expected to be frequently recreated from scratch.

This choice has direct consequences: we use **system containers** (LXC/LXD) rather than application containers (Docker), because system containers encapsulate a full Linux environment and are designed for persistence.

---

## 2. Hardware Affinity as a First-Class Concern

In a robotics laboratory, not all computation is equal. Some workloads require GPUs. Others require low-latency access to sensors or specific PCIe devices. Cloud-oriented infrastructure typically abstracts hardware away; CSAR does the opposite.

CSAR makes **hardware affinity explicit**: containers are placed on specific servers, bound to specific hardware resources, and the framework manages this placement deliberately rather than hiding it. The layer structure (Layer 0 → hardware, Layer 1 → containers, Layer 2 → workloads) reflects this: hardware resources are stable and long-lived; workloads are volatile and disposable.

---

## 3. Separation of Stability and Volatility

A research laboratory infrastructure must remain stable while experimental work proceeds. Experiments fail, configurations break, and software conflicts arise. These events should not propagate upward and destabilize the shared infrastructure.

CSAR achieves this by the strict separation between layers:

- **Layer 0** (kernel, storage, networking) changes rarely and conservatively.
- **Layer 1** (containers) provides isolation so that one user's broken environment does not affect others.
- **Layer 2** (ROS 2 nodes, GPU workloads) is explicitly disposable: a broken Layer 2 container can be deleted and recreated from a snapshot or image without touching the infrastructure below.

---

## 4. Multi-User by Design

Most container infrastructure is designed for a single user or a single application. CSAR is designed from the ground up for **multiple concurrent users** sharing physical resources.

This means:
- Each user has isolated storage, networking, and DNS
- GPU and device access is mediated, not freely shared
- Cross-user communication is explicit and controlled (via Discovery Server or DDS Router), not implicit
- Administrative actions (account creation, image publishing, cross-user copies) require administrator involvement

---

## 5. Infrastructure Extends ROS 2 Downward

ROS 2 defines a communication and computation model for robotic systems. CSAR extends this model downward into the infrastructure level: rather than leaving it implicit which machine runs which node, which network segment it belongs to, and which hardware it can access, CSAR makes these concerns explicit architectural decisions.

In this sense, CSAR provides an architectural framework that mediates between computation, resources, time, and physical locality.

---

## 6. Reproducibility as an Infrastructure Property

Reproducibility in robotics research is usually addressed at the software level (Docker images, version-pinned dependencies, CI). CSAR treats reproducibility as an **infrastructure property** as well:

- Container images are versioned and stored in a shared image server
- Snapshots provide point-in-time recovery
- The layer structure ensures that the environment in which an experiment runs is stable and can be recreated on another machine (by exporting and importing container images)
- The networking model ensures that communication topology is explicit and documented, not implicitly inherited from the host

---

## 7. CSAR is a Framework, Not a Platform

CSAR does not prescribe a specific vendor, orchestration system, or product. It is an **architectural framework**: a set of principles, layer definitions, and design decisions that can be implemented with different underlying tools.

The reference implementation uses LXD, ZeroTier, MikroTik, Prometheus, and Grafana — but these are implementation choices, not essential parts of the framework. Other implementations could use different container runtimes, different overlay networks, or different monitoring stacks, as long as the layer structure and design principles are respected.
