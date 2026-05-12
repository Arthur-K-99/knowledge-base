Here is **Version 1.3** of the Master Documentation.

I have added a completely new section at the top for **Connection & Identity Management** using your exact IP and SSH key details. I also added the **RCON setup instructions** and embedded the **Backup Restoration SOP** into the Disaster Recovery section.

Overwrite your file with this ultimate version.

---

# Project: GregTech New Horizons (ARM64 Architecture)

**Deployment Target:** Oracle Cloud Infrastructure (Ampere A1 Compute)
**OS:** Ubuntu 24.04 LTS (Minimal)
**Runtime:** Java 25 (OpenJDK/Temurin) + ZGC
**Operator:** Athanasios Kompouras
**Server IP:** 129.213.126.1

---

## 1. Architecture Overview

This server utilizes the Oracle Cloud Free Tier "Ampere" instance. It is highly optimized for the 1.7.10 JVM environment using experimental Java 25 features to minimize heap overhead.

| Component   | Specification               | Notes                                                             |
| ----------- | --------------------------- | ----------------------------------------------------------------- |
| **CPU**     | 4 OCPU (Ampere Altra ARM64) | Single-thread performance is adequate; multi-thread is excellent. |
| **RAM**     | 24 GB                       | Allocated: 12GB Min / 16GB Max Heap.                              |
| **Java**    | OpenJDK 25 (Early Access)   | Uses `CompactObjectHeaders` to reduce RAM usage by ~15%.          |
| **Storage** | 100 GB Boot Volume          | Standard Performance (10 VPU).                                    |
| **Network** | 1 Gbps Public               | Port 25565 (Game), 25575 (RCON).                                  |

---

## 2. Connection & Identity Management

### 2.1 SSH Access

To connect to the server from your local terminal/command prompt, use the private key (`main`):

```bash
ssh -i main ubuntu@129.213.126.1

```

### 2.2 The Two-User System

For security, this server uses a dual-identity architecture.

1. **`ubuntu` (The Admin):** This is the user you log into. It has `sudo` privileges to manage the OS, firewall, and system services.
2. **`gtnh` (The Service):** This user runs the Minecraft server. **It has no password** and cannot be logged into directly from the internet.

**To run commands as the Minecraft server (Backups, World Editing, Rclone):**

```bash
sudo su - gtnh

```

_(To return to the `ubuntu` user, type `exit`)_.

---

## 3. RCON (Remote Console) Configuration

RCON allows you to send server commands (whitelist, kick, give, time set) from your PC without logging into SSH or opening the game.

### 3.1 Enabling RCON

1. Open your server properties: `nano /home/gtnh/server.properties`
2. Ensure these lines are set:

```properties
enable-rcon=true
rcon.port=25575
rcon.password=SetAStrongPasswordHere

```

3. Restart the server: `sudo systemctl restart gtnh`

### 3.2 Connecting to RCON

