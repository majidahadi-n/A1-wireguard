# GRE Tunnel + WireGuard Setup

Initial setup for a VPS that routes WireGuard client traffic through a GRE tunnel.

## Architecture

```
WireGuard clients (10.8.0.x / 10.9.0.x)
        ↓
   wg2 / wg3 (host, wg-quick)
        ↓
      gre1 (GRE tunnel)
        ↓
   Tunnel provider (remote endpoint)
        ↓
     Internet
```

| Component | Role |
|-----------|------|
| `ppp0` / public IP | Server internet access |
| `gre1` | GRE tunnel to provider |
| `wg2` / `wg3` | WireGuard VPN interfaces (host) |
| Docker `wg-dashboard` | Web UI only (optional, for config/QR export) |

> VPN runs on the **host** via `wg-quick`. Docker is only for the dashboard UI.

---

## Variables

Replace these before running commands:

| Variable | Example | Description |
|----------|---------|-------------|
| `SERVER_IP` | `85.133.162.63` | VPS public IP (local GRE endpoint) |
| `TUNNEL_IP` | `185.237.84.75` | Remote GRE endpoint (from provider) |
| `PRIVATE_IP` | `172.17.50.114/30` | GRE tunnel private IP (from provider) |
| `TUNNEL_PEER` | `172.17.50.113` | Other side of the /30 (usually .113) |

---

## 1. Prerequisites

```bash
sudo apt update
sudo apt install -y wireguard wireguard-tools iptables-persistent
```

Enable IP forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
```

Optional — disable reverse path filtering (if routing issues occur):

```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
```

---

## 2. Create GRE Tunnel

```bash
sudo ip tunnel add gre1 mode gre remote <TUNNEL_IP> local <SERVER_IP> ttl 255
sudo ip addr add <PRIVATE_IP> dev gre1
sudo ip link set gre1 up
```

Verify:

```bash
ip tunnel show gre1
ip -br addr show gre1
ping -c 3 <TUNNEL_PEER>
```

Expected: `0% packet loss` to the tunnel peer.

---

## 3. WireGuard (host)

Configs live at `/srv/wg-dashboard/config/` (or `/etc/wireguard/`).

Example layout:

| Interface | Subnet | Listen port | Routing table |
|-----------|--------|-------------|---------------|
| `wg2` | `10.8.0.0/24` | `51281` | `202` |
| `wg3` | `10.9.0.0/24` | `51282` | `203` |

Bring up:

```bash
sudo wg-quick up /srv/wg-dashboard/config/wg2.conf
sudo wg-quick up /srv/wg-dashboard/config/wg3.conf
```

Verify:

```bash
sudo wg show
ip route show table 202
ip route show table 203
ip route get 8.8.8.8 from 10.8.0.2 iif wg2
```

Expected routing: `dev gre1 table 202` (or `203` for wg3).

---

## 4. Firewall / NAT

WireGuard `PostUp` in config files should handle most rules. Minimum required:

```bash
# Forward between WG and GRE
sudo iptables -A FORWARD -i wg2 -o gre1 -j ACCEPT
sudo iptables -A FORWARD -i gre1 -o wg2 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wg3 -o gre1 -j ACCEPT
sudo iptables -A FORWARD -i gre1 -o wg3 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# NAT out via tunnel
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o gre1 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 10.9.0.0/24 -o gre1 -j MASQUERADE
```

Open UDP ports on cloud firewall:

| Port | Interface |
|------|-----------|
| `51281/udp` | wg2 |
| `51282/udp` | wg3 |

---

## 5. Reboot Script

Create `/usr/local/bin/setup-gre-tunnel.sh`:

```bash
sudo tee /usr/local/bin/setup-gre-tunnel.sh << 'EOF'
#!/bin/bash

# Destroy broken tunnel
ip link delete gre1 2>/dev/null || true

# Build GRE tunnel
ip tunnel add gre1 mode gre remote <TUNNEL_IP> local <SERVER_IP> ttl 255
ip addr add <PRIVATE_IP> dev gre1
ip link set gre1 up

# Clean duplicate NAT rules
while iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o gre1 -j MASQUERADE 2>/dev/null; do true; done
while iptables -t nat -D POSTROUTING -s 10.9.0.0/24 -o gre1 -j MASQUERADE 2>/dev/null; do true; done
while iptables -t nat -D POSTROUTING -o gre1 -j MASQUERADE 2>/dev/null; do true; done

# Restart WireGuard
wg-quick down /srv/wg-dashboard/config/wg2.conf 2>/dev/null || true
wg-quick down /srv/wg-dashboard/config/wg3.conf 2>/dev/null || true
wg-quick up /srv/wg-dashboard/config/wg2.conf
wg-quick up /srv/wg-dashboard/config/wg3.conf

exit 0
EOF

sudo chmod +x /usr/local/bin/setup-gre-tunnel.sh
```

Add to root crontab:

```bash
sudo crontab -e
```

```
@reboot sleep 60 && /usr/local/bin/setup-gre-tunnel.sh
```

> The 60s delay waits for `ppp0` / network to come up before creating the tunnel.

---

## 6. WGDashboard (optional — UI only)

```yaml
# /srv/wg-dashboard/docker-compose.yaml
services:
  wg-dashboard:
    network_mode: host
    image: donaldzou/wgdashboard:v4.3.3
    container_name: wg-dashboard
    restart: "no"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - public_ip=<SERVER_IP>
      - dynamic_config=false
    volumes:
      - ./config:/etc/wireguard
      - ./data:/data
    devices:
      - /dev/net/tun:/dev/net/tun
```

Set peer endpoint IP in ini file:

```bash
sudo sed -i 's/^remote_endpoint = .*/remote_endpoint = <SERVER_IP>/' /srv/wg-dashboard/data/wg-dashboard.ini
```

Start dashboard when needed:

```bash
cd /srv/wg-dashboard && sudo docker compose up -d
```

Stop when done:

```bash
sudo docker compose down
```

Client configs are generated with:

```ini
Endpoint = <SERVER_IP>:51281   # wg2
Endpoint = <SERVER_IP>:51282   # wg3
```

> `docker compose down` does **not** stop VPN — WireGuard runs on the host.

---

## 7. Post-Setup Tests

```bash
# Tunnel
ping -c 3 <TUNNEL_PEER>

# WG routing
ip route get 8.8.8.8 from 10.8.0.2 iif wg2

# Internet via tunnel (temporary host route)
sudo ip route add 1.1.1.1 dev gre1
ping -c 3 -I gre1 1.1.1.1
sudo ip route del 1.1.1.1 dev gre1

# WG status
sudo wg show
```

### Do NOT use these as tunnel health checks

These fail by design (default route goes via `ppp0`, not `gre1`):

```bash
ping -I gre1 1.1.1.1          # wrong test
curl --interface gre1 https://google.com   # wrong test
```

WG client traffic uses policy routing (tables 202/203), not the main routing table.

---

## 8. After Reboot

No manual action needed. Cron runs `setup-gre-tunnel.sh` after 60 seconds.

Quick check:

```bash
ip -br link | grep -E 'gre1|wg2|wg3'
sudo wg show
ping -c 2 <TUNNEL_PEER>
```

---

## Related Docs

- [change-tunnel-ip.md](./change-tunnel-ip.md) — switch GRE remote endpoint
- [change-server-ip.md](./change-server-ip.md) — when VPS public IP changes
