# Deployment Templates

This directory contains configuration templates for deploying a CSAR infrastructure. All templates use placeholder values (e.g., `<YOUR_EDGE_HOST>`, `<YOUR_SERVER_IP>`) that must be replaced with your own values before use.

## Structure

```
deployment/
├── lxd/
│   └── profiles/        LXD profile YAML templates
├── networking/
│   ├── dns/             Per-user virtual bridge and DNS service templates
│   └── dds/             DDS Router and Discovery Server configs
└── monitoring/
    └── prometheus/      Prometheus config for LXD metrics
```

## Before You Start

1. Read [`docs/architecture.md`](../docs/architecture.md) to understand what each component does.
2. Read [`docs/administration-guide.md`](../docs/administration-guide.md) for the step-by-step account creation procedure.
3. Replace all `<PLACEHOLDER>` values in the templates with your environment's actual values.

## Placeholder Convention

All placeholders follow the pattern `<DESCRIPTION_IN_CAPS>`. Replace every occurrence before use:

| Placeholder | Replace with |
|-------------|-------------|
| `<YOUR_EDGE_HOST>` | Hostname or IP of your edge server |
| `<YOUR_UEDGE_HOST>` | Hostname or IP of your high-performance server |
| `<USERNAME>` | CSAR username (e.g. `alice`) |
| `<ADDUID>` | Unix UID of the user (e.g. `1003`) |
| `<N>` | Numeric value of the last three UID digits used for IP addressing (e.g. UID 1003 → `3`) |
| `<SERVER>` | Server name (`edge` or `uedge`) |
| `<SERVER_ID>` | Server IP octet (`1` for edge, `2` for uedge) |
| `<DEFAULT_POOL_NAME>` | LXD storage pool name (e.g. `default`) |
| `<HOST_UID>` | Unix UID of the host user running X11/PulseAudio (output of `id -u`) |
| `<DNS_SERVER_1>` | Institution primary DNS server IP |
| `<DNS_SERVER_2>` | Institution secondary DNS server IP |
| `<ROUTER_IP>` | CSAR router IP for `.mapir` DNS resolution |
| `<ROUTER_PUBLIC_IP>` | CSAR router public IP (WAN-facing) |
| `<BRIDGE_IP>` | Virtual bridge IP for the user's subnet (e.g. `10.1.3.1`) |
| `<CONTAINER_IP>` | Internal IP of a specific container |
| `<WAN_PORT>` | WAN-facing TCP port for the DDS Router |
| `<DS_ID_1>` | First DDS Discovery Server ID for this container (e.g. `1`) |
| `<DS_ID_2>` | Second DDS Discovery Server ID for this container (e.g. `2`) |
| `<ROS_DOMAIN_ID>` | ROS 2 domain ID used by the nodes |
| `<VERSION>` | Software version tag (e.g. for the DDS Router Docker image) |
| `<NETWORK_ID>` | ZeroTier network ID |
