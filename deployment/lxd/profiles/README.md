# LXD Profile Templates

This directory contains annotated LXD profile templates for CSAR deployments.

## Profiles

| File | Purpose | Applied by |
|------|---------|-----------|
| `default.yaml` | Root disk and network interface for every user container | Administrator, per user |
| `gpu.yaml` | NVIDIA GPU passthrough for CUDA workloads | User, on demand |
| `gui.yaml` | X11 display and PulseAudio forwarding for graphical applications | User, on demand |

## How Profiles Work in CSAR

Profiles are applied at container creation time and can also be added to running containers (after a restart). Multiple profiles can be stacked:

```bash
# Container with GPU access
lxc launch <server>:<image> <container> --profile default --profile gpu

# Container with GUI and GPU
lxc launch <server>:<image> <container> --profile default --profile gpu --profile gui

# Add a profile to an existing container
lxc profile add <container> gpu
lxc restart <container>
```

The `default` profile is applied automatically by CSAR when using `csar launch`. The `gpu` and `gui` profiles must be added explicitly.

## Applying a Profile from a File

```bash
lxc profile create gpu
cat gpu.yaml | lxc profile edit gpu
```

Or edit directly:

```bash
lxc profile edit gpu
```

## Verifying GPU Access

After applying the `gpu` profile and restarting the container:

```bash
# Inside the container
nvidia-smi
```

You should see the GPUs available on the host.

## Notes

- All `<PLACEHOLDER>` values must be replaced before use. See [`deployment/README.md`](../README.md) for the placeholder convention.
- The `gui` profile is primarily useful for local desktop setups. For remote graphical access, CSAR containers use SSH X11 forwarding (`ForwardX11 yes` in `~/.ssh/config`), which requires no additional profile.
- The `default` profile is managed per user by the administrator and is scoped to the user's LXD project (`--project user-<uid>`).
