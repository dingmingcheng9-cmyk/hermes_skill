---
name: linux-remote-desktop
category: devops
description: >-
  Set up remote desktop on a headless Linux server so it can be operated like
  Windows via RDP (mstsc). Covers two approaches: (A) lightweight XFCE4 + xrdp
  for servers with no desktop, and (B) native GNOME via vkms + gnome-remote-desktop
  for Ubuntu Desktop installations.
trigger: >-
  User wants to make their Linux server operable with a GUI/desktop via remote
  connection, or asks to 'install remote desktop', 'RDP', 'XFCE', 'xrdp',
  'GNOME remote', or 'make it like Windows'.
---

# Linux Remote Desktop

Turn a headless Linux server into a machine you can remote into with the
Windows Remote Desktop client (mstsc).

## First: detect what's already installed

**Critical first step** — don't blindly install XFCE if the system already
has a full desktop:

```bash
# Check if Ubuntu Desktop is installed
dpkg -l ubuntu-desktop 2>/dev/null | grep -q "^ii" && echo "Ubuntu Desktop detected"

# Check if a desktop session is already running
ps aux | grep -E "gnome-session|gnome-shell|xfce4-session|lxsession" | grep -v grep
```

If **Ubuntu Desktop + GDM3** are present, use **Approach B** (GNOME native).
Otherwise, use **Approach A** (XFCE + xrdp).

---

## Approach A: XFCE4 + xrdp (lightweight, works anywhere)

Best for: servers, minimal installs, low-memory environments (~100 MB).

```
Windows (mstsc) ──RDP──> xrdp (port 3389) ──Xorg──> XFCE4 desktop
```

### 1. Install

```bash
apt-get update
apt-get install -y xfce4 xfce4-goodies xrdp
```

### 2. Configure session

xrdp calls `/etc/X11/Xsession` which honors `~/.xsession`:

```bash
echo "startxfce4" > ~/.xsession
chmod +x ~/.xsession
```

### 3. Handle port 3389 conflict

If `gnome-remote-desktop` pulled in as a dependency blocks port 3389:

```bash
ss -tlnp | grep 3389
systemctl stop gnome-remote-desktop.service 2>/dev/null
systemctl disable gnome-remote-desktop.service 2>/dev/null
```

### 4. Start services

```bash
systemctl enable --now xrdp
systemctl enable --now xrdp-sesman
```

### 5. Firewall

```bash
ufw allow 3389/tcp
```

### 6. (Optional) CJK fonts

```bash
apt-get install -y fonts-noto-cjk
```

---

## Approach B: GNOME native via vkms + gnome-remote-desktop

Best for: Ubuntu Desktop installations, users who want the full GNOME experience
with native file manager, settings, extensions, etc. Requires ~300-400 MB RAM.

```
Windows (mstsc) ──RDP──> gnome-remote-desktop (port 3389)
                              │
                    GNOME Shell (already running)
                              │
                    vkms virtual display @ 1920×1080
```

### Prerequisites

- Ubuntu Desktop 22.04+ (GNOME + GDM3 installed)
- Intel/AMD GPU with DRM/KMS support
- `vkms` kernel module available: `modinfo vkms`

### Key concept

Headless servers have no physical monitor → Xorg initializes at 1024×768
minimum. `vkms` (Virtual Kernel Mode Setting) creates a software virtual
display that GNOME treats as a real monitor, unlocking full resolutions.

### 1. Load vkms module

```bash
modprobe vkms
# Verify: a new DRM device appears
ls /dev/dri/card1
```

### 2. Connect virtual display to GPU

First check if vkms auto-connected (modern kernels 6.x+ with modesetting
driver often pick it up automatically):

```bash
sudo -u <user> DISPLAY=:0 xrandr | grep Virtual
```

If `Virtual-1-1` appears and shows `connected`, skip the manual setup below.
Otherwise, connect manually:

```bash
# Find the provider IDs
xrandr --listproviders
# Provider 0: Intel GPU (cap: Source Output)
# Provider 1: vkms (cap: Sink Output)
xrandr --setprovideroutputsource <sink_id> <source_id>
xrandr --output Virtual-1-1 --mode 1920x1080
```

**Pitfall:** Provider IDs change on each boot. When manual setup is needed,
use a dynamic discovery script (see `references/vkms-discover-script.sh`).

**Note:** If vkms was loaded *after* the desktop session started, auto-connect
may not happen — try reloading the session (log out/in) before falling back
to manual provider linking.

### 3. Enable gnome-remote-desktop RDP

Run as the desktop user (usually **not** root). First enable the RDP backend:

```bash
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\
  grdctl rdp enable

# Allow remote control (not view-only)
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\
  grdctl rdp disable-view-only

# Verify
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\
  grdctl status
```

Then enable and start the systemd user service (use `enable --now` for
first-time setup; `restart` for subsequent config changes):

```bash
systemctl --user -M <user>@ enable --now gnome-remote-desktop.service
```

### 4. Set RDP credentials (⚠️ keyring pitfall)

gnome-remote-desktop stores credentials in the GNOME/Secret Service keyring.
`grdctl rdp set-credentials <username> <password>` **hangs** when the keyring
is locked or unavailable — common in headless/auto-login setups.

**Workaround: environment variables via systemd drop-in**

The daemon honors two test env vars that bypass the keyring entirely.
Create a systemd user service override:

