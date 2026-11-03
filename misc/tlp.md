# TLP

```bash
sudo apt install tlp
sudo tlp start
sudo systemctl enable tlp
sudo tlp-stat
```

```bash
sudo nano /etc/tlp.conf
```

```txt
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80

START_CHARGE_THRESH_BAT1=75
STOP_CHARGE_THRESH_BAT1=80

RUNTIME_PM_ON_BAT=auto
```