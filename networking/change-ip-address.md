
# Change ip address

---

## Temporary

---

Display network cards:

```bash
ip addr
```

Assign a new ip address:

```bash
sudo ip addr flush dev eno1
sudo ip addr add 192.168.1.100/24 dev eno1
sudo ip link set eno1 up
```

Add the default gateway:

```bash
sudo ip route add default via 192.168.1.1
```

Verify:

```bash
ip addr show eno1
ip route
```

---

## Permanently

---

Display Netplan's files:

```bash
ls /etc/netplan/
```

Edit a Netplan file:

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

```bash
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
            - 8.8.8.8
            - 1.1.1.1
```

Apply changes:

```bash
sudo netplan apply
```

Verify:

```bash
ip addr
```