- **Tool:** Download an RCON client like [MCRcon](https://github.com/Tiiffi/mcrcon) (CLI) or a GUI like RustAdmin/Icecon.
- **IP:** `129.213.126.1`
- **Port:** `25575`
- **Password:** Use the password set in `server.properties`.

---

## 4. Configuration Files (The "Golden Configs")

### 4.1 Systemd Service (`/etc/systemd/system/gtnh.service`)

This controls the auto-restart logic and JVM flags.

- **Critical Flag:** `-Djava.system.class.loader=...` (Prevents the `RetroFuturaBootstrap` crash).

```ini
[Unit]
Description=GregTech New Horizons Server
After=network.target

[Service]
User=gtnh
Group=gtnh
WorkingDirectory=/home/gtnh

# BOOT COMMAND
ExecStart=/opt/java25/bin/java \
    -Xms12G -Xmx16G \
    -Djava.system.class.loader=com.gtnewhorizons.retrofuturabootstrap.RfbSystemClassLoader \
    -XX:+UnlockExperimentalVMOptions -XX:+UseZGC \
    -XX:+UseCompactObjectHeaders \
    -XX:+DisableExplicitGC -XX:+AlwaysPreTouch \
    -XX:+PerfDisableSharedMem -XX:+UseNUMA \
    -jar lwjgl3ify-forgePatches.jar nogui

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

```

### 4.2 Backup Script & Rclone (`/home/gtnh/push-backups.sh`)

Handles off-site replication to Google Drive via Rclone.

**Critical Setup Step:** Rclone config is user-specific. The config file must exist in `/home/gtnh/.config/rclone/`.

```bash
# Fix Permissions Command
sudo mkdir -p /home/gtnh/.config/rclone
sudo cp /home/ubuntu/.config/rclone/rclone.conf /home/gtnh/.config/rclone/
sudo chown -R gtnh:gtnh /home/gtnh/.config/

```

**The Script:** Runs via cron at 04:30 daily (`crontab -e` as `gtnh` user).

```bash
#!/bin/bash
LOCAL_DIR="/home/gtnh/backups"
REMOTE_NAME="gdrive"
REMOTE_DIR="GTNH_Backups"
RETENTION_DAYS="60"

rclone copy $LOCAL_DIR $REMOTE_NAME:$REMOTE_DIR --transfers=4 --checkers=8 --stats=1m -v
if [ $? -eq 0 ]; then
    rclone delete --min-age ${RETENTION_DAYS}d $REMOTE_NAME:$REMOTE_DIR
fi

```

---

## 5. Networking & Firewall

### 5.1 The "Two-Firewall" Reality

Traffic must pass two distinct layers to reach the application.

1. **Oracle Cloud VCN:** Ingress Rule TCP 25565 and 25575.
2. **Ubuntu IPTables:** `sudo iptables -I INPUT 1 -p tcp --dport 25565 -j ACCEPT`

### 5.2 MTU Optimization (The "Jumbo Frame" Fix)

Oracle Cloud defaults to MTU 9000. Public internet uses MTU 1500. This mismatch causes packet loss and lag.

- **The Fix:** Enforce 1500 via Netplan in `/etc/netplan/99-custom-mtu.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s6:
      mtu: 1500
      dhcp4: true
```

- **Apply:** `sudo netplan apply`

---

## 6. Migration Procedures (SOP)

### 6.1 Singleplayer -> Server Migration

| Data Type     | Symptom                    | Fix Method                                                                                                          |
| ------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Inventory** | Log in with empty pockets. | Extract `Player` tag from local `level.dat` using NBTExplorer. Save as `<UUID>.dat`. Upload to `world/playerdata/`. |
| **Quests**    | Quest book is empty/reset. | Edit `QuestingData.json`. Use `sed` to replace Old UUID with New UUID globally.                                     |
| **Waypoints** | JourneyMap is blank.       | **Client-side fix.** Copy folder `journeymap/data/sp/<WorldName>` to `journeymap/data/mp/<ServerIP>`.               |

---

## 7. Disaster Recovery & Backups

### 7.1 Server Crash Loop Recovery

1. **Stop Service:** `sudo systemctl stop gtnh`
2. **Check Logs:** `cat /home/gtnh/crash-reports/crash-<latest>.txt`
3. **Clear Lock:** `rm /home/gtnh/world/session.lock` (Common cause after hard crash).
4. **Permissions:** `sudo chown -R gtnh:gtnh /home/gtnh/`

### 7.2 Restoring a Backup (SOP)

1. **Stop Server:** `sudo systemctl stop gtnh`
2. **Quarantine Old World:** `mv /home/gtnh/world /home/gtnh/world_corrupted`
3. **Fetch Backup:**

- From Local: `cp /home/gtnh/backups/YOUR_BACKUP.zip /home/gtnh/restore.zip`
- From Cloud: `sudo su - gtnh` -> `rclone copy gdrive:GTNH_Backups/YOUR_BACKUP.zip /home/gtnh/restore.zip`

4. **Extract:**

```bash
    mkdir /home/gtnh/temp_restore
    unzip /home/gtnh/restore.zip -d /home/gtnh/temp_restore
```

5.  **Move Files:** Move the extracted `world` folder (or contents) to `/home/gtnh/world`.
6.  **Fix Permissions (Critical):** `sudo chown -R gtnh:gtnh /home/gtnh/world`
7.  **Start Server:** `sudo systemctl start gtnh`

---

## 8. Cost & Compliance (The "Free Tier" Rules)

To ensure the bill remains $0.00:

1.  **Instance Count:** Maximum **1** VM active. Terminate all others.
2.  **Public IP:** Must be attached to the running VM.
3.  **Boot Volume:** **No Backup Policy** assigned.
4.  **Volume Performance:** Set to **Balanced (10 VPU)**.
5.  **Timezone:** Server set to `America/New_York` (`timedatectl`).

---

**Next Service Date:** Indefinite.

**Documentation Updated:** Jan 21, 2026 (v1.3 - Added SSH, Users, RCON, & Restore SOP).
