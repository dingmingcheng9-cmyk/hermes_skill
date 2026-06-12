---
name: linux-remote-desktop
category: devops
description: >-
  Set up remote desktop on a headless Linux server so it can be operated like
  Windows via RDP (mstsc) or VNC. Covers three approaches: (A) lightweight
  XFCE4 + xrdp for servers with no desktop, (B) native GNOME via vkms +
  gnome-remote-desktop for Ubuntu Desktop, and (C) x11vnc for sharing the
  existing GNOME session over VNC (no CredSSP issues).
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

**Note on built-in virtual displays:** On modern Intel GPUs with modesetting
driver and GNOME 42+, a virtual output `card1-Virtual-1` is often available
**without** loading vkms. Check `/sys/class/drm/card1-Virtual-1/status` — if
it shows `connected`, GNOME already has a virtual display. In that case you
can skip the vkms steps below and go directly to §3 (TLS cert).

### Key concept

Headless servers have no physical monitor → Xorg initializes at 1024×768
minimum. `vkms` (Virtual Kernel Mode Setting) creates a software virtual
display that GNOME treats as a real monitor, unlocking full resolutions.
But first check if the GPU driver already provides a virtual output — many
modern systems do, rendering vkms unnecessary.

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

### 3. Generate TLS certificate (required)

gnome-remote-desktop requires TLS for RDP connections. Generate a self-signed
certificate before enabling the service:

```bash
CERT_DIR=/home/<user>/.local/share/gnome-remote-desktop
mkdir -p "$CERT_DIR"

openssl req -x509 -nodes -newkey rsa:4096 \
  -keyout "$CERT_DIR/rdp-tls.key" \
  -out "$CERT_DIR/rdp-tls.crt" \
  -days 3650 -subj "/CN=$(hostname)" 2>/dev/null

chmod 600 "$CERT_DIR/rdp-tls.key"
chown -R <user>:<user> "$CERT_DIR"
```

**Pitfall:** Without TLS cert+key configured, the daemon starts but RDP
connections will fail silently or with cryptic TLS errors.

### 4. Enable gnome-remote-desktop RDP

Run as the desktop user (usually **not** root). Two equivalent methods:

**Method A — gsettings** (most reliable):
```bash
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  gsettings set org.gnome.desktop.remote-desktop.rdp tls-cert "$CERT_DIR/rdp-tls.crt"

sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  gsettings set org.gnome.desktop.remote-desktop.rdp tls-key "$CERT_DIR/rdp-tls.key"

sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  gsettings set org.gnome.desktop.remote-desktop.rdp enable true

sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  gsettings set org.gnome.desktop.remote-desktop.rdp view-only false
```

**Method B — grdctl** (simpler but requires working keyring):
```bash
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  grdctl rdp enable

sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  grdctl rdp disable-view-only

# Set TLS cert/key (grdctl may not expose this — fall back to gsettings if needed)
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  gsettings set org.gnome.desktop.remote-desktop.rdp tls-cert "$CERT_DIR/rdp-tls.crt"
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  gsettings set org.gnome.desktop.remote-desktop.rdp tls-key "$CERT_DIR/rdp-tls.key"

# Verify
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \\\
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \\\
  grdctl status
```

**Note:** `gsettings` is preferred for TLS and `view-only` settings because `grdctl`
subcommands for TLS are limited. `grdctl` is preferred for status checks and credentials.

Then enable and start the systemd user service (use `enable --now` for
first-time setup; `restart` for subsequent config changes):

**Method 1 — via machinectl** (requires `systemd-container` / `machinectl`):
```bash
systemctl --user -M <user>@ enable --now gnome-remote-desktop.service
```

**Method 2 — via env var injection** (fallback, works without machinectl):
```bash
sudo -u <user> \
  XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  systemctl --user enable --now gnome-remote-desktop.service
```
If Method 1 fails with `machinectl: command not found`, install `systemd-container` or
fall back to Method 2. The env-var approach manually points `systemctl --user` to
the correct D-Bus session bus.

### 5. Set RDP credentials (⚠️ keyring pitfall)

gnome-remote-desktop stores credentials in the GNOME/Secret Service keyring.
`grdctl rdp set-credentials <username> <password>` **works** when run as the
desktop user with the correct DBUS session:

```bash
sudo -u <user> \
  XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  grdctl rdp set-credentials <username> <password>
```

It **hangs** when the keyring is locked or the DBUS env is wrong (common when
running from a root terminal or SSH without proper DBUS inheritance).

**Workaround when keyring is unavailable: environment variables via systemd drop-in**

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

**⚠️ Pitfall: dual credential sources must stay in sync.** If you set credentials
via both `grdctl rdp set-credentials` (keyring) AND the override.conf env vars,
and they differ, RDP will fail with authentication errors. Keep them consistent:

