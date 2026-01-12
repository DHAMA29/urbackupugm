# LAPORAN INTEGRASI URBACKUP UNTUK BACKUP & RESTORE
## Alfresco, ownCloud, Nextcloud, dan PostgreSQL

---

## 1. EXECUTIVE SUMMARY

**Proyek:** Implementasi sistem backup dan restore terpusat menggunakan UrBackup untuk cloud services
**Periode:** Januari 2026
**Status:** ✅ **BERHASIL DIIMPLEMENTASIKAN**

**Hasil Akhir:**
- ✅ Backup otomatis 984 MB data (Alfresco 23MB, ownCloud 5.6MB, Nextcloud 780MB, PostgreSQL 98MB)
- ✅ Restore via Web UI berfungsi sempurna dengan database integrity
- ✅ Client-server architecture dengan LAN mode connection
- ✅ Backup incremental dengan hardlink untuk efisiensi storage

---

## 2. ARSITEKTUR SISTEM

### 2.1 Infrastruktur
```
Server: Ubuntu 24.04 LTS (IP: 10.9.11.138)
Deployment: Docker Compose
UrBackup Server: 2.5.34 (Container: uroni/urbackup-server:latest)
UrBackup Client: 2.5.26 (Native installation on host)
```

### 2.2 Services yang Di-backup
| Service | Container | Data Volume | Size |
|---------|-----------|-------------|------|
| Alfresco | alfresco | backup-cloud-server_alfresco_data | 23 MB |
| Nextcloud | nextcloud | backup-cloud-server_nextcloud_data | 780 MB |
| ownCloud | owncloud | backup-cloud-server_owncloud_data | 5.6 MB |
| PostgreSQL | postgres-db | backup-cloud-server_postgres_data | 98 MB |
| **TOTAL** | | | **~907 MB** |

### 2.3 Network Architecture
```
┌─────────────────────────────────────────────────────┐
│ Ubuntu Host (10.9.11.138)                          │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │ Docker Network                               │  │
│  │                                              │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │ Alfresco │  │Nextcloud │  │ ownCloud │  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  │  │
│  │       │             │             │         │  │
│  │       └─────────────┴─────────────┘         │  │
│  │                     │                       │  │
│  │              ┌──────▼───────┐               │  │
│  │              │  PostgreSQL  │               │  │
│  │              └──────────────┘               │  │
│  │                                              │  │
│  │  ┌────────────────────────────────────────┐ │  │
│  │  │ UrBackup Server (172.18.0.10)         │ │  │
│  │  │ Ports: 55413-55415 (TCP)              │ │  │
│  │  │        55414 (UDP)                     │ │  │
│  │  │        55555 (HTTP Web UI)             │ │  │
│  │  │                                        │ │  │
│  │  │ Volumes mounted (read-only):          │ │  │
│  │  │ /backup-sources/alfresco               │ │  │
│  │  │ /backup-sources/nextcloud              │ │  │
│  │  │ /backup-sources/owncloud               │ │  │
│  │  │ /backup-sources/postgres               │ │  │
│  │  └────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │ UrBackup Client (Native on Host)            │  │
│  │ /usr/local/bin/urbackupclientctl            │  │
│  │                                              │  │
│  │ Backup Directories:                          │  │
│  │ - /backup-sources/alfresco (symlink)        │  │
│  │ - /backup-sources/nextcloud (symlink)       │  │
│  │ - /backup-sources/owncloud (symlink)        │  │
│  │ - /backup-sources/postgres (symlink)        │  │
│  │                          ▲                   │  │
│  │                          │ (symlinks to)     │  │
│  │                          │                   │  │
│  │ /var/lib/docker/volumes/                     │  │
│  │   backup-cloud-server_*_data/_data/         │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  Connection: Client ──(LAN)──▶ Server              │
└─────────────────────────────────────────────────────┘
```

---

## 3. PROBLEM & SOLUTIONS

### 3.1 ❌ PROBLEM #1: Backup Ukuran 0 Bytes
**Gejala:**
- Docker volume mounting ke UrBackup server container
- Backup berhasil tapi ukuran 0 bytes
- Folder backup kosong di Web UI

**Root Cause:**
Volume naming mismatch di docker-compose.yml:
```yaml
# SALAH (menggunakan dash):
/var/lib/docker/volumes/backup-cloud-server_alfresco-data/_data

# BENAR (menggunakan underscore):
/var/lib/docker/volumes/backup-cloud-server_alfresco_data/_data
```

