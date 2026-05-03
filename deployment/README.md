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

All placeholders follow the pattern `<DESCRIPTION_IN_CAPS>`. For example:

| Placeholder | Replace with |
|-------------|-------------|
| `<YOUR_EDGE_HOST>` | The hostname or IP of your edge server |
| `<YOUR_UEDGE_HOST>` | The hostname or IP of your high-performance server |
| `<YOUR_CSAR_ROUTER_IP>` | The IP of your router |
| `<YOUR_DNS_SERVER_1>` | Your institution's primary DNS server |
| `<USERNAME>` | A specific CSAR username |
| `<ADDUID>` | The Unix UID of the user |
| `<NETWORK_ID>` | A ZeroTier network ID |
