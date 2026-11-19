# Nvidia

```bash
ubuntu-drivers devices
sudo apt update
sudo ubuntu-drivers autoinstall
sudo ubuntu-drivers install nvidia:595-open
```

```bash
nvidia-smi
watch -n 1 nvidia-smi
nvidia-smi -q
nvidia-smi -L
lspci | grep -i nvidia
```
