# Rollback: US GRE + wg4 (keep gre1, wg2, wg3)

Run **Iran block first**, then **US block**.

---

## Iran server

```bash
# Stop wg4 (PostDown cleans wg4/gre2 iptables + table 204 rules)
sudo wg-quick down /srv/wg-dashboard/config/wg4.conf 2>/dev/null || true

# Remove US tunnel
sudo ip link set gre2 down 2>/dev/null || true
sudo ip link delete gre2 2>/dev/null || true

# Extra cleanup if PostDown missed something
sudo ip rule del from 10.10.0.0/24 lookup 204 priority 204 2>/dev/null || true
sudo ip rule del to 10.10.0.0/24 lookup main priority 10 2>/dev/null || true
sudo ip route flush table 204 2>/dev/null || true

sudo iptables -D FORWARD -i wg4 -o gre2 -j ACCEPT 2>/dev/null || true
sudo iptables -D FORWARD -i gre2 -o wg4 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || true
sudo iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o gre2 -j MASQUERADE 2>/dev/null || true
sudo iptables -t mangle -D FORWARD -i wg4 -o gre2 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu 2>/dev/null || true

# Remove wg4 interface if still present
sudo ip link delete wg4 2>/dev/null || true

# Restore reboot script (gre1 + wg2/wg3 only) — see setup-gre-tunnel.sh in repo
sudo tee /usr/local/bin/setup-gre-tunnel.sh > /dev/null << 'EOF'
#!/bin/bash
IRAN_IP="85.133.162.63"
PROVIDER_REMOTE="185.237.84.75"
GRE1_ADDR="172.17.50.114/30"
WG_DIR="/srv/wg-dashboard/config"

ip link delete gre1 2>/dev/null || true
ip tunnel add gre1 mode gre remote "${PROVIDER_REMOTE}" local "${IRAN_IP}" ttl 255
ip addr add "${GRE1_ADDR}" dev gre1
ip link set gre1 up

while iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o gre1 -j MASQUERADE 2>/dev/null; do true; done
while iptables -t nat -D POSTROUTING -s 10.9.0.0/24 -o gre1 -j MASQUERADE 2>/dev/null; do true; done
while iptables -t nat -D POSTROUTING -o gre1 -j MASQUERADE 2>/dev/null; do true; done

wg-quick down "${WG_DIR}/wg2.conf" 2>/dev/null || true
wg-quick down "${WG_DIR}/wg3.conf" 2>/dev/null || true
wg-quick up "${WG_DIR}/wg2.conf"
wg-quick up "${WG_DIR}/wg3.conf"
exit 0
EOF
sudo chmod +x /usr/local/bin/setup-gre-tunnel.sh

# Remove US-only cron/script if added
sudo rm -f /usr/local/bin/setup-gre-tunnel-us.sh

# Dashboard: wg4 out of autostart (optional)
sudo sed -i 's/^autostart = .*/autostart = wg2||wg3/' /srv/wg-dashboard/data/wg-dashboard.ini
```

### Optional: delete wg4 files (does not touch wg2/wg3)

```bash
sudo rm -f /srv/wg-dashboard/config/wg4.conf \
  /srv/wg-dashboard/config/wg4-*.key \
  /srv/wg-dashboard/config/wg4-*.pub
```

### Verify Iran

```bash
ip tunnel show gre2 2>&1
ip link show wg4 2>&1
sudo wg show wg2
sudo wg show wg3
ip tunnel show gre1
ip route get 8.8.8.8 from 10.8.0.4 iif wg2
sudo crontab -l
```

---

## US server (107.172.73.109)

```bash
sudo ip link set gre-us down 2>/dev/null || true
sudo ip link delete gre-us 2>/dev/null || true

sudo iptables -D FORWARD -i gre-us -o eth0 -j ACCEPT 2>/dev/null || true
sudo iptables -D FORWARD -i eth0 -o gre-us -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || true
sudo iptables -t nat -D POSTROUTING -s 10.100.100.0/30 -o eth0 -j MASQUERADE 2>/dev/null || true
sudo iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o eth0 -j MASQUERADE 2>/dev/null || true

sudo rm -f /usr/local/bin/setup-gre-us.sh
sudo crontab -l   # remove line with setup-gre-us.sh if present
```

### Verify US

```bash
ip link show gre-us 2>&1
sudo iptables -S FORWARD | grep gre-us
sudo iptables -t nat -S POSTROUTING | grep -E '10.10|10.100'
```

---

## Crontab note

Iran should keep only:

```
@reboot sleep 60 && /usr/local/bin/setup-gre-tunnel.sh
```

Remove any line with `setup-gre-tunnel-us.sh` or `setup-gre-us.sh`.
