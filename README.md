# rocky-xrdp-tigervnc-openvpn-setup
Step-by-step guide to install and configure XRDP, TigerVNC &amp; OpenVPN on Rocky Linux for secure remote desktop and VPN access.

---

## Table of Contents

- [Overview](#overview)  
- [Prerequisites](#prerequisites)  
- [Step A: XRDP & TigerVNC Setup](#step-a-xrdp--tigervnc-setup)  
- [Step B: Harden SSH](#step-b-harden-ssh)  
- [Step C: OpenVPN Server](#step-c-openvpn-server)  
- [Step D: Generate Client Profile](#step-d-generate-client-profile)  
- [Step E: Testing & Troubleshooting](#step-e-testing--troubleshooting)  
- [Common Pitfalls & Solutions](#common-pitfalls--solutions)  

---

## Overview

This repository documents how to turn a Rocky Linux machine into a secure remote-access server, combining:

- **XRDP** (RDP-compatible remote desktop)  
- **TigerVNC** (native VNC service on a spare display)  
- **OpenVPN** (site-to-site or road-warrior VPN)  

By the end, you’ll be able to:

- RDP or VNC into your server over SSH or VPN  
- Lock down SSH to key-only on a non-standard port  
- Tunnel remote-desktop traffic securely  

---

## Prerequisites

- Rocky Linux **8** or **9**  
- Non-root user with **sudo** privileges  
- Basic familiarity with **systemd**, **firewalld**, and **SSH key** authentication

---

## Step A: XRDP & TigerVNC Setup

We’ll first experiment with the default, system-wide TigerVNC on display `:1`, see why it fails, then switch to a per-user service on `:2`. Finally we’ll configure XRDP to use Xorg.

### A1. Install required packages

```bash
sudo dnf update
sudo dnf install -y tigervnc-server xrdp xorgxrdp
```

Enable & start XRDP:

```bash
sudo systemctl enable --now xrdp xrdp-sesman
ss -tlnp | grep 3389   # verify xrdp is listening on 3389
```

### A2. Free up an X display

By default, GDM (the GNOME Display Manager) holds display :0 (port 5900) and our local GUI session usually takes :1 (port 5901). We can either stop that service or simply pick a higher display number for VNC.

```bash
# To disable GDM (so :0 and :1 are free for VNC if desired)
sudo systemctl disable --now gdm.service
sudo systemctl mask   gdm.service

# (Later, to restore GDM)
sudo systemctl unmask   gdm.service
sudo systemctl enable --now gdm.service
```

### A3. Install Xfce & create our VNC startup

1. Install Xfce (or another desktop environment):

```bash
sudo dnf groupinstall -y "Xfce"
```

2. As our non-root user ($USER), set up ~/.vnc/xstartup and password:

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/sh
[ -r "$HOME/.Xresources" ] && xrdb "$HOME/.Xresources"
exec startxfce4 &
EOF
chmod 700 ~/.vnc/xstartup
vncpasswd   # choose a VNC password
```

### A4. Map & start TigerVNC on display :N

1. Pick an unused display number:

```bash
DISPLAY=3   # or 2, 4, etc.
```

2. System-wide service (always-on)

```bash
echo ":${DISPLAY}=$USER" | sudo tee /etc/tigervnc/vncserver.users

sudo systemctl daemon-reload
sudo systemctl enable --now vncserver@:${DISPLAY}.service

ss -tlnp | grep $((5900+DISPLAY))
# we should see Xvnc listening on port 5900+N (example: port 5903)
```

3. Per-user service (starts on user login)

```bash
sudo loginctl enable-linger $USER

mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/vncserver@.service << 'EOF'
[Unit]
Description=TigerVNC Server for display %i
After=graphical-session.target

[Service]
Type=forking
WorkingDirectory=%h
ExecStart=/usr/bin/vncserver %i -geometry 1280x800 -localhost no
ExecStop=/usr/bin/vncserver -kill %i

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now vncserver@:${DISPLAY}.service

ss -tlnp | grep $((5900+DISPLAY))
```

Note:
If we use the per-user TigerVNC service (~/.config/systemd/user/vncserver@.service), it waits until our graphical-session target is active, so with GDM running, VNC won’t start until we log out of GNOME. To avoid that, either use the system-wide vncserver@.service (runs at boot) or adjust After= in the user unit to default.target.


### A5. Configure XRDP session types

Edit /etc/xrdp/xrdp.ini and enable the backends we want:

```ini
# --- Xorg session (requires xorgxrdp) ---
[Xorg]
name=Xorg
lib=libxup.so
username=ask
password=ask
ip=127.0.0.1
port=-1

# --- TigerVNC session (requires tigervnc-server) ---
[Xvnc]
name=Xvnc
lib=libvnc.so
username=ask
password=ask
port=-1
```
- Only Xorg: comment out the [Xvnc] block

- Only VNC: comment out the [Xorg] block

- Both: leave both sections active

Restart XRDP:

```bash
sudo systemctl restart xrdp
```

### A6. Test both VNC & RDP

- VNC:
In our VNC client, connect to

```makefile
<SERVER_IP_OR_HOSTNAME>:<5900+DISPLAY>
```
example: 192.168.0.230:5903 if DISPLAY=3, and enter the VNC password.

- RDP:
Open Windows Remote Desktop Connection (mstsc) to

```makefile
<SERVER_IP_OR_HOSTNAME>:3389
```
then log in with our Linux username/password.

---
