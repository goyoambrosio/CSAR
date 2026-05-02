# CSAR Per-User DNS Configuration Templates

This directory contains the two configuration artifacts that together make
container DNS resolution work for a CSAR user:

| File | What it is | Where it goes |
|------|-----------|---------------|
| `lxdbr-ADDUID.yaml` | LXD virtual bridge network config | Applied via `lxc network edit lxdbr-<ADDUID>` |
| `lxd-dns-lxdbr-ADDUID.service` | systemd service that registers the bridge with `systemd-resolved` | `/etc/systemd/system/lxd-dns-lxdbr-<ADDUID>.service` |

Both files must be created **once per user** by the CSAR administrator during
account setup. See [`docs/administration-guide.md`](../../../docs/administration-guide.md)
for the full procedure.

---

## How It Works

LXD creates a virtual bridge (`lxdbr-<ADDUID>`) for each user and runs an
embedded dnsmasq instance on it. This dnsmasq resolves container FQDNs within
the user's subnet (e.g. `c1.alice.uedge.mapir → 10.2.3.45`).

However, dnsmasq alone is not enough — the host's `systemd-resolved` does not
know about this bridge by default. The systemd service registers the bridge
gateway IP as the DNS server for the user's domain, so that queries for
`*.alice.uedge.mapir` from the host (or from containers on other bridges) are
forwarded to the right dnsmasq instance.

The dnsmasq `raw.dnsmasq` config in the network file handles three cases:

1. Queries for `*.<SERVER>.mapir` → forwarded to `127.0.0.53` (the local
   systemd-resolved stub, which knows about all registered per-user bridges
   on the same server)
2. Queries for `*.mapir` → forwarded to the CSAR router (which resolves
   cross-server names like `uedge.mapir` or `edge.mapir`)
3. All other queries → forwarded to the institution's DNS servers

---

## Quick Reference: Admin Steps

```bash
ADDUID=<uid>
USERNAME=<username>
SERVER=<server>         # edge or uedge
SERVER_ID=<1_or_2>      # 1 for edge, 2 for uedge
N=$(printf "%02d" $((ADDUID % 100)))

# 1. Create and configure the virtual bridge
lxc network create lxdbr-$ADDUID
cat lxdbr-ADDUID.yaml | \
  sed "s/<ADDUID>/$ADDUID/g; s/<N>/$N/g; s/<USERNAME>/$USERNAME/g; \
       s/<SERVER>/$SERVER/g; s/<SERVER_ID>/$SERVER_ID/g; \
       s/<DNS_SERVER_1>/<YOUR_DNS_1>/g; s/<DNS_SERVER_2>/<YOUR_DNS_2>/g; \
       s/<ROUTER_IP>/<YOUR_ROUTER_IP>/g" | \
  lxc network edit lxdbr-$ADDUID

# 2. Install the systemd service
sudo cp lxd-dns-lxdbr-ADDUID.service \
        /etc/systemd/system/lxd-dns-lxdbr-$ADDUID.service
sudo sed -i \
  "s/<ADDUID>/$ADDUID/g; s/<BRIDGE_IP>/10.$SERVER_ID.$N.1/g; \
   s/<USERNAME>/$USERNAME/g; s/<SERVER>/$SERVER/g" \
  /etc/systemd/system/lxd-dns-lxdbr-$ADDUID.service

# 3. Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now lxd-dns-lxdbr-$ADDUID.service

# 4. Verify
sudo systemctl status lxd-dns-lxdbr-$ADDUID.service
resolvectl status lxdbr-$ADDUID
```

Expected output from `resolvectl status lxdbr-$ADDUID`:

```
Link N (lxdbr-<ADDUID>)
    Current Scopes: DNS
         Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.<SERVER_ID>.<N>.1
       DNS Servers: 10.<SERVER_ID>.<N>.1
        DNS Domain: ~<USERNAME>.<SERVER>.mapir
```