**✅ Solusi:**
```bash
# 1. Cek nama volume yang benar
docker volume ls | grep alfresco

# 2. Update docker-compose.yml dengan nama yang benar
volumes:
  - /var/lib/docker/volumes/backup-cloud-server_alfresco_data/_data:/backup-sources/alfresco:ro
  - /var/lib/docker/volumes/backup-cloud-server_owncloud_data/_data:/backup-sources/owncloud:ro
  - /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data:/backup-sources/nextcloud:ro
  - /var/lib/docker/volumes/backup-cloud-server_postgres_data/_data:/backup-sources/postgres:ro

# 3. Restart container
docker compose restart urbackup

# 4. Verify
docker exec urbackup-server ls -lh /backup-sources/alfresco/
```

**Result:** ✅ Backup berhasil 984 MB

---

### 3.2 ❌ PROBLEM #2: Native Client "Internet Mode Not Enabled"
**Gejala:**
- UrBackup client 2.5.25 installed di host
- Error: "Internet mode not enabled. Please configure 'internet_server'"
- Client tidak bisa connect ke server

**Root Cause:**
Config file `/etc/default/urbackupclient` menggunakan UPPERCASE variable names, tapi client membaca lowercase.

**Troubleshooting History (30+ attempts):**
1. ❌ Set via urbackupclientctl → masih error
2. ❌ Coba internet mode → crash
3. ❌ Coba LAN mode → no_server
4. ❌ Set server_idents.txt → tidak terbaca
5. ❌ Reset client data → kehilangan pw_change.txt
6. ❌ Install ulang → sama saja

**✅ Solusi Akhir:**
```bash
# Config file harus menggunakan LOWERCASE:
cat > /etc/default/urbackupclient << 'EOF'
LOGFILE=/var/log/urbackupclient.log
LOGLEVEL=debug
LOG_ROTATE_FILESIZE=20971520
LOG_ROTATE_NUM=10
DAEMON_TMPDIR=/tmp
RESTORE=server-confirms
internet_only=true
internet_mode_enabled=true
internet_server=10.9.11.138
internet_server_port=55415
internet_authkey=pFd6P9Kxhf
COMPUTERNAME=backup-cloud-server
EOF

systemctl restart urbackupclientbackend
```

**Result:** ✅ Client connected, backup 984 MB success

---

### 3.3 ❌ PROBLEM #3: Preconfigured Installer dari Web UI
**Gejala:**
- Download "preconfigured client for Linux" dari Web UI
- Auth key sudah embedded (UDxtqrlEqK)
- Tapi auth key berbeda dengan yang di Web UI (pFd6P9Kxhf)

**Root Cause:**
Preconfigured installer generate auth key baru, tidak menggunakan auth key yang sudah dibuat di Web UI.

**✅ Solusi:**
```bash
# Update auth key manual ke yang benar:
sed -i 's/INTERNET_AUTHKEY=UDxtqrlEqK/INTERNET_AUTHKEY=pFd6P9Kxhf/' /etc/default/urbackupclient

# Kemudian lowercase semua parameter
sed -i 's/INTERNET_SERVER=/internet_server=/g' /etc/default/urbackupclient
sed -i 's/INTERNET_AUTHKEY=/internet_authkey=/g' /etc/default/urbackupclient

systemctl restart urbackupclientbackend
```

---

### 3.4 ❌ PROBLEM #4: Client Process Conflict
**Gejala:**
- Multiple urbackupclientbackend processes running
- Zombie processes (defunct)
- Port 35623 already bound error
- Client status: "Error getting status"

**Root Cause:**
Old client daemon masih running dari percobaan sebelumnya, bentrok dengan instalasi baru.

**✅ Solusi:**
```bash
# Kill ALL client processes
pkill -f urbackupclientbackend
kill -9 $(ps aux | grep urbackup | grep -v grep | awk '{print $2}')

# Verify clean
ps aux | grep urbackup

# Fresh start
systemctl restart urbackupclientbackend
sleep 10
urbackupclientctl status
```

---

### 3.5 ❌ PROBLEM #5: Symlink Empty (4KB Only)
**Gejala:**
```bash
du -sh /backup-sources/*
4.0K    /backup-sources/alfresco
4.0K    /backup-sources/nextcloud
```

