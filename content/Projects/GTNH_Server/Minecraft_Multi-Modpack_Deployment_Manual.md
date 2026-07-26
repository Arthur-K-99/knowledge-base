# OCI Minecraft Multi-Modpack Deployment Manual

**Version:** 2.2

**Last verified:** July 26, 2026

**Host:** Oracle Cloud Infrastructure Ampere A1 Free Tier

**Server IP:** `129.213.126.1`

**OS:** Ubuntu 24.04 LTS (ARM64)

## 1. Current State

This VM hosts multiple modpacks, but only one can run through the shared
`minecraft.service`.

| Pack | Version | State | Directory | Java | Heap |
| --- | --- | --- | --- | --- | --- |
| Nomifactory CEu | 1.7.7, Normal mode | **Active** | `/srv/minecraft/packs/nomiceu` | OpenJDK 8 ARM64 | 8–16 GB |
| GregTech: New Horizons | 2.8.4 | Paused | `/home/gtnh` | Temurin 25 ARM64 | 16–20 GB |

The public game address remains `129.213.126.1:25565`. The client pack and
version must match the active server.

Nomifactory CEu 1.7.7 was installed from the
[official release](https://github.com/Nomi-CEu/Nomi-CEu/releases/tag/1.7.7).
The project requires Java 8 for its standard Forge server, as documented in the
[official server setup guide](https://github.com/Nomi-CEu/Nomi-CEu/wiki/Page-1%3A-Player-Information#section-3-server-installation-and-updating).

## 2. SSH Access

From the Windows computer that holds the private key:

```powershell
ssh -i C:\Users\athanasios\main ubuntu@129.213.126.1
```

The two Linux identities remain:

- `ubuntu`: administrative account with `sudo`.
- `gtnh`: unprivileged service account used by both Minecraft packs.

The service account name is historical; it does not mean GTNH is active.

## 3. One-Active-Pack Architecture

```text
/srv/minecraft/
├── active -> /srv/minecraft/packs/nomiceu
├── active-pack
└── packs/
    ├── gtnh -> /home/gtnh
    └── nomiceu/
```

The shared systemd unit is `/etc/systemd/system/minecraft.service`. It always
starts `/srv/minecraft/active/start-server.sh`.

The old `gtnh.service` is disabled and has a systemd condition that blocks
accidental direct starts. GTNH must be started through `mc-switch`.

### 3.1 Status

```bash
cat /srv/minecraft/active-pack
systemctl status minecraft
journalctl -u minecraft -f
```

Only one Java server process should exist:

```bash
pgrep -a -u gtnh java
```

### 3.2 Switch Packs

Switch to GTNH:

```bash
sudo mc-switch gtnh
```

Switch to Nomifactory CEu:

```bash
sudo mc-switch nomiceu
```

`mc-switch` performs these operations in order:

1. Gracefully stops `minecraft.service`.
2. Creates an offline ZIP of the current pack's world.
3. Tests the complete ZIP and prints its SHA-256 checksum.
4. Changes `/srv/minecraft/active`.
5. Starts `minecraft.service`.

The command returns after the new Java process starts. Follow the journal until
the selected pack prints its ready message.

### 3.3 Pause Without Switching

To pause the active server:

```bash
sudo systemctl stop minecraft
sudo mc-backup nomiceu
```

Use `gtnh` instead of `nomiceu` when GTNH is active. `mc-backup` refuses to run
while a Minecraft Java process exists.

Resume the same pack:

```bash
sudo systemctl start minecraft
```

## 4. Shared Systemd Unit

`/etc/systemd/system/minecraft.service`:

```ini
[Unit]
Description=Minecraft Server (active modpack)
Wants=network-online.target
After=network-online.target
ConditionPathIsSymbolicLink=/srv/minecraft/active

[Service]
Type=simple
User=gtnh
Group=gtnh
WorkingDirectory=/srv/minecraft/active
ExecStart=/srv/minecraft/active/start-server.sh
Restart=on-failure
RestartSec=15
TimeoutStopSec=180
KillSignal=SIGTERM
SuccessExitStatus=0 143
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Useful service commands:

```bash
sudo systemctl start minecraft
sudo systemctl stop minecraft
sudo systemctl restart minecraft
sudo systemctl enable minecraft
```

Do not use `gtnh.service`.

### 4.1 Secure Java Server Console

After connecting over SSH, open the active pack's interactive server console:

```bash
sudo mc-console
```

Enter normal Minecraft server commands without a leading slash, for example:

```text
list
say Server maintenance begins in 10 minutes
save-all
whitelist add PlayerName
```

Press `Ctrl+D` or `Ctrl+C` to leave the console. A single command can also be
sent without opening an interactive session:

```bash
sudo mc-console list
```

`mc-console` reads `/srv/minecraft/active-pack` and automatically uses the
matching pack's root-only RCON credential. The credentials are stored under
`/etc/minecraft/rcon/`; do not copy or display them. For read-only live output,
use `sudo journalctl -u minecraft -f`.

## 5. Pack Runtimes

### 5.1 GTNH

GTNH starts through `/home/gtnh/start-server.sh` and uses:

```text
/opt/java25/bin/java
-Xms16G -Xmx20G
ZGC
CompactObjectHeaders
lwjgl3ify-forgePatches.jar
```

The GTNH world directory is `/home/gtnh/World` with a capital `W`.

### 5.2 Nomifactory CEu

Nomifactory starts through
`/srv/minecraft/packs/nomiceu/start-server.sh` and uses:

```text
/usr/lib/jvm/java-8-openjdk-arm64/jre/bin/java
-Xms8G -Xmx16G
G1GC
forge-1.12.2-14.23.5.2860.jar
```

Important Nomifactory settings:

| Setting | Value |
| --- | --- |
| Version | 1.7.7 |
| Mode | Normal |
| World generator | Lost Cities |
| World directory | `/srv/minecraft/packs/nomiceu/world` |
| Port | 25565 |
| Whitelist | Enabled |
| RCON | Enabled for `mc-console`; blocked from non-loopback interfaces |
| View distance | 8 |

GTNH's operator, whitelist, and ban JSON files were copied into Nomifactory.

## 6. Backups

### 6.1 Authoritative Switch Backups

Every `mc-switch` creates and tests an offline world ZIP:

```text
<pack-directory>/switch-backups/<pack>-switch-YYYY-MM-DD-HH-MM-SS.zip
```

These are the preferred restore points because the Java process is stopped
before the archive is created.

Latest verified Nomifactory switch backup:

```text
File: /srv/minecraft/packs/nomiceu/switch-backups/nomiceu-switch-2026-07-26-15-29-39.zip
Size: 3,322,524 bytes
SHA-256: 577c9a12a8fef9a4361efd5c1c2918965c7cded05ce9927487bc4911994e7447
```

### 6.2 GTNH Cold Archive

GTNH was gracefully stopped on July 26, 2026. The shutdown log confirmed:

- Stopping server
- Saving players
- Saving worlds
- Saving chunks for `World`

A full local cold archive was then created. It contains the world, server
files, mods, configs, and the original systemd unit. Logs, earlier backups, and
rclone credentials were deliberately excluded.

```text
File: /home/gtnh/cold-archives/GTNH-cold-2026-07-26-14-14-10.tar.gz
Size: 1,807,595,047 bytes
SHA-256: b221d80eb25d70e3dbbe500ad27b85ab6b35aef5f3d18fc78de9744b682a79cb
```

The gzip stream and required restore paths were tested successfully.

This new full cold archive is local only. Uploading it to Google Drive requires
explicit approval because it includes the full server snapshot.

### 6.3 Existing GTNH World Backup

The latest pre-pause GTNH world ZIP was fully decompressed and tested:

```text
File: /home/gtnh/backups/2026-07-22-20-32-39.zip
Size: 1,332,435,833 bytes
SHA-256: 90dcf43cb5da7fef2aac9a1e709909b4fa720b00081bc6d28c5cec50ae94ed6d
```

Google Drive contains a copy with the same filename, byte size, and MD5 at
`gdrive:Backups/GTNH`.

The existing GTNH cron job remains:

```cron
30 04 * * * /home/gtnh/push-backups.sh >> /home/gtnh/backup_log.txt 2>&1
```

It uploads existing local GTNH backups and prunes remote backups older than 14
days while preserving the newest remote file.

### 6.4 Nomifactory Local and Google Drive Backups

FTB Backups is enabled in
`/srv/minecraft/packs/nomiceu/config/ftbbackups.cfg`:

| Setting | Value |
| --- | --- |
| Interval | 30 minutes |
| Backups retained | 10 |
| Maximum total size | 20 GB |
| Only with players online | No |
| Directory | `/srv/minecraft/packs/nomiceu/backups` |

Do not rely on FTB Backups alone when changing packs. The offline switch backup
is the authoritative safety step.

Nomifactory uploads both backup types to separate Google Drive subdirectories:

```text
gdrive:Backups/
├── GTNH/
└── Nomifactory_CEu/
    ├── scheduled/
    └── switch/
```

The upload script is
`/srv/minecraft/packs/nomiceu/push-backups.sh`. It copies and verifies each
backup category independently, deletes remote files older than 14 days only
after successful verification, and always preserves the newest remote file in
each category.

The `gtnh` service account runs it daily at 05:00:

```cron
00 05 * * * /srv/minecraft/packs/nomiceu/push-backups.sh >> /srv/minecraft/packs/nomiceu/backup-upload.log 2>&1
```

The first off-site verification on July 26, 2026 confirmed:

```text
Remote: gdrive:Backups/Nomifactory_CEu/switch/nomiceu-switch-2026-07-26-14-47-33.zip
Size: 3,060,849 bytes
Local MD5:  0bce9941e78846529839162056e7aeab
Remote MD5: 0bce9941e78846529839162056e7aeab
```

The scheduled-backup remote directory was empty during initial setup because
the first 30-minute FTB backup had not yet been created. The daily upload will
copy it after it appears.

## 7. Restore Procedures

### 7.1 Restore a Nomifactory Switch Backup

```bash
sudo systemctl stop minecraft
cd /srv/minecraft/packs/nomiceu
sudo mv world world.pre-restore
sudo -u gtnh unzip switch-backups/nomiceu-switch-YYYY-MM-DD-HH-MM-SS.zip
sudo chown -R gtnh:gtnh world
sudo systemctl start minecraft
```

Verify the ready banner:

```bash
grep -F "Players Can Now Join" /srv/minecraft/packs/nomiceu/logs/latest.log
```

Delete `world.pre-restore` only after players verify the restored world.

### 7.2 Restore the GTNH World ZIP

```bash
sudo systemctl stop minecraft
cd /home/gtnh
sudo mv World World.pre-restore
sudo -u gtnh unzip backups/2026-07-22-20-32-39.zip
sudo chown -R gtnh:gtnh World
sudo mc-switch gtnh
```

If GTNH is already the active pack, start it with
`sudo systemctl start minecraft` after the restore instead of running another
switch backup.

### 7.3 Inspect the Full GTNH Cold Archive

```bash
sudo -u gtnh gzip -t /home/gtnh/cold-archives/GTNH-cold-2026-07-26-14-14-10.tar.gz
sudo -u gtnh tar -tzf /home/gtnh/cold-archives/GTNH-cold-2026-07-26-14-14-10.tar.gz | less
```

The archive stores paths relative to `/`, including `home/gtnh/` and the
legacy systemd unit. Restore selected paths into a staging directory first;
do not extract it blindly over a running server.

## 8. Networking

The OCI VCN and Ubuntu firewall must allow TCP 25565. Both packs use that same
port, which is another guard against simultaneous operation.

Both packs have RCON configured on TCP 25575 for `mc-console`. The
`minecraft-rcon-firewall.service` installs explicit IPv4 and IPv6 rules that
allow this port only over the loopback interface and drop every non-loopback
connection. Do not expose TCP 25575 through OCI or remove these host firewall
rules. The Minecraft game port remains publicly reachable on TCP 25565.

The host Netplan MTU remains 1500:

```yaml
network:
  version: 2
  ethernets:
    enp0s6:
      mtu: 1500
      dhcp4: true
```

## 9. Free-Tier and Capacity Notes

- One OCI VM is used; no second instance was created.
- The boot volume is approximately 100 GB.
- At deployment verification, approximately 74 GB was free before adding
  Nomifactory and the new cold archive.
- Only one Minecraft Java process should run.
- Nomifactory's 16 GB maximum heap leaves more OS headroom than
  GTNH's 20 GB maximum heap.
- Keep pack backups in separate directories and periodically review disk use:

```bash
df -h /
du -sh /home/gtnh /srv/minecraft/packs/nomiceu
```

## 10. Deployment Verification Record

On July 26, 2026:

- `minecraft.service` was enabled and active.
- Active pack marker: `nomiceu`.
- Nomifactory reported version 1.7.7, Normal mode, port 25565, and
  `Players Can Now Join!`.
- Nomifactory's Java heap was configured at 8 GB minimum and 16 GB maximum.
- Exactly one Java process was running under `gtnh`.
- GTNH was inactive, disabled, and protected from accidental legacy-unit
  starts.
- Port 25565 was listening on the Nomifactory Java process.
- `sudo mc-console list` returned successfully using the active pack's
  root-only credential.
- TCP 25575 was blocked from the public Internet by explicit IPv4 and IPv6
  firewall rules.
- A Nomifactory offline switch backup passed a full ZIP integrity test.
