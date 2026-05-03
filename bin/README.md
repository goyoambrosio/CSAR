# `csar` — CSAR Command-Line Tool

`csar` is a Bash wrapper around `lxc` and system tools that provides
CSAR-specific subcommands for storage inspection, container launching
with GPU support, DNS management, and hardware monitoring.

---

## Installation

Copy the script to a directory on your `PATH` and make it executable:

```bash
sudo cp csar /usr/local/bin/csar
sudo chmod +x /usr/local/bin/csar
```

Verify:

```bash
csar
```

---

## Dependencies

| Dependency | Required by | Install |
|------------|-------------|---------|
| `lxc` (LXD client) | `containers --run`, `launch` | pre-installed on CSAR servers |
| `zfs` | `free`, `containers` | pre-installed on CSAR servers |
| `nvidia-smi` | `gpu`, `tempgpu` | NVIDIA driver package |
| `sensors` | `tempcpu` | `sudo apt install lm-sensors` |
| `systemctl` | `restart-dns`, `restart-all-dns` | standard systemd |

---

## Subcommands

### `csar free`

Show available storage in each ZFS pool on the server.

```
$ csar free
pool1 available storage: 278G
pool2 available storage: 3.41T
```

---

### `csar containers [-r|--run]`

Without arguments: list all ZFS container datasets with compressed
and uncompressed sizes.

With `-r` or `--run`: list all currently running LXD containers
across all projects (admin view).

```bash
csar containers          # storage view
csar containers --run    # running containers view
```

---

### `csar launch <image> <container-name>`

Create a new LXD container from a CSAR image with GPU access
automatically configured (`nvidia.runtime=true` and a `gpu` device).

```bash
csar launch uedge:tuxlab-jazzy-ros2-humble-cuda:2.1 mycontainer
```

This is equivalent to:

```bash
lxc launch <image> <name> -c nvidia.runtime=true
lxc config device add <name> gpu gpu
```

---

### `csar restart-dns`

Restart the `lxd-dns-lxdbr-<uid>` systemd service for the **current
user**. Use this if container DNS resolution stops working after a
network change or system event.

```bash
csar restart-dns
```

---

### `csar restart-all-dns` *(admin only)*

Restart all active `lxd-dns-lxdbr-*` services on the server — i.e.,
the DNS service of every CSAR user simultaneously.

```bash
sudo csar restart-all-dns
```

> **Admin use only.** This affects all users on the server.

---

### `csar gpu [-c|--cmd]`

Show GPU memory usage broken down by LXD container and owner.
Reads active compute processes from `nvidia-smi` and resolves them
to their container via `/proc/<pid>/cgroup`.

```bash
csar gpu           # show PID, GPU memory, container, owner
csar gpu --cmd     # also show the full command line of each process
```

Example output:

```
PID=12345 GPU_MEM=4823MiB project=user-1003 container=c1 owner=alice(uid=1003)
PID=12346 GPU_MEM=9102MiB project=user-1007 container=slam owner=bob(uid=1007)
```

---

### `csar tempgpu [-w|--watch] [-i|--interval <sec>]`

Show a formatted table of GPU temperatures, VRAM temperature, fan
speed, power draw, and GPU/memory utilization for all GPUs on the host.

```bash
csar tempgpu                   # one-shot snapshot
csar tempgpu --watch           # refresh every 2 seconds
csar tempgpu --watch -i 5      # refresh every 5 seconds
```

Example output:

```
GPU Name                             Temp     VRAM     Fan      Power      GPU%     Mem%
0   NVIDIA RTX 6000 Ada Generation  42°C     --       30%      85W        12%      18%
1   NVIDIA RTX 6000 Ada Generation  38°C     --       30%      14W        0%       0%
2   NVIDIA RTX 6000 Ada Generation  40°C     --       30%      11W        0%       0%
```

---

### `csar tempcpu [-w|--watch] [-i|--interval <sec>]`

Show host CPU (Tctl) and NVMe composite temperatures using `lm-sensors`.

```bash
csar tempcpu                   # one-shot snapshot
csar tempcpu --watch           # refresh every 2 seconds
csar tempcpu --watch -i 10     # refresh every 10 seconds
```

Requires `lm-sensors`:

```bash
sudo apt install lm-sensors
sudo sensors-detect            # run once to detect hardware
```

---

## Notes

- `restart-all-dns` contains internal comments in Spanish — these are
  harmless and may be cleaned up in a future version.
- `tempcpu` uses `k10temp` (AMD) sensor naming. On Intel hosts the
  sensor block name will differ and the `awk` pattern may need
  adjustment.
- All monitoring subcommands (`gpu`, `tempgpu`, `tempcpu`) read
  hardware state directly and require no special privileges beyond
  standard user access to `nvidia-smi` and `sensors`.