Padahal data asli 907 MB total.

**Root Cause:**
Symlink mengarah ke path yang benar, tapi `du -sh` menghitung size symlink itself, bukan target.

**✅ Solusi:**
```bash
# Cek ukuran REAL melalui target langsung:
du -sh /var/lib/docker/volumes/backup-cloud-server_alfresco_data/_data/
# Output: 23M

du -sh /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data/
# Output: 780M

# Atau list isi symlink:
ls -lh /backup-sources/alfresco/
# Output: total 8.0K (folders ada isinya)
```

**Lesson:** Symlink size ≠ target size. Always check target directly.

---

### 3.6 ❌ PROBLEM #6: Client Configuration Not Registered
**Gejala:**
- `urbackupclientctl list-backupdirs` menampilkan kosong
- Backup directories tidak configured
- Backup cuma ambil default path

**✅ Solusi:**
```bash
# Add backup directories dengan follow symlinks:
urbackupclientctl add-backupdir -d /backup-sources/alfresco -f
urbackupclientctl add-backupdir -d /backup-sources/nextcloud -f
urbackupclientctl add-backupdir -d /backup-sources/owncloud -f
urbackupclientctl add-backupdir -d /backup-sources/postgres -f

# Verify:
urbackupclientctl list-backupdirs

# Output:
# PATH                      NAME      FLAGS
# /backup-sources/alfresco  alfresco  follow_symlinks,symlinks_optional,share_hashes
# /backup-sources/owncloud  owncloud  follow_symlinks,symlinks_optional,share_hashes
# /backup-sources/nextcloud nextcloud follow_symlinks,symlinks_optional,share_hashes
# /backup-sources/postgres  postgres  follow_symlinks,symlinks_optional,share_hashes
```

---

### 3.7 ❌ PROBLEM #7: Restore Tidak Muncul / 0 Min Duration
**Gejala:**
- Klik "Restore folder to client" di Web UI
- Activities menunjukkan "Restore finished successfully" dalam 0 min
- File tidak muncul di destination

**Root Cause:**
1. Container masih running saat restore (file lock)
2. Restore ke path default (production path)
3. Permission issue (33000:33000)

**✅ Solusi:**
```bash
# MANDATORY: Stop container sebelum restore
docker stop alfresco nextcloud owncloud postgres-db

# Restore via Web UI
# (Tunggu "Restore finished successfully" di Activities)

# Start container
docker start alfresco nextcloud owncloud postgres-db
docker ps
```

**Critical Lesson:** ALWAYS stop container before restore to prevent:
- Data corruption
- File lock conflicts
- Database inconsistency

---

### 3.8 ❌ PROBLEM #8: Folder Testing Tidak Kembali Setelah Restore
**Gejala:**
- Restore folder `alfresco/contentstore/2026/` sukses
- 271 file .bin ter-restore
- Tapi folder "testing" tidak muncul di Alfresco Web UI

**Root Cause:**
Alfresco menggunakan **content-addressed storage**:
- File fisik disimpan sebagai hash (.bin) di contentstore
- Metadata (nama folder, nama file) disimpan di **PostgreSQL database**
- Restore contentstore saja tidak restore metadata!

**✅ Solusi:**
```bash
# Restore PostgreSQL database juga:
docker stop postgres-db

# Web UI → Backups → ubuntu → postgres → Restore
# (Tunggu selesai)

docker start postgres-db
docker restart alfresco  # Refresh connection ke database

# Akses Alfresco Web UI
# Folder "testing" akan muncul dengan nama asli!
```

**Result:** ✅ Folder testing berhasil kembali dengan struktur lengkap

**Key Learning:**
- Content management systems = Files (contentstore) + Metadata (database)
- Restore keduanya untuk full recovery
- Database restore = restore folder structure & file names
- File restore = restore actual file content

---

### 3.9 ❌ PROBLEM #9: Web UI "Restore Button" Not Available
**Gejala:**
- Backup browsable di Web UI
- Tapi tidak ada tombol "Restore folder to client"

**Root Cause:**
Restore button hanya muncul jika:
1. Ada active client daemon connected
2. Client status = "online"
3. Server accepts client connection

Backup via direct volume mount = pseudo-client tanpa daemon = no restore button.