```bash
# After changing one, update the other
sudo -u <user> \
  XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  grdctl rdp set-credentials <username> <new_password>

# Then update override.conf
sed -i 's/GNOME_REMOTE_DESKTOP_TEST_RDP_PASSWORD=.*/GNOME_REMOTE_DESKTOP_TEST_RDP_PASSWORD=<new_password>/' \
  /home/<user>/.config/systemd/user/gnome-remote-desktop.service.d/override.conf

sudo -u <user> \
  XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  systemctl --user daemon-reload

sudo -u <user> \
  XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  systemctl --user restart gnome-remote-desktop.service
```

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

### 6. Verify

```bash
ss -tlnp | grep 3389  # should show gnome-remote-de
sudo -u <user> xrandr | grep Virtual
```

**Quick env-var injection check** — verify the daemon picked up credentials:
```bash
# Find the daemon PID
PID=$(pgrep -u <user> -f gnome-remote-desktop-daemon)
# Check env vars are present
cat /proc/$PID/environ 2>/dev/null | tr '\0' '\n' | grep TEST_RDP
# Should show TEST_RDP_USERNAME and TEST_RDP_PASSWORD
```

If the env vars don't appear, the systemd drop-in wasn't loaded — double-check:
```bash
# Verify the override is recognized
systemctl --user -M <user>@ status gnome-remote-desktop.service | grep Drop-In
# Should list the override.conf path
```

### 7. Check daemon logs for connection issues

After a client attempts to connect, check the systemd journal for errors:

```bash
sudo journalctl -u user@$(id -u <user>).service --no-pager -n 100 | grep -i "remote\|rdp\|credential\|MIC\|NLA\|nego" | tail -20
```

**Verify env var support in the binary** — if the `TEST_RDP` vars aren't being picked
up, confirm the daemon actually supports them:
```bash
strings /usr/libexec/gnome-remote-desktop-daemon | grep -i TEST_RDP
# Should show both GNOME_REMOTE_DESKTOP_TEST_RDP_USERNAME and _PASSWORD
```

If they're absent, the daemon was built without the env-var workaround — fall back
to `secret-tool` or xrdp instead.**journalctl note for root users:** `journalctl --user -u gnome-remote-desktop.service` won't
work from root. Always use `sudo journalctl -u user@$(id -u <user>).service` when
running as root.

### 8. Handle MIC / CredSSP incompatibility (Windows 10/11 clients)

**Before diving into the CredSSP fix — check if the desktop is just asleep.**

A very common cause of RDP failure that mimics CredSSP symptoms: when the GNOME
desktop goes idle/sleep/locked with no physical monitor, gnome-remote-desktop's
RDP negotiation fails partway through — the mstsc window flashes and disappears
without a clear error. This is NOT a CredSSP issue.

Quick test: connect with x11vnc/TigerVNC first. If VNC connects and wakes the
session, then disconnect and immediately try mstsc — if it works, the problem
was desktop sleep, not CredSSP. Permanently disable sleep (see §9 → Preventing
desktop sleep) and mstsc will connect directly.

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

### 9. Fallback: x11vnc when RDP fails with Windows

If the Windows-side CredSSP fix (§8) doesn't resolve the issue, or if you
prefer a solution that works out of the box with any Windows edition (including
Home, which lacks gpedit.msc), switch to **x11vnc**.

x11vnc captures the **running** GNOME display (`:0`) and serves it via the VNC
protocol (port 5900). It shares the same native desktop as g-r-d but avoids
the CredSSP/MIC compatibility issue entirely — VNC uses the simpler RFB protocol.

**x11vnc also works as a desktop wake-up tool:** if the GNOME desktop is sleeping
and mstsc (RDP) can't connect, connecting via x11vnc first wakes up the session.
Then disconnect VNC and use mstsc immediately — RDP will work. This dual approach
lets you keep both services running: x11vnc on 5900 and g-r-d on 3389.

Key differences from alternatives:

| vs g-r-d RDP | vs xrdp |
|---|---|
| No CredSSP issues — works with ALL Windows versions | Shares the SAME GNOME session (not a new one) |
| VNC is unencrypted (fine on LAN, tunnel SSH for WAN) | No extra desktop to launch/maintain |
| Port 5900 instead of 3389 | No extra memory for a second X session |

Full setup instructions in `references/x11vnc-setup.md` (install, password,
systemd user service, firewall, Windows client steps).

**Summary of commands:**

```bash
apt-get install -y x11vnc
x11vnc -storepasswd <password> /home/<user>/.vnc/passwd
chown -R <user>:<user> /home/<user>/.vnc
# Create systemd user service (see reference file for full ExecStart)
# Then:
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  systemctl --user enable --now x11vnc.service
ufw allow 5900/tcp
```

**Performance tuning:** x11vnc can feel laggy by default. Add these flags to
ExecStart for much better LAN performance:
```
-ncache 10 -ncache_cr -wait 30 -defer 5 -xrandr
```
See `references/x11vnc-setup.md` → Performance Tuning for details.

**Preventing desktop sleep (so RDP works without VNC wake-up):**

