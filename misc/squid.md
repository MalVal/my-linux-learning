# Squid

```bash
sudo yum update
sudo yum install squid
systemctl status squid
```

Choose the allowed IP's and add the port number

```bash
cd /etc
sudo nano squid.conf
acl myacl src 212.166.44.73/32
http_access allow myacl
http_port 443
```

```bash
systemctl start squid
sudo ss -aptn
```