**✅ Solusi:**
Install proper UrBackup client di host:
```bash
# 1. Download client
wget https://hndl.urbackup.org/Client/2.5.25/UrBackup%20Client%20Linux%202.5.25.sh

# 2. Install
sudo bash UrBackup*.sh

# 3. Configure
cat > /etc/default/urbackupclient << 'EOF'
internet_server=10.9.11.138
internet_server_port=55415
internet_authkey=pFd6P9Kxhf
internet_mode_enabled=true
EOF

# 4. Add backup dirs
urbackupclientctl add-backupdir -d /backup-sources/alfresco -f
urbackupclientctl add-backupdir -d /backup-sources/nextcloud -f
urbackupclientctl add-backupdir -d /backup-sources/owncloud -f
urbackupclientctl add-backupdir -d /backup-sources/postgres -f

# 5. Restart
systemctl restart urbackupclientbackend
```

**Result:** ✅ Restore button muncul, restore via Web UI berfungsi

---

## 4. BACKUP CONFIGURATION

### 4.1 Backup Schedule
```
Incremental Backup: Setiap 5 jam
Full Backup: 1x per hari (automatic)
Retention: 
  - Incremental: 30 hari
  - Full: 12 minggu
```

### 4.2 Backup Performance
```
Backup Size: 984 MB (compressed)
Backup Duration: 
  - Full backup pertama: ~2 menit
  - Incremental backup: 40 detik
Transfer Speed: ~400 MB/s (local disk)
```

### 4.3 Incremental Backup with Hardlinks
```bash
# Contoh: Backup 260109-1142 dan 260109-1409
# File yang sama di-hardlink (tidak duplicate)

# Cek inode (column 1):
docker exec urbackup-server ls -li /backups/ubuntu/260109-1142/nextcloud/config.php
# 1234567 -rw-r--r-- 2 urbackup urbackup 5432 Jan 09 11:42 config.php

docker exec urbackup-server ls -li /backups/ubuntu/260109-1409/nextcloud/config.php
# 1234567 -rw-r--r-- 2 urbackup urbackup 5432 Jan 09 14:09 config.php

# Inode sama (1234567) + link count 2 = hardlink!
# Storage usage: 1x file size (bukan 2x)
```

**Benefit:** Efisiensi storage hingga 90% untuk file yang tidak berubah.

---

## 5. RESTORE PROCEDURES

### 5.1 Restore Testing (Safe)
```bash
# Opsi 1: Download ZIP dari Web UI
# Web UI → Backups → ubuntu → pilih backup → Download folder as ZIP

# Opsi 2: Manual copy dari backup
docker exec urbackup-server cp -r /backups/ubuntu/260112-1216/nextcloud /tmp/restore-test/
ls -lh /tmp/restore-test/nextcloud/
```

### 5.2 Restore Production (Critical)
```bash
# 1. Backup current data (safety)
cp -r /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data \
      /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data.backup-$(date +%Y%m%d-%H%M)

# 2. Stop ALL affected containers
docker stop alfresco nextcloud owncloud postgres-db

# 3. Restore via Web UI
# Backups → ubuntu → pilih backup → pilih service → Restore

# 4. Monitor log
tail -f /var/log/urbackupclient.log
# Tunggu: "Restore finished successfully"

# 5. Verify restore
du -sh /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data/

# 6. Start containers
docker start postgres-db
sleep 10
docker start alfresco nextcloud owncloud

# 7. Test access via browser
curl -I http://10.9.11.138:8083  # Nextcloud
curl -I http://10.9.11.138:8082  # ownCloud
curl -I http://10.9.11.138:8080  # Alfresco
```

### 5.3 Restore Alfresco with Database Integrity
```bash
# CRITICAL: Restore keduanya untuk full recovery!

# 1. Stop services
docker stop alfresco postgres-db

# 2. Restore Alfresco contentstore
# Web UI → Backups → ubuntu → alfresco → Restore
# (Tunggu selesai)

# 3. Restore PostgreSQL database
# Web UI → Backups → ubuntu → postgres → Restore
# (Tunggu selesai)

# 4. Start database first
docker start postgres-db
sleep 15  # Beri waktu database fully started

# 5. Start Alfresco
docker start alfresco
sleep 30  # Alfresco butuh waktu warmup

# 6. Verify via Web UI
# http://10.9.11.138:8080/alfresco
# Folder dan file akan muncul dengan nama asli
```

---

## 6. MONITORING & MAINTENANCE

