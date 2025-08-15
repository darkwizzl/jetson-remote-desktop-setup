

# Jetson Headless Setup: Serial Console, Network, SSH, and VNC Desktop (Beginner-Friendly)

This guide walks through bringing a NVIDIA Jetson board online without a monitor: connect over serial, get its IP, SSH in, start a VNC server, and attach the XFCE desktop so the remote view isn’t just a black screen.

Follow each section in order. Commands are in dedicated blocks for easy copy/paste.

***

## 1) Power and Connect

- Power on the Jetson.
- Connect a USB-C cable between the Jetson and the host computer (Linux/macOS/Windows).
- Keep an Ethernet cable ready to connect the Jetson to the host’s network (or router) for IP access.

***

## 2) Find the Jetson’s Serial Port (on the host)

Run this FIRST (before plugging USB-C) then plug USB-C to spot the new serial device.

Linux/macOS:

```bash
ls /dev/tty*
```

Common device names:

- Linux: `/dev/ttyACM0`, `/dev/ttyUSB0`
- macOS: `/dev/tty.usbmodem*` or `/dev/tty.usbserial*`
- Windows (PowerShell): use Device Manager to find “Ports (COM \& LPT)” → note COM number.

Windows tip with WSL: use native Windows serial tools (e.g., PuTTY) pointing to COMx.

***

## 3) Open a Serial Console with screen (host)

Replace the device path with what you found, and use 115200 baud.

Linux/macOS:

```bash
screen /dev/ttyACM0 115200
```

- If screen opens to a blank cursor, press Enter.
- Log in with the Jetson’s username and password when prompted.

To exit screen later: press Ctrl+A then K, then Y to confirm.

If screen is not installed:

```bash
# Debian/Ubuntu (host)
sudo apt update && sudo apt install screen
```


***

## 4) Give the Jetson Network Access

- Connect the Jetson’s Ethernet port to a network that provides Internet and LAN (your router or host’s shared Internet).
- On the Jetson serial console, check IP addresses:

```bash
ifconfig | grep inet
```

Look for an IP on the Ethernet interface (often `eth0`), e.g., `192.168.1.123`. Note this IP.

If ifconfig is missing, use:

```bash
ip -4 addr
```


***

## 5) SSH into the Jetson (from host)

From the host terminal, SSH using the IP found above:

```bash
ssh USERNAME@JETSON_IP
```

Example:

```bash
ssh ubuntu@192.168.1.123
```

Accept the fingerprint, then enter the Jetson password.

***

## 6) Install and Start a VNC Server (Jetson)

Install TigerVNC server:

```bash
sudo apt update
sudo apt install -y tigervnc-standalone-server tigervnc-common
```

Set a VNC password (first time only):

```bash
vncpasswd
```

Start a VNC session on display :1 (maps to TCP port 5901):

```bash
vncserver :1 -localhost no
```

Notes:

- `:1` → port 5901, `:2` → 5902, etc.
- `-localhost no` allows remote connections from other machines on the LAN. For higher security, consider SSH tunneling and keep `-localhost yes`.

Check running VNC servers:

```bash
vncserver -list
```

Kill a specific VNC server (example for :1):

```bash
vncserver -kill :1
```


***

## 7) Connect from the Host to the Jetson’s VNC

Install a VNC viewer on the host (TigerVNC Viewer is fine).

Linux (host):

```bash
sudo apt install -y tigervnc-viewer
```

Open the viewer and connect to:

```
JETSON_IP:5901
```

or shorthand:

```
JETSON_IP:1
```

Enter the VNC password when prompted.

If you see a black screen, that means the VNC is working but no desktop session is attached yet—continue to the next section to install and launch XFCE.

***

## 8) Install XFCE Desktop (Jetson)

XFCE provides a lightweight desktop environment that works well over VNC.

```bash
sudo apt update
sudo apt install -y xfce4 xfce4-goodies
```


***

## 9) Attach XFCE to the VNC Session (Jetson)

Find the active VNC display:

```bash
vncserver -list
```

Example output:

```
TigerVNC server sessions:

X DISPLAY #     PROCESS ID
:1              12345
```

Stop the running session to ensure clean startup:

```bash
vncserver -kill :1
```

Create a startup script so VNC runs XFCE when it starts:

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
startxfce4 &
EOF
chmod +x ~/.vnc/xstartup
```

Start VNC again on :1 with remote access enabled:

```bash
vncserver :1 -localhost no
```

Reconnect with the VNC viewer to `JETSON_IP:1`—you should now see the full XFCE desktop instead of a black screen.

***

## 10) Common Management Commands (Jetson)

- List sessions:

```bash
vncserver -list
```

- Kill a session:

```bash
vncserver -kill :1
```

- Change VNC password:

```bash
vncpasswd
```

- Start on a different display (e.g., :2 → 5902):

```bash
vncserver :2 -localhost no
```


***

## 11) Optional: Make VNC Start Automatically on Boot (Jetson)

Create a systemd user service for convenience.

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/vncserver@:1.service << 'EOF'
[Unit]
Description=Start TigerVNC server at startup
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/usr/bin/vncserver %i -localhost no
ExecStop=/usr/bin/vncserver -kill %i
PIDFile=/home/%u/.vnc/%H%i.pid

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable vncserver@:1.service
systemctl --user start vncserver@:1.service
```

If systemd user services aren’t active, you may need:

```bash
loginctl enable-linger $USER
```


***

## 12) Quick Troubleshooting

- No serial device found:

```bash
# Try different cable/port, and check permissions on Linux
sudo dmesg | tail -n 50
```

- SSH blocked:
    - Ensure Ethernet is connected and Jetson has an IP.
    - Ping test from host:

```bash
ping -c 3 JETSON_IP
```

- VNC black screen persists:
    - Make sure `~/.vnc/xstartup` is executable and contains `startxfce4 &`.
    - Kill and restart the VNC server:

```bash
vncserver -kill :1
vncserver :1 -localhost no
```

- Authentication failures:

```bash
vncpasswd
```


***

You’re all set: serial console for first-time access, SSH for remote shell, and TigerVNC with XFCE for a full remote desktop.

