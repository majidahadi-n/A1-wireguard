# when change thunnel IP 

```bash
sudo ip link set gre1 down && sudo ip link delete gre1
```

```bash
sudo ip tunnel add gre1 mode gre remote  **<NEW_TUNNEL_IP>** local **<SERVER_IP>** ttl 255
```


```bash
sudo ip addr add **<IP_PRIVATE>** dev gre1
```


```bash
sudo ip link set gre1 up
```


## test

```bash
ping -c 3 172.17.50.113
```

## for reboot script

```bash
sudo sed -i 's/remote **<OLD_TUNNEL_IP>**/remote **<NEW_TUNNEL_IP>**/' /usr/local/bin/setup-gre-tunnel.sh
```