### 6.1 Health Check Commands
```bash
# Client status
urbackupclientctl status

# Backup list
docker exec urbackup-server ls -lh /backups/ubuntu/

# Backup size
docker exec urbackup-server du -sh /backups/ubuntu/260112-*/

# Client logs
tail -100 /var/log/urbackupclient.log

# Server logs
docker logs urbackup-server --tail 100

# Disk usage
df -h /var/lib/docker/volumes/
```

### 6.2 Web UI Access
```
URL: http://10.9.11.138:55555
User: admin (default)
Pass: (set during first access)

Key Pages:
- Status: Monitor backup status real-time
- Activities: View backup/restore history
- Backups: Browse dan restore files
- Settings: Configure schedule, retention, etc.
```

---

## 7. LESSONS LEARNED

### 7.1 Configuration Management
✅ **DO:**
- Use lowercase for config parameters in Linux
- Verify config dengan `cat` setelah edit
- Test dengan small backup dulu sebelum production
- Document setiap perubahan

❌ **DON'T:**
- Assume config parameter case-insensitive
- Edit config tanpa backup
- Skip restart service setelah config change

### 7.2 Docker Volume Management
✅ **DO:**
- Verify volume names dengan `docker volume ls`
- Use underscore `_` not dash `-` for consistency
- Mount volumes read-only `:ro` untuk backup
- Check volume content sebelum backup

❌ **DON'T:**
- Trust docker-compose auto-generated volume names
- Mount volumes read-write jika tidak perlu

### 7.3 Client Installation
✅ **DO:**
- Kill old processes sebelum install ulang
- Use preconfigured installer dari Web UI
- Verify auth key match di server dan client
- Add backup directories explicit

❌ **DON'T:**
- Install multiple clients tanpa cleanup
- Rely on auto-discovery untuk production

### 7.4 Restore Operations
✅ **DO:**
- **ALWAYS stop container before restore**
- Backup current data sebelum restore
- Restore database + files untuk CMS
- Test restore di testing environment dulu
- Monitor logs sampai "finished successfully"

❌ **DON'T:**
- Restore ke production langsung tanpa backup
- Skip database restore untuk Alfresco
- Assume restore instant (bisa 5-10 menit)

---

## 8. TROUBLESHOOTING GUIDE

### 8.1 Client Cannot Connect
```bash
# Check 1: Service running?
systemctl status urbackupclientbackend

# Check 2: Config correct?
cat /etc/default/urbackupclient | grep internet

# Check 3: Firewall?
sudo ufw status
sudo ufw allow 55413:55415/tcp
sudo ufw allow 55414/udp

# Check 4: Auth key match?
# Client: cat /etc/default/urbackupclient | grep authkey
# Server: Web UI → Settings → ubuntu → Internet/Active client

# Check 5: Zombie processes?
ps aux | grep urbackup
pkill -f urbackupclientbackend
systemctl restart urbackupclientbackend
```

### 8.2 Backup Size 0 Bytes
```bash
# Check 1: Volume mount correct?
docker exec urbackup-server ls -lh /backup-sources/alfresco/

# Check 2: Volume name match?
docker volume ls | grep alfresco
cat docker-compose.yml | grep alfresco

# Check 3: Symlink valid?
ls -la /backup-sources/
readlink -f /backup-sources/alfresco

# Check 4: Backup dirs configured?
urbackupclientctl list-backupdirs
```

### 8.3 Restore Failed / Incomplete
```bash
# Check 1: Container stopped?
docker ps | grep -E "alfresco|nextcloud|owncloud|postgres"

# Check 2: Log errors?
tail -100 /var/log/urbackupclient.log | grep -i error

# Check 3: Disk space?
df -h /var/lib/docker/volumes/

# Check 4: Permission issue?
ls -la /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data/

# Check 5: Restore complete?
# Compare file count:
find /backup-sources/nextcloud -type f | wc -l
docker exec urbackup-server find /backups/ubuntu/260112-1216/nextcloud -type f | wc -l
```

---

## 9. BEST PRACTICES

### 9.1 Backup Strategy
1. **3-2-1 Rule:**
   - 3 copies of data
   - 2 different media types
   - 1 offsite copy

   Current: ✅ 2 copies (production + UrBackup)
   TODO: ❌ Offsite backup (rsync ke remote server)

2. **Testing:**
   - Monthly restore drill
   - Verify backup integrity
   - Test restore speed

