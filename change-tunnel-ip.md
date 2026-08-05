# When Tunnel Remote IP Changes

Runbook when switching GRE remote endpoint (e.g. `185.237.84.75` → `87.107.94.241`).

Server IP and WireGuard configs do **not** change.

---

## Variables

| Variable | Example | Description |
|----------|---------|-------------|
| `SERVER_IP` | `85.133.162.63` | VPS public IP (unchanged) |
| `OLD_TUNNEL_IP` | `185.237.84.75` | Current remote endpoint |
| `NEW_TUNNEL_IP` | `87.107.94.241` | New remote endpoint |
| `PRIVATE_IP` | `172.17.50.114/30` | Usually unchanged |
| `TUNNEL_PEER` | `172.17.50.113` | Other side of /30 |

> Confirm with provider that the new remote IP is configured for your tunnel.

---

## 1. Recreate GRE Tunnel (live)

```bash
sudo ip link set gre1 down && sudo ip link delete gre1
```

```bash
sudo ip tunnel add gre1 mode gre remote <NEW_TUNNEL_IP> local <SERVER_IP> ttl 255
```

```bash
sudo ip addr add <PRIVATE_IP> dev gre1
```

```bash
sudo ip link set gre1 up
```

---

## 2. Test

```bash
ip tunnel show gre1
ping -c 3 <TUNNEL_PEER>
ip route get 8.8.8.8 from 10.8.0.2 iif wg2
```

If `ping <TUNNEL_PEER>` fails → provider has not configured the new remote IP yet.

---

## 3. Update Reboot Script

```bash
sudo sed -i 's/remote <OLD_TUNNEL_IP>/remote <NEW_TUNNEL_IP>/' /usr/local/bin/setup-gre-tunnel.sh
grep "ip tunnel add" /usr/local/bin/setup-gre-tunnel.sh
```

Example:

```bash
sudo sed -i 's/remote 185.237.84.75/remote 87.107.94.241/' /usr/local/bin/setup-gre-tunnel.sh
```

---

## What Does NOT Change

| Component | Change? |
|-----------|---------|
| `wg2.conf` / `wg3.conf` | No |
| Client configs / Endpoint | No |
| WGDashboard | No |
| iptables / routing tables | No |
| `SERVER_IP` (local) | No |

---

## Related Docs

- [setup.md](./setup.md) — initial setup
- [change-server-ip.md](./change-server-ip.md) — when VPS public IP changes
