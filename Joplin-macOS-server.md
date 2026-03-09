# Joplin LAN Sync — macOS Setup

Sync your Joplin notes between a **Mac** (using a pendrive as storage) and an **Android device** over your local Wi-Fi network — no cloud service required.

---

## 🧠 How It Works

```
Joplin Mac ──sync──▶ /Volumes/Pendrive/Joplin ◀──WebDAV──▶ Joplin Android
                                ▲
                       wsgidav (LaunchAgent,
                        runs silently on boot)
```

---

## 🛠️ Requirements

- macOS 11 (Big Sur) or later
- Python 3.x — install via [Homebrew](https://brew.sh) (recommended)
- A USB pendrive (always plugged in when Mac is on)
- Joplin installed on both Mac ([joplinapp.org](https://joplinapp.org)) and Android
- Both devices on the **same Wi-Fi network**

---

## 📦 Step 1 — Install Python & wsgidav

If you don't have Homebrew:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Install Python via Homebrew:

```bash
brew install python
```

Install wsgidav:

```bash
pip3 install wsgidav cheroot
```

---

## 💾 Step 2 — Find Your Pendrive Mount Path

On macOS, pendrives mount automatically under `/Volumes/`. Find yours:

```bash
ls /Volumes/
```

You'll see something like `/Volumes/MyDrive` or `/Volumes/USB`. Note the exact name — it is case-sensitive.

---

## 📁 Step 3 — Create the Joplin Sync Folder

```bash
mkdir -p /Volumes/MyDrive/Joplin
```

> Replace `MyDrive` with your actual pendrive name from Step 2.

---

## 🌐 Step 4 — Test the WebDAV Server

```bash
wsgidav --host=0.0.0.0 --port=8080 --root=/Volumes/MyDrive/Joplin --auth=anonymous
```

Open your browser and go to `http://localhost:8080` — you should see an empty directory listing. ✅  
Press `Ctrl+C` to stop it.

> If `wsgidav` is not found, use:
> ```bash
> python3 -m wsgidav --host=0.0.0.0 --port=8080 --root=/Volumes/MyDrive/Joplin --auth=anonymous
> ```

---

## ⚙️ Step 5 — Auto-Start on Login via LaunchAgent

macOS uses **LaunchAgents** instead of systemd to run background services.

Find the full path of wsgidav:

```bash
which wsgidav
# e.g. /usr/local/bin/wsgidav or /opt/homebrew/bin/wsgidav (Apple Silicon)
```

Create the LaunchAgent plist file:

```bash
nano ~/Library/LaunchAgents/com.joplin.webdav.plist
```

Paste the following (replace `/usr/local/bin/wsgidav` with your actual path and `MyDrive` with your pendrive name):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.joplin.webdav</string>

    <key>ProgramArguments</key>
    <array>
      <string>/usr/local/bin/wsgidav</string>
      <string>--host=0.0.0.0</string>
      <string>--port=8080</string>
      <string>--root=/Volumes/MyDrive/Joplin</string>
      <string>--auth=anonymous</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/joplin-webdav.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/joplin-webdav-error.log</string>
  </dict>
</plist>
```

Load the LaunchAgent:

```bash
launchctl load ~/Library/LaunchAgents/com.joplin.webdav.plist
```

Verify it is running:

```bash
launchctl list | grep joplin
```

You should see `com.joplin.webdav` in the list ✅

---

## 🔥 Step 6 — Allow Port 8080 Through macOS Firewall

macOS firewall is usually off by default for outgoing connections. If you have it enabled:

1. Go to **System Settings → Network → Firewall**
2. Click **Options**
3. Add `wsgidav` to the allowed apps list

Or allow it via terminal:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add $(which wsgidav)
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp $(which wsgidav)
```

---

## 📱 Step 7 — Configure Joplin Android

Find your Mac's local IP address:

```bash
ipconfig getifaddr en0
# en0 is usually Wi-Fi on Mac
# Try en1 if en0 gives nothing
```

In **Joplin Android → ⚙️ Configuration → Synchronisation**:

| Field | Value |
|---|---|
| Sync Target | WebDAV |
| WebDAV URL | `http://192.168.1.x:8080/` |
| Username | *(leave blank)* |
| Password | *(leave blank)* |

Tap **Check synchronisation configuration** → should say ✅ OK  
Then tap **Synchronise**.

---

## 💻 Step 8 — Configure Joplin Mac to Sync to Pendrive

In **Joplin Mac → Preferences → Synchronisation**:

- **Sync Target:** File System
- **Directory:** `/Volumes/MyDrive/Joplin`
- Click **Apply** → Click **Synchronise Now**

Also set **Synchronisation interval** to `5 minutes` for automatic syncing.

---

## 🔒 Step 9 — Enable End-to-End Encryption (Recommended)

Joplin's built-in E2EE encrypts all note files before writing them to the pendrive — protecting data both at rest and in transit.

**On Joplin Mac:**
1. Go to **Preferences → Encryption → Enable Encryption**
2. Set a strong master password
3. Let it re-sync all notes

**On Joplin Android:**
1. Go to **Configuration → Encryption**
2. Enter the same master password
3. Tap Synchronise

> ⚠️ **Do not lose this password.** Notes are unrecoverable without it.

---

## 🌍 Step 10 — Reserve a Static Local IP (Important)

Your Mac's local IP can change after a router reboot, breaking the Android sync URL. Fix this in your router:

1. Open `http://192.168.1.1` in your browser
2. Find **DHCP Reservation** or **Address Reservation** (under LAN / DHCP settings)
3. Bind your Mac's MAC address to a fixed IP permanently

Find your Mac's Wi-Fi MAC address:

```bash
networksetup -getmacaddress en0
```

---

## ✅ Verification Checklist

After a fresh reboot or login:

```bash
# Check LaunchAgent is running
launchctl list | grep joplin

# Check port is listening
lsof -i :8080
```

You should see `wsgidav` listening on port `8080` ✅

Check the logs if something goes wrong:

```bash
cat /tmp/joplin-webdav.log
cat /tmp/joplin-webdav-error.log
```

---

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| `wsgidav` not found | Run `brew install python` then `pip3 install wsgidav cheroot` |
| Pendrive path not found | Run `ls /Volumes/` to confirm the exact pendrive name |
| LaunchAgent not starting | Run `launchctl load ~/Library/LaunchAgents/com.joplin.webdav.plist` again |
| Android can't connect | Check firewall (Step 6); confirm both devices on same Wi-Fi |
| Notes don't appear on Android | Sync Mac Joplin first (Step 8), then sync Android |
| IP changed, Android can't sync | Reserve static IP in router (Step 10); update WebDAV URL in Joplin Android |
| Pendrive ejected unexpectedly | Replug pendrive; restart LaunchAgent: `launchctl kickstart gui/$(id -u)/com.joplin.webdav` |
| Apple Silicon Mac (M1/M2/M3) | wsgidav path is `/opt/homebrew/bin/wsgidav` instead of `/usr/local/bin/wsgidav` |

---

## 📌 Notes

- The pendrive must be plugged in **before login** for the folder path to be valid
- On **Apple Silicon Macs** (M1/M2/M3), Homebrew installs to `/opt/homebrew/` instead of `/usr/local/`
- This setup works **only on your local Wi-Fi network** — not over the internet
- `wsgidav` uses near-zero CPU/RAM when idle
- Tested with wsgidav 4.3.x and Joplin 3.x

---

## 📜 License

MIT
