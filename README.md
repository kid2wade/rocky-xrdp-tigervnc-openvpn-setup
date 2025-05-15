# rocky-xrdp-tigervnc-openvpn-setup
Step-by-step guide to install and configure XRDP, TigerVNC &amp; OpenVPN on Rocky Linux for secure remote desktop and VPN access.

---

## Table of Contents

- [Overview](#overview)  
- [Prerequisites](#prerequisites)  
- [Step A: XRDP & TigerVNC Setup](#step-a-xrdp--tigervnc-setup)  
- [Stea B: Harden SSH](#step-b-harden-ssh)  
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

## Step B: Harden SSH

To lock down SSH, we’ll switch to key-based login only, disable root logins, and (optionally) move SSH off port 22.

### B1. Generate an SSH key pair (if you don’t have one)

On your **local** machine:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# Accept defaults and enter a passphrase when prompted
```

### B2. Copy your public key to the server

```bash
ssh-copy-id -p 22 $USER@<SERVER_IP_OR_HOSTNAME>
```
This installs your key in /home/$USER/.ssh/authorized_keys.

### B3. Edit the SSH daemon config

```bash
sudo vi /etc/ssh/sshd_config
```

Make these changes (uncomment or add):

```diff
-Port 22
+Port 2222                   # optional: change SSH port to 2222

 PermitRootLogin no          # disable root SSH logins
 PasswordAuthentication no   # turn off password-based logins
 PubkeyAuthentication yes    # enable key-based auth only
```

Tip: if you changed the port, remember to open it in your firewall:

```bash
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

### B4. Restart SSHD

```bash
sudo systemctl restart sshd
```

### B5. Test your new SSH settings

From your local machine:

```bash
ssh -p 2222 $USER@<SERVER_IP_OR_HOSTNAME>
```

You should be prompted for your key’s passphrase, not a system password. If it works, SSH is now hardened.

---

## Step C: OpenVPN Server

We’ll install OpenVPN and Easy-RSA, build our PKI, configure the server, open the firewall, and expose the VPN via DuckDNS + router port-forwarding.

### C1. Install OpenVPN & Easy-RSA

```bash
sudo dnf install -y epel-release
sudo dnf install -y openvpn easy-rsa
```

### C2. Initialize the PKI directory

```bash
sudo mkdir -p /etc/openvpn/easy-rsa
sudo cp -r /usr/share/easy-rsa/3/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa
sudo ./easyrsa init-pki
```

### C3. Build CA, server certificate & DH parameters

```bash
sudo ./easyrsa build-ca nopass
sudo ./easyrsa gen-req server nopass
sudo ./easyrsa sign-req server server
sudo ./easyrsa gen-dh
```

Files created:

- /etc/openvpn/easy-rsa/pki/ca.crt

- /etc/openvpn/easy-rsa/pki/issued/server.crt

- /etc/openvpn/easy-rsa/pki/private/server.key

- /etc/openvpn/easy-rsa/pki/dh.pem


### C4. Configure the OpenVPN server

```bash
sudo mkdir -p /etc/openvpn/server
sudo cp /usr/share/doc/openvpn/sample/sample-config-files/server.conf /etc/openvpn/server/server.conf
```

Edit /etc/openvpn/server/server.conf:

```ini
port 1194
proto udp
dev tun

ca   /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key  /etc/openvpn/easy-rsa/pki/private/server.key
dh   /etc/openvpn/easy-rsa/pki/dh.pem

server 10.8.0.0 255.255.255.0

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 8.8.8.8"

keepalive 10 120
cipher AES-256-CBC
user nobody
group nobody
persist-key
persist-tun

status openvpn-status.log
log    openvpn.log
verb 3
explicit-exit-notify 1
```

### C5. Enable IP forwarding & configure firewall

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-openvpn.conf
sudo sysctl --system

sudo firewall-cmd --permanent --add-service=openvpn
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```

If SELinux is enforcing, relabel:

```bash
sudo restorecon -Rv /etc/openvpn
```

### C6. Start the OpenVPN service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openvpn-server@server.service
```

Verify:

```bash
sudo systemctl status openvpn-server@server.service
ss -u -lnp | grep 1194
```

### C7. Expose VPN via DuckDNS & Router Port-Forwarding

1. Register a free DuckDNS hostname

Sign in at https://www.duckdns.org/, create a domain (example: myvpn.duckdns.org).

2. Install the DuckDNS updater on Rocky:

```bash
sudo dnf install -y curl
sudo mkdir -p /etc/duckdns
cat << 'EOF' | sudo tee /etc/duckdns/duck.sh
#!/usr/bin/env bash
echo "https://www.duckdns.org/update?domains=YOURDOMAIN&token=YOURTOKEN&ip=" \
  | curl -k -o /etc/duckdns/duck.log -K -
EOF
sudo chmod +x /etc/duckdns/duck.sh
echo "*/5 * * * * root /etc/duckdns/duck.sh >/dev/null 2>&1" | sudo tee /etc/cron.d/duckdns
```
Replace YOURDOMAIN and YOURTOKEN with your DuckDNS values.

3. Configure your router to forward UDP port 1194 → <SERVER_LAN_IP>:1194.

4. Test external reachability (from a different network):

```bash
# On Windows PowerShell:
Test-NetConnection -ComputerName myvpn.duckdns.org -Port 1194 -InformationLevel Detailed
``` 

Clients will now use:

```bash
remote myvpn.duckdns.org 1194
```
---

## Step D: Generate Client Profile

We’ll create and sign a client certificate, gather all necessary files, build a base configuration, then bundle everything into a single .ovpn file.

### D1. Generate & sign the client certificate

```bash
cd /etc/openvpn/easy-rsa
sudo ./easyrsa gen-req client1 nopass
sudo ./easyrsa sign-req client client1
```

Outputs:

- /etc/openvpn/easy-rsa/pki/issued/client1.crt

- /etc/openvpn/easy-rsa/pki/private/client1.key

### D2. Collect CA, client cert/key & HMAC key

```bash
mkdir -p ~/client-configs/files
cp /etc/openvpn/easy-rsa/pki/ca.crt        ~/client-configs/files/
cp /etc/openvpn/easy-rsa/pki/issued/client1.crt  ~/client-configs/files/
cp /etc/openvpn/easy-rsa/pki/private/client1.key ~/client-configs/files/
sudo cp /etc/openvpn/ta.key                     ~/client-configs/files/
```

If ta.key is missing, generate it:

```bash
sudo openvpn --genkey --secret /etc/openvpn/ta.key
```

### D3. Create a base client config

```bash
cp /usr/share/doc/openvpn/sample/sample-config-files/client.conf ~/client-configs/base.conf
```

Edit ~/client-configs/base.conf:

```ini
client
dev tun
proto udp
remote myvpn.duckdns.org 1194
resolv-retry infinite
nobind
persist-key
persist-tun

ca   ca.crt
cert client1.crt
key  client1.key
tls-auth ta.key 1

cipher AES-256-CBC
verb 3
keepalive 10 120
```

### D4. Bundle everything into one .ovpn

```bash
cat ~/client-configs/base.conf \
    <(printf '\n<ca>\n') ~/client-configs/files/ca.crt \
    <(printf '\n</ca>\n<cert>\n') ~/client-configs/files/client1.crt \
    <(printf '\n</cert>\n<key>\n') ~/client-configs/files/client1.key \
    <(printf '\n</key>\n<tls-auth>\n') ~/client-configs/files/ta.key \
    <(printf '\n</tls-auth>\n') \
  > ~/client-configs/files/client1.ovpn
```

That produces a single client1.ovpn containing all certificates and keys.

### D5. Test the client connection

On the client machine:

1. Install the OpenVPN client.

2. Copy down client1.ovpn.

3. Run:

```bash
sudo openvpn --config client1.ovpn
```

4. Look for “Initialization Sequence Completed.”

5. Verify connectivity:

```bash
ping 10.8.0.1
```

---

## Step E: Testing & Troubleshooting

Once Steps A–D are complete, let’s verify end-to-end functionality and cover quick fixes.

### E1. Verify SSH

```bash
# From your local machine:
ssh -p 2222 $USER@<SERVER_IP_OR_HOSTNAME>
```

- You should get a shell without a password prompt.

- If you’re locked out, use your console or cloud-provider serial access to revert /etc/ssh/sshd_config.

### E2. Test XRDP

1. On Windows, launch Remote Desktop Connection (mstsc).

2. Connect to:

```makefile
<SERVER_IP_OR_HOSTNAME>:3389
```

3. Log in with your Linux user.

4. You should land in your Xfce (or GNOME) desktop.

If it fails:

- sudo systemctl status xrdp xrdp-sesman

- Check /var/log/xrdp.log and /var/log/xrdp-sesman.log.

- Ensure port 3389 is open in firewall-cmd --list-ports or --list-services.

### E3. Test TigerVNC

1. On Windows, open your VNC viewer.

2. Connect to:

```makefile
<SERVER_IP_OR_HOSTNAME>:<5900+DISPLAY>
```

(example: 10.8.0.1:5903 if DISPLAY=3)

3. Enter the VNC password.

If you see “connection refused”:

- Confirm the VNC service is active:

```bash
sudo systemctl status vncserver@:<DISPLAY>.service
```

- Check that /etc/tigervnc/vncserver.users has :<DISPLAY>=$USER.

- If using the per-user unit, ensure you’ve logged in (or switched After= to default.target).

- Verify port with ss -tlnp | grep 5903.

### E4. Test OpenVPN

1. On your client, import and connect with client1.ovpn.

2. Wait for “Initialization Sequence Completed.”

3. Ping the server’s VPN IP:

```bash
ping 10.8.0.1
```

4. Once online, re-test RDP/VNC over the VPN IP (10.8.0.1:3389 or :5903).

If it fails:

- On the server, inspect:

```bash
sudo journalctl -u openvpn-server@server -n 20
```

- Ensure openvpn-server@server.service is running.

- Check firewall rules:

```bash
firewall-cmd --list-all
```

- Verify DuckDNS is updating:

```bash
cat /etc/duckdns/duck.log
```

- From an external network, test port-forward:

```bash
Test-NetConnection -ComputerName myvpn.duckdns.org -Port 1194
```
---

## Common Pitfalls & Solutions

| Issue                   | Symptoms                       | Fix                                                                                                                                                                             |
|-------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **XRDP not listening**  | RDP client times out           | <pre>sudo systemctl restart xrdp<br>ss -tlnp &#124; grep 3389<br>sudo firewall-cmd --add-service=rdp --reload</pre>                                                              |
| **VNC “No user…”**      | Service fails with “No user…”  | <pre>echo ":N=$USER" &#124; sudo tee /etc/tigervnc/vncserver.users<br>vncpasswd</pre>                                                                                             |
| **VNC starts only…**    | Per-user unit doesn’t start    | <pre>sudo systemctl enable --now vncserver@:&lt;N&gt;.service</pre><br>Or change `After=` to `default.target` in your user unit.                                                 |
| **OpenVPN config error**| Error opening configuration    | Ensure `/etc/openvpn/server/` exists and **all** paths in `server.conf` are absolute (e.g. `ca /etc/openvpn/.../ca.crt`).                                                       |
| **OpenVPN won’t bind**  | Service exit code 1            | <pre>ss -u -lnp &#124; grep 1194<br>journalctl -u openvpn-server@server -n 20<br>sudo firewall-cmd --add-service=openvpn --reload</pre>                                            |
| **DuckDNS not updating**| DNS still shows old IP         | <pre>cat /etc/duckdns/duck.log<br>/etc/duckdns/duck.sh</pre><br>Check your `/etc/cron.d/duckdns` syntax.                                                                          |
| **Locked out of SSH**   | Cannot connect over SSH        | Use console access to revert changes in `/etc/ssh/sshd_config` (e.g. `PermitRootLogin no`, custom port).<br>Then `sudo systemctl restart sshd`.                                 |

---
