# When Server Public IP Changes

Runbook when the VPS public IP changes (e.g. `87.107.94.235` → `85.133.162.63`).

Provider usually updates only the server IP. The GRE tunnel provider must also update their side to accept the new local IP.

---

## Variables

| Variable | Old | New |
|----------|-----|-----|
| `OLD_SERVER_IP` | `87.107.94.235` | — |
| `NEW_SERVER_IP` | — | `85.133.162.63` |
| `TUNNEL_IP` | unchanged | e.g. `185.237.84.75` |
| `PRIVATE_IP` | unchanged | e.g. `172.17.50.114/30` |
| `TUNNEL_PEER` | unchanged | e.g. `172.17.50.113` |

> `TUNNEL_IP` and `PRIVATE_IP` usually stay the same. Confirm with provider.

---

## Checklist

```
□ Provider updated GRE tunnel with new SERVER_IP
□ GRE tunnel recreated with new local IP
□ Reboot script updated
□ WGDashboard remote_endpoint updated
□ Client configs re-exported (Endpoint IP changed)
□ UDP 51281/51282 open on new IP (cloud firewall)
```

---

## 1. Recreate GRE Tunnel

Only `local` changes. `remote` stays the same unless provider says otherwise.

```bash
sudo ip link set gre1 down && sudo ip link delete gre1
```

```bash
sudo ip tunnel add gre1 mode gre remote <TUNNEL_IP> local <NEW_SERVER_IP> ttl 255
```

```bash
sudo ip addr add <PRIVATE_IP> dev gre1
```

```bash
sudo ip link set gre1 up
```

---

## 2. Test Tunnel

```bash
ip tunnel show gre1
ping -c 3 <TUNNEL_PEER>
```

If peer ping fails → call provider. They must allow GRE from `<NEW_SERVER_IP>`.

Test internet via tunnel:

```bash
sudo ip route add 1.1.1.1 dev gre1
ping -c 3 -I gre1 1.1.1.1
curl -s --interface gre1 --connect-timeout 5 -o /dev/null -w "HTTP: %{http_code}\n" https://1.1.1.1
sudo ip route del 1.1.1.1 dev gre1
```

Test WG client routing (no client needed):

```bash
ip route get 8.8.8.8 from 10.8.0.2 iif wg2
ip route get 8.8.8.8 from 10.9.0.2 iif wg3
```

Expected: `dev gre1 table 202` / `table 203`.

---

## 3. Update Reboot Script

```bash
sudo sed -i 's/local <OLD_SERVER_IP>/local <NEW_SERVER_IP>/' /usr/local/bin/setup-gre-tunnel.sh
grep "ip tunnel add" /usr/local/bin/setup-gre-tunnel.sh
```

Example:

```bash
sudo sed -i 's/local 87.107.94.235/local 85.133.162.63/' /usr/local/bin/setup-gre-tunnel.sh
```

---

## 4. Update WGDashboard (client Endpoint IP)

Client configs use `Endpoint = <SERVER_IP>:port`. After server IP change, re-export all configs.

### ini file

```bash
sudo sed -i 's/^remote_endpoint = .*/remote_endpoint = <NEW_SERVER_IP>/' /srv/wg-dashboard/data/wg-dashboard.ini
sudo grep remote_endpoint /srv/wg-dashboard/data/wg-dashboard.ini
```

### docker-compose (optional)

```yaml
environment:
  - public_ip=<NEW_SERVER_IP>
  - dynamic_config=false
```

```bash
cd /srv/wg-dashboard && sudo docker compose up -d
```

Or set in dashboard UI: **Settings → Peers Settings → Remote Endpoint**.

---

## 5. WireGuard — Server Side

**No changes needed** to `wg2.conf` / `wg3.conf`, keys, routing tables, or iptables.

Restart only if tunnel was recreated while WG was up:

```bash
sudo wg-quick down /srv/wg-dashboard/config/wg2.conf
sudo wg-quick down /srv/wg-dashboard/config/wg3.conf
sudo wg-quick up /srv/wg-dashboard/config/wg2.conf
sudo wg-quick up /srv/wg-dashboard/config/wg3.conf
```

Or run the reboot script:

```bash
sudo /usr/local/bin/setup-gre-tunnel.sh
```

---

## 6. Client Configs

Only this line changes in client `.conf` files:

```ini
# Old
Endpoint = <OLD_SERVER_IP>:51281

# New
Endpoint = <NEW_SERVER_IP>:51281
```

| What changes | What stays |
|--------------|------------|
| `Endpoint` IP | PrivateKey, PublicKey |
| | Address (`10.8.0.x`) |
| | AllowedIPs, DNS, MTU |

Re-export from WGDashboard or tell users to edit Endpoint manually.

Ports:

| Interface | Port |
|-----------|------|
| wg2 | `51281` |
| wg3 | `51282` |

---

## 7. Cloud Firewall

Ensure on the new server IP:

| Rule | Port |
|------|------|
| UDP | `51281` (wg2) |
| UDP | `51282` (wg3) |

---

## 8. Verify After Reboot

No manual action needed if script and ini are updated.

```bash
ip tunnel show gre1
ip -br link | grep -E 'gre1|wg2|wg3'
sudo wg show
ping -c 2 <TUNNEL_PEER>
```

Connect one client and check on server:

```bash
sudo wg show wg2 | grep -A3 "latest handshake"
```

Expected: `latest handshake: X seconds ago`.

---

## Common Mistakes

| Mistake | Reality |
|---------|---------|
| `ping -I gre1 1.1.1.1` fails | Normal — default route uses `ppp0`, not `gre1` |
| Regenerate all WG keys | Not needed — only Endpoint IP changes |
| Change `TUNNEL_IP` when server IP changes | Usually not needed — only `local` changes |
| Edit server `wg2.conf` Endpoint | Server config has no Endpoint — that's client-side |

---

## Related Docs

- [setup.md](./setup.md) — initial setup
- [change-tunnel-ip.md](./change-tunnel-ip.md) — switch GRE remote endpoint
