# 📓 Joplin Self-Hosted Sync via WebDAV on Windows

Sync your Joplin notes between a **Windows PC** (using a pendrive as storage) and an **Android device** over your local Wi-Fi network — no cloud service required.

---

## 🧠 How It Works

```
Joplin PC ──sync──▶ D:\Joplin (Pendrive) ◀──WebDAV──▶ Joplin Android
                         ▲
                   wsgidav server
                (runs silently on boot)
```

1. Joplin PC syncs notes to a folder on your pendrive (`D:\Joplin`)
2. A lightweight WebDAV server (`wsgidav`) serves that folder over your local network
3. Joplin Android connects to the WebDAV server and syncs notes

---

## 🛠️ Requirements

- Windows 10/11 PC
- Python 3.x installed ([python.org](https://python.org)) — check **"Add to PATH"** during install
- A USB pendrive (always plugged in when laptop is on)
- Joplin installed on both PC ([joplinapp.org](https://joplinapp.org)) and Android
- Both devices on the **same Wi-Fi network**

---

## 📦 Step 1 — Install wsgidav

Open **Command Prompt** and run:

```cmd
python -m pip install wsgidav cheroot
```

> ⚠️ If `pip` gives a launcher error, always use `python -m pip` instead of `pip` directly.

---

## 💾 Step 2 — Set Up the Pendrive Folder

Check your pendrive drive letter:

```cmd
wmic logicaldisk get name,description
```

Look for `Removable Disk` — note its letter (e.g. `D:`).

Create a dedicated Joplin sync folder on it:

```cmd
mkdir D:\Joplin
```

---

## 🌐 Step 3 — Test the WebDAV Server

Find the wsgidav executable path (for Windows Store Python):

```
C:\Users\<YourName>\AppData\Local\Packages\PythonSoftwareFoundation.Python.3.11_qbz5n2kfra8p0\LocalCache\local-packages\Python311\Scripts\wsgidav.exe
```

> Your path may differ slightly depending on Python version. Check with:
> ```cmd
> dir "%LOCALAPPDATA%\Packages\PythonSoftwareFoundation*\LocalCache\local-packages\Python3*\Scripts\wsgidav.exe" /s
> ```

Run the server manually to test:

```cmd
"<full-path-to-wsgidav.exe>" --host=0.0.0.0 --port=8080 --root=D:\Joplin --auth=anonymous
```

Open your browser and go to `http://localhost:8080` — you should see an empty directory listing. ✅  
Press `Ctrl+C` to stop it.

---

## 🤫 Step 4 — Create a Silent Background Launcher

Instead of a CMD window, use a VBScript to launch the server invisibly.

Open **Notepad**, paste the following, and save as `C:\Users\<YourName>\webdav-silent.vbs`  
*(In Save As dialog, set "Save as type" to **All Files**, not .txt)*

```vbscript
Set oShell = CreateObject("WScript.Shell")
oShell.Run """C:\Users\<YourName>\AppData\Local\Packages\PythonSoftwareFoundation.Python.3.11_qbz5n2kfra8p0\LocalCache\local-packages\Python311\Scripts\wsgidav.exe"" --host=0.0.0.0 --port=8080 --root=D:\Joplin --auth=anonymous", 0, False
```

> Replace `<YourName>` with your actual Windows username throughout this file.

Test it by double-clicking the `.vbs` file — **no window should appear**.  
Confirm the server is running:

```cmd
netstat -an | findstr 8080
```

You should see `0.0.0.0:8080 ... LISTENING` ✅

---

## ⚙️ Step 5 — Auto-Start on Boot via Task Scheduler

1. Press **Win + R** → type `taskschd.msc` → Enter
2. Click **Create Task** (not "Create Basic Task")
3. Configure each tab as follows:

### General Tab
- Name: `Joplin WebDAV`
- ✅ Run whether user is logged on or not
- ✅ Run with highest privileges
- Configure for: `Windows 10`

### Triggers Tab
- Click **New**
- Begin the task: **At startup**
- ✅ Delay task for: **30 seconds** *(lets Wi-Fi connect first)*

### Actions Tab
- Click **New**
- Action: **Start a program**
- Program/script: `wscript.exe`
- Arguments: `"C:\Users\<YourName>\webdav-silent.vbs"`

### Conditions Tab
- ❌ Uncheck **"Start the task only if the computer is on AC power"**

### Settings Tab
- ✅ If the task fails, restart every: **1 minute**

4. Click **OK** and enter your Windows password when prompted.

---

## 🔥 Step 6 — Allow Port 8080 Through Windows Firewall

Run in **Command Prompt as Administrator**:

```cmd
netsh advfirewall firewall add rule name="Joplin WebDAV" dir=in action=allow protocol=TCP localport=8080
```

---

## 📱 Step 7 — Configure Joplin Android

First, find your laptop's local IP address:

```cmd
ipconfig
```

Look for **IPv4 Address** under **Wi-Fi adapter** (e.g. `192.168.1.3`).

In **Joplin Android → ⚙️ Configuration → Synchronisation**:

| Field | Value |
|---|---|
| Sync Target | WebDAV |
| WebDAV URL | `http://192.168.1.3:8080/` |
| Username | *(leave blank)* |
| Password | *(leave blank)* |

Tap **Check synchronisation configuration** → should say ✅ OK  
Then tap **Synchronise**.

---

## 💻 Step 8 — Configure Joplin PC to Sync to Pendrive

In **Joplin PC → Tools → Options → Synchronisation**:

- **Sync Target:** File System
- **Directory:** `D:\Joplin`
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

Your laptop's local IP (`192.168.1.3`) can change after a router reboot, breaking the Android sync URL. Fix this in your router:

1. Open `http://192.168.1.1` in your browser (your router admin panel)
2. Find **DHCP Reservation** or **Address Reservation** (usually under LAN settings)
3. Bind your laptop's MAC address to `192.168.1.3` permanently

Find your laptop's MAC address:

```cmd
getmac /v
```

Look for the MAC on the **Wi-Fi** row (format: `D6-18-5B-F7-F6-05`).

---

## ✅ Verification Checklist

After a fresh reboot, run:

```cmd
netstat -an | findstr 8080
```

Expected output:
```
TCP    0.0.0.0:8080    0.0.0.0:0    LISTENING
```

If you see `LISTENING` — the server started automatically in the background. ✅

---

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| `pip` gives launcher error | Use `python -m pip` instead |
| `wsgidav` not recognized | Use full path to `wsgidav.exe` as shown in Step 3 |
| Android can't connect | Check firewall rule (Step 6) and confirm both devices are on same Wi-Fi |
| Notes don't appear on Android | Sync PC Joplin first (Step 8), then sync Android |
| Server doesn't start on boot | Check Task Scheduler history for errors; verify `.vbs` path is correct |
| Pendrive letter changed | Update `--root=` path in `webdav-silent.vbs` and Task Scheduler action |
| IP changed, Android can't sync | Reserve static IP in router (Step 10); update WebDAV URL in Joplin Android |

---

## 📁 File Structure

```
D:\Joplin\          ← Pendrive (WebDAV root)
  ├── .joplin/        ← Joplin sync metadata
  ├── *.md            ← Encrypted note files (if E2EE enabled)
  └── ...

C:\Users\<YourName>\
  └── webdav-silent.vbs   ← Silent server launcher
```

---

## 📌 Notes

- The pendrive **must be plugged in before the laptop boots** for the folder path to be valid
- This setup works **only on your local Wi-Fi network** — not over the internet
- `wsgidav` uses minimal CPU/RAM when idle (effectively 0% between syncs)
- Tested on Windows 10 (Build 26200) with Python 3.11.9 and Joplin 3.x

---

## 📜 License

MIT — use freely, no warranty.
