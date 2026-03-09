# Joplin LAN Sync — Linux Setup

Sync your Joplin notes between a **Linux PC** (using a pendrive as storage) and an **Android device** over your local Wi-Fi network — no cloud service required.

---

## 🧠 How It Works

```
Joplin PC ──sync──▶ /mnt/pendrive/Joplin ◀──WebDAV──▶ Joplin Android
                              ▲
                     wsgidav (systemd service,
                      runs silently on boot)
```

---

## 🛠️ Requirements

- A Linux PC (Debian/Ubuntu/Arch/Fedora or any distro)
- Python 3.x (usually pre-installed)
- `pip` or `pip3`
- A USB pendrive (always plugged in when PC is on)
- Joplin installed on both PC ([joplinapp.org](https://joplinapp.org)) and Android
- Both devices on the **same Wi-Fi network**

---

## 📦 Step 1 — Install wsgidav

```bash
pip3 install wsgidav cheroot
```

If `pip3` is not found:

```bash
# Debian/Ubuntu
sudo apt install python3-pip

# Arch
sudo pacman -S python-pip

# Fedora
sudo dnf install python3-pip
```

---

## 💾 Step 2 — Auto-Mount the Pendrive on Boot

Find your pendrive's UUID:

```bash
lsblk -f
```

Look for your pendrive (usually `sdb1` or `sdc1`) and copy its UUID.

Create a mount point:

```bash
sudo mkdir -p /mnt/pendrive
```

Add it to `/etc/fstab` so it mounts automatically on boot:

```bash
sudo nano /etc/fstab
```

Add this line at the bottom (replace `YOUR-UUID` with your actual UUID):

```
UUID=YOUR-UUID  /mnt/pendrive  auto  defaults,nofail,x-systemd.automount  0  0
```

> The `nofail` flag ensures Linux boots normally even if the pendrive is missing.

Apply the changes:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

Verify it mounted:

```bash
ls /mnt/pendrive
```

---

## 📁 Step 3 — Create the Joplin Sync Folder

```bash
mkdir -p /mnt/pendrive/Joplin
```

---

## 🌐 Step 4 — Test the WebDAV Server

```bash
wsgidav --host=0.0.0.0 --port=8080 --root=/mnt/pendrive/Joplin --auth=anonymous
```

Open your browser and go to `http://localhost:8080` — you should see an empty directory listing. ✅  
Press `Ctrl+C` to stop it.

> If `wsgidav` is not found, use the full path:
> ```bash
> python3 -m wsgidav --host=0.0.0.0 --port=8080 --root=/mnt/pendrive/Joplin --auth=anonymous
> ```

---

## ⚙️ Step 5 — Auto-Start on Boot via systemd

Find the full path of wsgidav:

```bash
which wsgidav
# e.g. /usr/local/bin/wsgidav or /home/<user>/.local/bin/wsgidav
```

Create a systemd service file:

```bash
sudo nano /etc/systemd/system/joplin-webdav.service
```

Paste the following (replace `/usr/local/bin/wsgidav` with your actual path from above):

```ini
[Unit]
Description=Joplin WebDAV Sync Server
After=local-fs.target network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/wsgidav --host=0.0.0.0 --port=8080 --root=/mnt/pendrive/Joplin --auth=anonymous
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable joplin-webdav.service
sudo systemctl start joplin-webdav.service
```

Verify it is running:

```bash
sudo systemctl status joplin-webdav.service
```

You should see `Active: active (running)` ✅

---

## 🔥 Step 6 — Allow Port 8080 Through Firewall

**UFW (Ubuntu/Debian):**
```bash
sudo ufw allow 8080/tcp
sudo ufw reload
```

**firewalld (Fedora/Arch):**
```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

---

## 📱 Step 7 — Configure Joplin Android

Find your laptop's local IP address:

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
# Look for something like 192.168.1.5
```

In **Joplin Android → ⚙️ Configuration → Synchronisation**:

| Field | Value |
|---|---|
| Sync Target | WebDAV |
| WebDAV URL | `http://192.168.1.5:8080/` |
| Username | *(leave blank)* |
| Password | *(leave blank)* |

Tap **Check synchronisation configuration** → should say ✅ OK  
Then tap **Synchronise**.

---

## 💻 Step 8 — Configure Joplin PC to Sync to Pendrive

In **Joplin PC → Tools → Options → Synchronisation**:

- **Sync Target:** File System
- **Directory:** `/mnt/pendrive/Joplin`
- Click **Apply** → Click **Synchronise Now**

Also set **Synchronisation interval** to `5 minutes` for automatic syncing.

---

## 🔒 Step 9 — Enable End-to-End Encryption (Recommended)

Joplin's built-in E2EE encrypts all note files before writing them to the pendrive — protecting data both at rest and in transit.

**On Joplin PC:**
1. Go to **Tools → Encryption → Enable Encryption**
2. Set a strong master password
3. Let it re-sync all notes

**On Joplin Android:**
1. Go to **Configuration → Encryption**
2. Enter the same master password
3. Tap Synchronise

> ⚠️ **Do not lose this password.** Notes are unrecoverable without it.

---

## 🌍 Step 10 — Reserve a Static Local IP (Important)

Your laptop's local IP can change after a router reboot, breaking the Android sync URL. Fix this in your router:

1. Open `http://192.168.1.1` in your browser
2. Find **DHCP Reservation** or **Address Reservation** (under LAN / DHCP settings)
3. Bind your laptop's MAC address to a fixed IP permanently

Find your MAC address:

```bash
ip link show
# Look for "link/ether" under your Wi-Fi interface (e.g. wlan0)
```

---

## ✅ Verification Checklist

After a fresh reboot:

```bash
# Check server is running
sudo systemctl status joplin-webdav.service

# Check port is listening
ss -tlnp | grep 8080
```

Expected output includes `0.0.0.0:8080` with state `LISTEN` ✅

---

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| `wsgidav` not found | Use `python3 -m wsgidav` or install with `pip3 install wsgidav cheroot` |
| Pendrive not mounting on boot | Double-check UUID in `/etc/fstab`; run `sudo mount -a` to test |
| Service fails to start | Run `journalctl -u joplin-webdav.service` to see error logs |
| Android can't connect | Check firewall (Step 6); confirm both devices on same Wi-Fi |
| Notes don't appear on Android | Sync PC Joplin first (Step 8), then sync Android |
| IP changed, Android can't sync | Reserve static IP in router (Step 10); update WebDAV URL in Joplin Android |
| Pendrive mount point empty | Run `sudo mount -a` or replug the pendrive |

---

## 📌 Notes

- The pendrive must be plugged in **before boot** for systemd to mount it correctly
- This setup works **only on your local Wi-Fi network** — not over the internet
- `wsgidav` uses near-zero CPU/RAM when idle
- Tested with wsgidav 4.3.x and Joplin 3.x

---

## 📜 License

MIT
