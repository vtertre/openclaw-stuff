# OpenClaw setup on a fresh Ubuntu 24.04

## 1. Init

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

---

## 2. If WiFi is broken

Broadcom driver available in the current `noble-updates` (`...23ubuntu1.1`) is currently broken for 6.17 kernel.

### 2.1 Cleanup and enable `noble-proposed`

```bash
sudo apt purge -y bcmwl-kernel-source broadcom-sta-dkms broadcom-sta-common broadcom-sta-source
sudo apt autoremove -y
sudo add-apt-repository "deb http://archive.ubuntu.com/ubuntu noble-proposed restricted"
sudo apt update
```

### 2.2 Check newest version

Check for a newer verison than `...23ubuntu1.1` coming from `noble-proposed`. `...23ubuntu1.2` should be available at the time of writing.

```bash
apt policy broadcom-sta-dkms bcmwl-kernel-source
```

### 2.3 Install from `noble-proposed`

```bash
sudo apt -t noble-proposed install bcmwl-kernel-source
```

### 2.4 Reboot

```bash
sudo reboot
```

### 2.5 Verify

WiFi interface should now be visible.

```bash
nmcli device
ip link
```

### 2.6 ⚠️ Disable `noble-proposed`

```bash
sudo add-apt-repository --remove "deb http://archive.ubuntu.com/ubuntu noble-proposed restricted"
sudo apt update
```

---

## 3. Install Node 24

```bash
sudo apt install curl
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
```

---

## 4. Onboard OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```
