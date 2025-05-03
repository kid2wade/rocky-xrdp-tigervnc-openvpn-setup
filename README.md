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
```

### A2. System-wide TigerVNC on display :1 (troubleshoot & disable)

**Note:**  
   - Display `:0` is already in use by your local GUI (GDM).  
   - When I ran `vncserver@:1.service`, it consistently failed with:  

     ```
     Fatal server error:
     (EE) Cannot establish any listening sockets…
     ```
   - Rather than fight stale locks on `:1`, I moved to a free display, `:2`.

1. Create your VNC password:

```bash
vncpasswd
```

2. Create your VNC startup script (so VNC knows which desktop to launch):

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/sh
exec startxfce4 &
EOF
chmod +x ~/.vnc/xstartup
```

3. Map your user to :2 in /etc/tigervnc/vncserver.users:

```bash
echo ":2=$USER" | sudo tee /etc/tigervnc/vncserver.users
```

   - If you really want to experiment on :1, you can still do:

   ```bash
   echo ":1=$USER" | sudo tee /etc/tigervnc/vncserver.users  
   ```

4. Clean up any leftover locks on both displays:

```bash
sudo rm -f /tmp/.X1-lock /tmp/.X11-unix/X1 \
             /tmp/.X2-lock /tmp/.X11-unix/X2
```

5. Disable the broken :1 service and enable the working :2 service:

```bash
sudo systemctl disable --now vncserver@:1.service  
sudo systemctl enable --now vncserver@:2.service
```

6. Verify TigerVNC is listening on port 5902:

```bash
ss -tlnp | grep 5902
```

### A3. Per-user TigerVNC on spare display :2

1. Enable “linger” so your user-units can run after logout:

```bash
sudo loginctl enable-linger $USER
```

2. Create the user-unit directory:

```bash
mkdir -p ~/.config/systemd/user
```

3. Write the per-user service at ~/.config/systemd/user/vncserver@.service:

```ini
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
```

4. Reload, enable & start it under your user:

```bash
systemctl --user daemon-reload
systemctl --user enable --now vncserver@:2.service
```

5. Confirm VNC is listening on port 5902:

```bash
ss -tlnp | grep 5902
```

### A4. Configure XRDP session types

1. XRDP can offer multiple backends—here Xorg **and** TigerVNC—by (un)commentin sections in `/etc/xrdp/xrdp.ini`. Edit that file so that:

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

- Only Xorg: comment out the entire [Xvnc] block.
- Only VNC: comment out the entire [Xorg] block.
- Both: leave both sections active.

💡 Tip: Make sure you have xorgxrdp installed for [Xorg] and tigervnc-server for [Xvnc] before enabling either.

2. Finally, restart and enable XRDP:

```bash
sudo systemctl enable --now xrdp xrdp-sesman
```

### A5. Test both VNC & RDP

VNC: connect to your-server:5902 with your VNC password.

RDP: point your RDP client at your-server:3389, log in with your Linux user.

