# joplin-lan-sync

Sync your Joplin notes across devices over your **local Wi-Fi network** — no cloud service, no subscription, no internet required.

Uses a lightweight **WebDAV server** running on your PC to serve a local folder (optionally on a pendrive) that Joplin Android syncs with.

---

## 💡 How It Works

```
Joplin PC ──sync──▶ Local Folder / Pendrive ◀──WebDAV──▶ Joplin Android
                              ▲
                     WebDAV server (runs silently on boot)
```

- Joplin on your PC syncs notes to a local folder
- A WebDAV server serves that folder over your LAN
- Joplin on Android connects to the server and syncs

---

## ✨ Features

- 🔒 Fully local — no data leaves your network
- ⚡ Lightweight — near-zero CPU/RAM usage when idle
- 🔐 Compatible with Joplin's built-in End-to-End Encryption
- 🚀 Auto-starts silently on boot
- 💾 Works with a pendrive as portable storage

---

## 📖 Setup Guides

<!-- Add your platform guides here -->

- Windows — ([Find here](https://joplinapp.org](https://github.com/myst-25/joplin-webdav-tutorial/blob/main/Joplin-Windows-server.md))
- Linux — *coming soon*
- macOS — *coming soon*

---

## 📋 Requirements

- Joplin installed on PC ([joplinapp.org](https://joplinapp.org))
- Joplin installed on Android ([Playstore](https://play.google.com/store/apps/details?id=net.cozic.joplin&hl=en-US))
- Both devices on the same Wi-Fi network
- Python 3.x (for wsgidav WebDAV server)

---

## 📌 Notes

- Your PC must be **on and connected to Wi-Fi** for Android to sync
- Tested with Joplin 3.x and wsgidav 4.3.x
- Enable Joplin's built-in **E2EE** (Tools → Encryption) to encrypt notes at rest and in transit

---

## 📜 License

MIT