If both x11vnc (VNC) and gnome-remote-desktop (RDP) are running, and you want
mstsc to connect directly without needing VNC to wake the session first:

```bash
sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  gsettings set org.gnome.desktop.screensaver lock-enabled false

sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  gsettings set org.gnome.desktop.screensaver idle-activation-enabled false

sudo -u <user> XDG_RUNTIME_DIR=/run/user/$(id -u <user>) \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus \
  gsettings set org.gnome.desktop.session idle-delay 0

# Disable DPMS (screen power management)
sudo -u <user> DISPLAY=:0 xset s off -dpms
```

Add `ExecStartPre=/usr/bin/xset s off -dpms` to the x11vnc systemd unit so
DPMS is disabled on boot (note: requires `Environment=DISPLAY=:0` too).

**Pitfall:** `x11vnc -noreset` flag was REMOVED in x11vnc 0.9.16+. Including it
causes the service to fail with exit code 1. Check flags before creating the
service file — use `x11vnc -opts` to inspect supported options.

### 10. Make persistent

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

| Factor | Approach A (XFCE+xrdp) | Approach B (GNOME+vkms+RDP) | Plan C (x11vnc) |
|--------|----------------------|------------------------|-----------------|
| RAM usage | ~100 MB | ~300-400 MB | ~10 MB (adds to existing GNOME) |
| Desktop feel | Lightweight (Win7-like) | Full GNOME (modern) | Same running GNOME session |
| Prerequisites | None | Ubuntu Desktop already installed, GPU with modesetting | Ubuntu Desktop already installed |
| Virtual display | Built-in (xrdp creates one) | Requires vkms or built-in card1-Virtual-1 | Shares existing display (:0) |
| Resolution | Auto-adapts to client | Fixed 1920×1080 via vkms | Native display resolution |
| Shares current session | ❌ (new X session) | ✅ (same GNOME) | ✅ (same GNOME) |
| Protocol | RDP | RDP (CredSSP) | VNC (no CredSSP) |
| Win11 Home compatible | ✅ | ❌ (CredSSP issues) | ✅ |
| Encryption | Built-in (TLS) | Built-in (TLS) | None (SSH tunnel for WAN) |
| Client needed on Windows | mstsc (built-in) | mstsc (built-in) | TigerVNC / RealVNC (download) |
| File manager | Thunar | Nautilus | Nautilus (shared) |
| Extensions | None | GNOME Shell extensions | Same GNOME extensions |

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
| gnome-remote-desktop daemon inactive | `systemctl --user -M <user>@ enable --now ...` or fallback env-var method (see §4 Method 2) |
| vkms not available on custom kernel | Fall back to Approach A (XFCE+xrdp) |
| Provider IDs different after reboot | Use dynamic discovery script, not hardcoded IDs |
| grdctl set-credentials hangs | Use correct DBUS (`DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u <user>)/bus`) or env-var workaround via systemd drop-in (see §5) |
| "Username is not set, denying client" | Credentials not stored — apply the env-var workaround |
| "Password is not set, denying client" | Same fix as above |
| daemon dies after manual start | Use systemd user service, not background terminal |
| Keyring + override.conf credentials out of sync | Update both sources: run `grdctl rdp set-credentials` AND edit the password in override.conf (see §5) |
| "MIC verification failed" / Windows error 0x0 on connect | Windows CredSSP fix: gpedit → Encryption Oracle Remediation → Mitigated, or registry `AllowEncryptionOracle=2` (see Step 7 in Approach B) |
| "server supports only NLA Security" in logs | gnome-remote-desktop is hardcoded to NLA only — no server-side toggle. Apply the Windows-side CredSSP fix above or switch to x11vnc. |
|| x11vnc service exits with code 1 | Check for removed flags (`-noreset` was removed in 0.9.16+). Run `x11vnc -opts` to list valid options. |
|| x11vnc black screen / wrong display | Ensure `-display :0` targets the correct running X session. Verify with `who` and `loginctl`. |
|| x11vnc "Authentication failed" | Password file was set for wrong user or with wrong tool. Re-run `x11vnc -storepasswd <pass> /home/<user>/.vnc/passwd` as the desktop user. |
|| VNC laggy over LAN | Add `-ncache 10 -ncache_cr -wait 30 -defer 5 -xrandr` to x11vnc ExecStart (see §9 Performance tuning). On the client side, use Tight/ZRLE encoding and 16-bit color depth. |
|| mstsc window flashes and disappears (no error) | Desktop asleep/locked. Connect VNC first to wake the session, or disable desktop sleep permanently (see §9 → Preventing desktop sleep). |
|| VNC connects but RDP fails until VNC is used first | Desktop idle state blocks RDP negotiation. Fix: `gsettings set org.gnome.desktop.screensaver lock-enabled false` + `xset s off -dpms` (see §9). |
|| x11vnc ExecStartPre=xset fails at boot | xset needs DISPLAY=:0. Add `Environment=DISPLAY=:0` to [Service] section. |
