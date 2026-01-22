# Project: GregTech New Horizons (ARM64 Architecture)

**Deployment Target:** Oracle Cloud Infrastructure (Ampere A1 Compute)

**OS:** Ubuntu 24.04 LTS (Minimal)

**Runtime:** Java 25 (OpenJDK/Temurin) + ZGC

**Operator:** Athanasios Kompouras

---

## 1. Architecture Overview

This server utilizes the Oracle Cloud Free Tier "Ampere" instance. It is highly optimized for the 1.7.10 JVM environment using experimental Java 25 features to minimize heap overhead.

|**Component**|**Specification**|**Notes**|
|---|---|---|
|**CPU**|4 OCPU (Ampere Altra ARM64)|Single-thread performance is adequate; multi-thread is excellent.|
|**RAM**|24 GB|Allocated: 12GB Min / 16GB Max Heap.|
|**Java**|OpenJDK 25 (Early Access)|Uses `CompactObjectHeaders` to reduce RAM usage by ~15%.|
|**Storage**|100 GB Boot Volume|Standard Performance (10 VPU).|
|**Network**|1 Gbps Public|Port 25565 (Game), 25575 (RCON/Local).|

---

## 2. Environment Setup

### 2.1 Java 25 Installation (Manual)

Ubuntu 24.04 repos do not contain Java 25. We installed it manually to `/opt`.

- **Source:** Eclipse Temurin / Adoptium (Nightly/EA builds).
    
- **Path:** `/opt/java25`
    
- **Verification:** `/opt/java25/bin/java -version`
    

### 2.2 User & Permissions

- **Service User:** `gtnh` (No sudo access, dedicated home directory).
    
- **Root Folder:** `/home/gtnh/`
    
- **Ownership Fix:** If permissions break: `sudo chown -R gtnh:gtnh /home/gtnh`
    

---

## 3. Configuration Files (The "Golden Configs")

### 3.1 Systemd Service (`/etc/systemd/system/gtnh.service`)

This controls the auto-restart logic and JVM flags.

- **Critical Flag 1:** `-Djava.system.class.loader=...` (Prevents the `RetroFuturaBootstrap` crash).
    
- **Critical Flag 2:** `-XX:+UseCompactObjectHeaders` (The memory saver).
    
- **Critical Flag 3:** `-XX:+UseZGC` (Low latency GC).
    

Ini, TOML

```
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

### 3.2 Backup Script & Rclone (`/home/gtnh/push-backups.sh`)

Handles off-site replication to Google Drive via Rclone.

**Critical Setup Step:** Rclone config is user-specific. Since the script runs as `gtnh`, the config file must exist in `/home/gtnh/.config/rclone/`.

Bash

```
# Fix Permissions Command
sudo mkdir -p /home/gtnh/.config/rclone
sudo cp /home/ubuntu/.config/rclone/rclone.conf /home/gtnh/.config/rclone/
sudo chown -R gtnh:gtnh /home/gtnh/.config/
```

**The Script:**

Bash

```
#!/bin/bash
LOCAL_DIR="/home/gtnh/backups"
REMOTE_NAME="gdrive"
REMOTE_DIR="GTNH_Backups"
RETENTION_DAYS="60"

# Sync to Cloud
rclone copy $LOCAL_DIR $REMOTE_NAME:$REMOTE_DIR --transfers=4 --checkers=8 --stats=1m -v

# Prune Old Files
if [ $? -eq 0 ]; then
    rclone delete --min-age ${RETENTION_DAYS}d $REMOTE_NAME:$REMOTE_DIR
fi
```

- **Automation:** Runs via cron at 04:30 daily (`crontab -e`).
    

---

## 4. Networking & Firewall

### 4.1 The "Two-Firewall" Reality

Traffic must pass two distinct layers to reach the application.

1. **Oracle Cloud VCN:** Ingress Rule TCP 25565 (Source Port: Blank, Stateless: Unchecked).
    
2. **Ubuntu IPTables:** `sudo iptables -I INPUT 1 -p tcp --dport 25565 -j ACCEPT`
    

### 4.2 Troubleshooting Connectivity

If connection fails:

1. Check internal listening: `sudo ss -tulpn | grep 25565`
    
2. Sniff traffic: `sudo tcpdump -i any port 25565`
    

### 4.3 MTU Optimization (The "Jumbo Frame" Fix)

Oracle Cloud defaults to MTU 9000. Public internet uses MTU 1500. This mismatch causes packet loss and lag.

- **The Fix:** Enforce 1500 via Netplan.
    
- **File:** `/etc/netplan/99-custom-mtu.yaml`
    

YAML

```
network:
  version: 2
  ethernets:
    enp0s6:
      mtu: 1500
      dhcp4: true
```

- **Apply:** `sudo netplan apply`
    

---

## 5. Migration Procedures (SOP)

### 5.1 Singleplayer -> Server Migration

|**Data Type**|**Symptom**|**Fix Method**|
|---|---|---|
|**Inventory**|Log in with empty pockets.|Extract `Player` tag from local `level.dat` using NBTExplorer. Save as `<UUID>.dat`. Upload to `world/playerdata/`.|
|**Quests**|Quest book is empty/reset.|Edit `QuestingData.json`. Use `sed` to replace Old UUID with New UUID globally.|
|**Waypoints**|JourneyMap is blank.|**Client-side fix.** Copy folder `journeymap/data/sp/<WorldName>` to `journeymap/data/mp/<ServerIP>`.|

### 5.2 World Folder Names

- **Linux is Case Sensitive.** `World` != `world`.
    
- **Fix:** Rename folder to lowercase `world` to match `server.properties`.
    

---

## 6. Disaster Recovery

### 6.1 Server Crash Loop

1. **Stop Service:** `sudo systemctl stop gtnh`
    
2. **Check Logs:** `cat /home/gtnh/crash-reports/crash-<latest>.txt`
    
3. **Clear Lock:** `rm /home/gtnh/world/session.lock` (Common cause after hard crash).
    
4. **Permissions:** `sudo chown -R gtnh:gtnh /home/gtnh/`
    

### 6.2 Total Infrastructure Loss (Oracle Deletes Instance)

1. Spin up new VPS (Any provider).
    
2. Install Java 25 & Rclone.
    
3. Authenticate Rclone (`rclone config`).
    
4. Pull Backup: `rclone copy gdrive:GTNH_Backups/latest.zip .`
    
5. Unzip and Run. **Data loss < 24 hours.**
    

---

## 7. Cost & Compliance (The "Free Tier" Rules)

To ensure the bill remains $0.00:

1. **Instance Count:** Maximum **1** VM active. Terminate all others.
    
2. **Public IP:** Must be attached to the running VM.
    
3. **Boot Volume:** **No Backup Policy** assigned.
    
4. **Volume Performance:** Set to **Balanced (10 VPU)**.
    
5. **Timezone:** Server set to `America/New_York` (`timedatectl`).
    

---

**Next Service Date:** Indefinite.

**Documentation Updated:** Jan 21, 2026 (v1.2 - Added Rclone Permissions).