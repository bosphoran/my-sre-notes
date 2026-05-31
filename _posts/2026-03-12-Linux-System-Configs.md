---
layout: post
title: "Linux System Configuration"
date: 2026-03-12
categories: [linux]
---

## Linux PC Configs

### Screen Brightness

```bash
ls /sys/class/backlight/ # gives the backlight interface
zafar@zserver:~$ amdgpu_bl1

cat /sys/class/backlight/amdgpu_bl1/max_brightness # max brightness value
zafar@zserver:~$ 255
cat /sys/class/backlight/amdgpu_bl1/brightness # current brightness value
zafar@zserver:~$ 90
```

```bash
echo 200 | sudo tee /sys/class/backlight/amdgpu_bl1/brightness # sets brightness to 200/255
```

### Helper Tools for Brightness Control:

```bash
sudo apt install brightnessctl -y
sudo brightnessctl set +20% # increase brightness by %20
sudo brightnessctl set 30%- # decrease brightness by %30
sudo brightnessctl set 70% # set brightness to %70
```

### CLI font size

```bash
sudo dpkg-reconfigure console-setup # opens the setup menu
```

- Encoding → leave as default (UTF-8).
- Character set → choose "Latin".
- Font → pick something like "Terminus".
- Font size → choose a larger one (e.g., 16x32).

After you finish, the console font will be bigger immediately.

---

### SSID/WiFi Connection

```bash
iwgetid -r # which SSID I am connected to
ip a show # ipconfig version of Linux/ show IP address
hostname -I # shows the local IP address
networkctl status wlo1 # ipconfig /all
nmcli device wifi list # list availabel wifi
sudo nmcli dev wifi connect "NEW_SSID" password "NEW_PASSWORD" # immediate SSID change
```

### 🔧 Disable swap permanently

- Edit /etc/fstab and remove or comment out the swap entry:

```bash
sudo nano /etc/fstab
```

- Look for a line like:

```bash
/swapfile   none    swap    sw    0   0
```

- Add a # at the beginning to comment it out:

```bash
# /swapfile   none    swap    sw    0   0
```

- Save and exit.

> ✅ Takeaway: Disabling swap is mandatory for kubelet to run. Do it once with swapoff -a, then make it permanent by editing /etc/fstab.