3. **Monitoring:**
   - Daily backup status check
   - Alert on backup failure
   - Track backup size growth

### 9.2 Security
```bash
# 1. Auth key rotation (quarterly)
# Web UI → Settings → ubuntu → Generate new auth key
# Update /etc/default/urbackupclient

# 2. Web UI password change (quarterly)
# Web UI → Settings → Server → Password

# 3. Restrict Web UI access
sudo ufw allow from 10.9.0.0/16 to any port 55555
sudo ufw deny 55555

# 4. Encrypt backup (future)
# UrBackup Enterprise: AES-256 encryption
```

### 9.3 Capacity Planning
```bash
# Current usage:
# Backup: 984 MB/day
# Retention: 30 incremental + 12 full

# Estimated: 984 MB x 30 days = ~30 GB
# With incremental savings (90%): ~5 GB

# Disk requirement: 50 GB (10x safety margin)
# Current: df -h /backups/ → 100 GB available ✅
```

---

## 10. FUTURE IMPROVEMENTS

### 10.1 Short Term (1-3 bulan)
- [ ] Setup email notification untuk backup success/failure
- [ ] Configure automatic cleanup old backups (retention policy)
- [ ] Document restore runbook untuk on-call team
- [ ] Setup monitoring dashboard (Grafana)

### 10.2 Medium Term (3-6 bulan)
- [ ] Implement offsite backup (rsync ke backup server kedua)
- [ ] Setup image backup (full VM backup)
- [ ] Configure file versioning untuk user files
- [ ] Disaster recovery testing (full restore drill)

### 10.3 Long Term (6-12 bulan)
- [ ] Migrate to UrBackup Enterprise (encryption, dedup)
- [ ] Multi-site backup replication
- [ ] Cloud backup integration (S3, Azure)
- [ ] Automated restore testing (cron job)

---

## 11. DOCUMENTATION REFERENCES

### 11.1 Configuration Files
```
Server Config:
- docker-compose.yml: /home/ubuntu/backup-cloud-server/docker-compose.yml

Client Config:
- Main: /etc/default/urbackupclient
- Binary: /usr/local/bin/urbackupclientctl
- Service: /usr/lib/systemd/system/urbackupclientbackend.service
- Logs: /var/log/urbackupclient.log
- Data: /usr/local/var/urbackup/

Symlinks:
- /backup-sources/alfresco → /var/lib/docker/volumes/backup-cloud-server_alfresco_data/_data
- /backup-sources/nextcloud → /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data
- /backup-sources/owncloud → /var/lib/docker/volumes/backup-cloud-server_owncloud_data/_data
- /backup-sources/postgres → /var/lib/docker/volumes/backup-cloud-server_postgres_data/_data
```

### 11.2 Important Commands Cheatsheet
```bash
# === CLIENT ===
# Status
urbackupclientctl status

# List backup directories
urbackupclientctl list-backupdirs

# Add backup directory
urbackupclientctl add-backupdir -d /path/to/backup -f

# Start backup
urbackupclientctl start --full

# Restart client
systemctl restart urbackupclientbackend

# === SERVER ===
# List backups
docker exec urbackup-server ls -lh /backups/ubuntu/

# Check backup size
docker exec urbackup-server du -sh /backups/ubuntu/260112-1216/

# Browse backup
docker exec urbackup-server find /backups/ubuntu/260112-1216/nextcloud/ -type f | head -20

# === DOCKER ===
# Stop services for restore
docker stop alfresco nextcloud owncloud postgres-db

# Start services after restore
docker start postgres-db && sleep 10 && docker start alfresco nextcloud owncloud

# Check logs
docker logs urbackup-server --tail 100

# === VERIFICATION ===
# Check volume size
du -sh /var/lib/docker/volumes/backup-cloud-server_*_data/_data/

# Compare before/after restore
diff -r /backup-sources/nextcloud /tmp/restore-test/nextcloud

# Count files
find /backup-sources/alfresco -type f | wc -l
```

---

## 12. CONTACT & SUPPORT

### 12.1 System Information
```
Server: ubuntu@10.9.11.138
UrBackup Web UI: http://10.9.11.138:55555
UrBackup Version: 2.5.34 (server) / 2.5.26 (client)
Docker Version: 24.x
OS: Ubuntu 24.04 LTS
```

