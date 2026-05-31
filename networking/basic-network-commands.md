
# Basic Network Commands

---

Display network cards:

```bash
ip addr
```

Display routes:

```bash
ip route
```

Display DNS servers:

```bash
cat /etc/resolv.conf
```

Ping a device:

```bash
ping -c 4 192.168.1.1
```

Analyze packets' path:

```bash
sudo apt update
sudo apt install traceroute
traceroute google.com
```

Scan ports:

```bash
sudo apt install nmap
nmap localhost
nmap 192.168.1.10
```

Check active connections:

```bash
ss -tulpn
netstat -tulpn
```

Capture network traffic:

```bash
sudo apt install wireshark
sudo wireshark
```

Real-time monitoring:

```bash
sudo apt install iftop
sudo iftop
```