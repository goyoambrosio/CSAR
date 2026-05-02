# Changelog

All notable changes to this repository will be documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

---

## [1.0.0] — 2026-05-02

Initial public release, accompanying the paper submission:
> **CSAR: Containerized System Architecture for Robotics**
> Ambrosio-Cestero et al., MAPIR Group, University of Málaga.

### Added

- `README.md` — repository overview, layer summary, and getting started guide
- `CITATION.cff` — machine-readable citation for the paper
- `LICENSE` — Apache License 2.0
- `docs/architecture.md` — Layer 0/1/2 descriptions and ROS 2 communication model
- `docs/design-principles.md` — seven core conceptual principles of the framework
- `docs/glossary.md` — CSAR-specific terminology (framework, layers, containers, networking, hardware, ROS 2)
- `docs/networking.md` — per-user virtual networks, DNS architecture, secure overlay, DDS routing, MQTT
- `docs/user-guide.md` — end-user guide covering SSH access, containers, images, storage, and monitoring
- `docs/administration-guide.md` — administrator guide covering account lifecycle, DNS services, and monitoring stack
- `deployment/lxd/profiles/default.yaml` — per-user LXD profile template
- `deployment/lxd/profiles/gpu.yaml` — NVIDIA GPU passthrough profile template
- `deployment/lxd/profiles/gui.yaml` — X11 and PulseAudio forwarding profile template
- `deployment/networking/dns/lxdbr-ADDUID.yaml` — per-user LXD virtual bridge network template
- `deployment/networking/dns/lxd-dns-lxdbr-ADDUID.service` — systemd service template for per-user DNS registration
- `deployment/networking/dds/ddsrouter.service` — DDS Router systemd service template (server-side)
- `deployment/networking/dds/ddsrouter_server.yaml` — DDS Router config template (CSAR discovery container)
- `deployment/networking/dds/ddsrouter_client.yaml` — DDS Router config template (remote host / robot)
- `deployment/monitoring/prometheus/prometheus.yml` — Prometheus config template for LXD metrics
- `use-cases/uc1-slam-5g/README.md` — edge-offloaded 3D SLAM over 5G use case documentation
- `use-cases/uc2-semantic-mapping/README.md` — GPU-accelerated distributed semantic mapping use case documentation
- `paper/README.md` — paper reference and abstract

[Unreleased]: https://github.com/goyoambrosio/CSAR/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/goyoambrosio/CSAR/releases/tag/v1.0.0