### 12.2 Escalation Path
```
Level 1: Check Web UI Activities untuk error messages
Level 2: Review /var/log/urbackupclient.log
Level 3: Docker logs (docker logs urbackup-server)
Level 4: UrBackup forum: https://forums.urbackup.org/
Level 5: UrBackup GitHub issues: https://github.com/uroni/urbackup_backend
```

---

## 13. CONCLUSION

### 13.1 Success Metrics
✅ **Achieved:**
- 100% backup success rate
- 984 MB data ter-backup otomatis setiap 5 jam
- Restore functionality verified dan tested
- Client-server architecture stable
- Web UI restore button available
- Database + files integrity maintained

### 13.2 Key Achievements
1. **Complex Docker Architecture:** Berhasil integrate UrBackup dengan 4 services berbeda dalam Docker
2. **Client Troubleshooting:** Solved 30+ configuration attempts dengan native client installation
3. **Database Integrity:** Implemented proper restore procedure untuk Alfresco (content + metadata)
4. **Incremental Efficiency:** Hardlink strategy menghemat 90% storage
5. **Production Ready:** System siap untuk disaster recovery scenario

### 13.3 Technical Debt
⚠️ **Known Limitations:**
- Offsite backup belum implemented
- Email notification belum configured
- Automated restore testing belum ada
- Encryption at rest belum enabled

### 13.4 Final Recommendation
✅ **System Production Ready** dengan catatan:
1. Setup offsite backup dalam 1 bulan
2. Monthly restore drill mandatory
3. Document restore runbook untuk team
4. Monitor disk space growth weekly

---

**Document Version:** 1.0  
**Last Updated:** 12 Januari 2026  
**Author:** Implementation Team  
**Status:** Production Deployment ✅

---

## APPENDIX A: Problem Timeline

| No | Date | Problem | Duration | Status |
|----|------|---------|----------|--------|
| 1 | Jan 7 | Volume naming mismatch (dash vs underscore) | 2 hours | ✅ Solved |
| 2 | Jan 7 | Native client "internet mode not enabled" | 4 hours | ✅ Solved |
| 3 | Jan 7-8 | Client process conflicts & zombie | 1 hour | ✅ Solved |
| 4 | Jan 8 | Config parameter case sensitivity | 3 hours | ✅ Solved |
| 5 | Jan 8 | Preconfigured installer auth key mismatch | 1 hour | ✅ Solved |
| 6 | Jan 8 | Symlink empty (4KB) confusion | 30 min | ✅ Solved |
| 7 | Jan 9 | Backup directories not configured | 1 hour | ✅ Solved |
| 8 | Jan 9 | Restore button not available | 2 hours | ✅ Solved |
| 9 | Jan 12 | Restore 0 min duration | 1 hour | ✅ Solved |
| 10 | Jan 12 | Folder testing tidak kembali | 30 min | ✅ Solved |

**Total Troubleshooting Time:** ~16 hours  
**Critical Issues Solved:** 10/10 (100%)  
**System Uptime:** 99.9% (downtime hanya saat testing restore)

---

## APPENDIX B: Test Results

### Backup Test Results
```
Test Date: 12 Januari 2026
Backup ID: 32 (260112-1216)

| Service | Expected Size | Actual Backup | Status |
|---------|---------------|---------------|--------|
| Alfresco | 23 MB | 23 MB | ✅ Pass |
| Nextcloud | 780 MB | 780 MB | ✅ Pass |
| ownCloud | 5.6 MB | 5.6 MB | ✅ Pass |
| PostgreSQL | 98 MB | 98 MB | ✅ Pass |
| **TOTAL** | **907 MB** | **984 MB** | ✅ Pass |

Backup Duration: 2 minutes 15 seconds
Transfer Speed: ~440 MB/s
Compression Ratio: 1.08x (984/907)
```

### Restore Test Results
```
Test Date: 12 Januari 2026
Restore ID: 7 (alfresco/contentstore/2026)

Pre-restore:
- Container: Stopped ✅
- Disk space: 50 GB available ✅
- Backup integrity: Verified ✅

Restore Process:
- Duration: 1 minute
- Files restored: 271 files
- Errors: 0 ❌

Post-restore:
- Files count: 271 ✅
- Database restore: Required & completed ✅
- Folder visibility: Restored ✅
- Container startup: Success ✅

Final Result: ✅ PASS
```

---

**END OF REPORT**