```bash
mkdir -p /home/<user>/.config/systemd/user/gnome-remote-desktop.service.d

cat > /home/<user>/.config/systemd/user/gnome-remote-desktop.service.d/override.conf << 'EOF'
[Service]
Environment=GNOME_REMOTE_DESKTOP_TEST_RDP_USERNAME=<username>
Environment=GNOME_REMOTE_DESKTOP_TEST_RDP_PASSWORD=<password>
EOF

chown -R <user>:<user> /home/<user>/.config/systemd
systemctl --user -M <user>@ daemon-reload
systemctl --user -M <user>@ restart gnome-remote-desktop.service
```

These env vars are prefixed `TEST_` in the binary's code but work identically
in production — the daemon checks them before looking up the keyring.

**Alternative: direct keyring write via secret-tool**

If the keyring is unlocked, you can store credentials with:

```bash
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  secret-tool store --label='RDP credentials' \
    protocol rdp \
    username <username> \
    password <password>
```

But this also hangs if the keyring needs unlocking — the env-var approach is
more reliable for headless setups.

### 5. Verify

```bash
ss -tlnp | grep 3389  # should show gnome-remote-de
sudo -u <user> xrandr | grep Virtual
```

### 6. Check daemon logs for connection issues

After a client attempts to connect, check the systemd journal for errors:

```bash
sudo journalctl -u user@$(id -u <user>).service --no-pager -n 100 | grep -i "remote\|rdp\|credential\|MIC\|NLA\|nego" | tail -20
```

### 7. Handle MIC / CredSSP incompatibility (Windows 10/11 clients)

If the Windows client errors with *"发生身份验证错误。给函数提供的标志无效"* (error code 0x0),
check the server logs for:

```
[ERROR][com.winpr.sspi.NTLM] - Message Integrity Check (MIC) verification failed!
```

This is a known CredSSP version mismatch between gnome-remote-desktop's bundled
FreeRDP 2.x and updated Windows 10/11 clients. The gnome-remote-desktop daemon
**only supports NLA Security** — there is no server-side toggle to enable RDP or
TLS fallback.

**Fix on the Windows client (two methods):**

**Method 1 — Group Policy** (Windows Pro/Enterprise):
```
Win+R → gpedit.msc
Computer Configuration → Administrative Templates → System → Credentials Delegation
→ "Encryption Oracle Remediation" → Enabled → Protection Level: Mitigated
```

**Method 2 — Registry** (all Windows editions including Home):
```
Path: HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters
DWORD: AllowEncryptionOracle = 2  (2=Force Updated Clients, 1=Mitigated)
```

After applying either fix, restart Windows and reconnect.

### 8. Make persistent

**Kernel module auto-load:**
```bash
grep -q vkms /etc/modules-load.d/vkms.conf 2>/dev/null || echo "vkms" | tee /etc/modules-load.d/vkms.conf
```

**GNOME autostart for virtual display (only needed if auto-connection fails):**
Create `~/.config/autostart/virtual-display.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Virtual Display Setup
Exec=/home/<user>/.local/bin/setup-virtual-display.sh
X-GNOME-Autostart-enabled=true
X-GNOME-Autostart-Delay=5
```

The script must dynamically discover provider IDs (see
`references/vkms-discover-script.sh`). Install it to
`/home/<user>/.local/bin/setup-virtual-display.sh` and mark executable.

---

## Decision matrix

| Factor | Approach A (XFCE+xrdp) | Approach B (GNOME+vkms) |
|--------|----------------------|------------------------|
| RAM usage | ~100 MB | ~300-400 MB |
| Desktop feel | Lightweight (Win7-like) | Full GNOME (modern) |
| Prerequisites | None | Ubuntu Desktop already installed |
| Virtual display | Built-in (xrdp creates one) | Requires vkms module |
| Resolution | Auto-adapts to client | Fixed 1920×1080 via vkms |
| File manager | Thunar | Nautilus |
| GPU acceleration | Software | Hardware (via i915) |
| Extension support | None | GNOME Shell extensions |

## Client connection

- **Windows:** Win+R → `mstsc` → enter server IP → username/password
- **Linux:** `xfreerdp /v:<ip> /u:<user>` or `rdesktop <ip>`
- **macOS:** Microsoft Remote Desktop from App Store

## Common pitfalls

| Issue | Fix |
|-------|-----|
| Port 3389 in use by wrong service | `ss -tlnp \| grep 3389` → kill/disable the conflictor |
| Root login fails (xrdp) | `AllowRootLogin=true` in `/etc/xrdp/sesman.ini` |
| Blank screen (xrdp) | Missing `~/.xsession` → ensure it contains `startxfce4` |
| Chinese chars as boxes | Install `fonts-noto-cjk` |
| Virtual display stuck at 1024×768 | Check if vkms is loaded (`lsmod | grep vkms`). If loaded but Virtual-1-1 absent, run xrandr provider setup. On modern kernels it may auto-connect — try log out/in first. |
| gnome-remote-desktop daemon inactive | `systemctl --user -M <user>@ enable --now gnome-remote-desktop.service` |
| vkms not available on custom kernel | Fall back to Approach A (XFCE+xrdp) |
| Provider IDs different after reboot | Use dynamic discovery script, not hardcoded IDs |
| grdctl set-credentials hangs | env-var workaround via systemd drop-in (see §4) |
| "Username is not set, denying client" | Credentials not stored — apply the env-var workaround |
| "Password is not set, denying client" | Same fix as above |
| Daemon dies after manual start | Use systemd user service, not background terminal |
| "MIC verification failed" / Windows error 0x0 on connect | Windows CredSSP fix: gpedit → Encryption Oracle Remediation → Mitigated, or registry `AllowEncryptionOracle=2` (see Step 7 in Approach B) |
| "server supports only NLA Security" in logs | gnome-remote-desktop is hardcoded to NLA only — no server-side toggle. Apply the Windows-side CredSSP fix above. |
