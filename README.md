# LAPORAN INTEGRASI URBACKUP
## Sistem Backup & Restore untuk Cloud Services

---

**Server:** Ubuntu 24.04 LTS 
**Tanggal:** Januari 2026  
**Status:** ✅ **BERHASIL DIIMPLEMENTASIKAN**

---

## 1. TOOLS YANG DIGUNAKAN

Tools yang digunakan dalam pengembangan dan dokumentasi proyek ini:

- **Visual Studio Code (VSCode)** - Editor kode untuk mengedit file konfigurasi dan dokumentasi
- **GitHub Copilot Agent** - Asisten AI untuk membantu coding dan troubleshooting
- **ChatGPT** - Asisten AI untuk brainstorming solusi dan dokumentasi
- **Terminal** - Antarmuka baris perintah untuk menjalankan perintah Linux dan Docker
- **Notepad** - Editor teks sederhana untuk catatan cepat dan editing

---

## 2. TUJUAN PROYEK

Membangun sistem backup dan restore terpusat menggunakan UrBackup untuk melindungi data dari:
- **Alfresco** - Document Management System
- **Nextcloud** - File Sharing & Collaboration
- **ownCloud** - Private Cloud Storage
- **PostgreSQL** - Database untuk Alfresco

**Target:** Backup otomatis dengan kemampuan restore via Web UI


### TOOLS YANG DIGUNAKAN

Tools yang digunakan dalam pengembangan dan dokumentasi proyek ini:

- **Visual Studio Code (VSCode)** - Editor kode untuk mengedit file konfigurasi dan dokumentasi
- **GitHub Copilot Agent** - Asisten AI untuk membantu coding dan troubleshooting
- **ChatGPT** - Asisten AI untuk brainstorming solusi dan dokumentasi
- **Terminal** - Antarmuka baris perintah untuk menjalankan perintah Linux dan Docker
- **Notepad** - Editor teks sederhana untuk catatan cepat dan editing

---

## 3. FUNGSI MASING-MASING SERVICE

### Alfresco 7.4.1
**Fungsi:** Document Management System (DMS) untuk manajemen dokumen enterprise
- Port: 8080
- Data: Contentstore (file fisik) + PostgreSQL (metadata)
- Volume: backup-cloud-server_alfresco_data (23 MB)
- **Penting:** Restore memerlukan database DAN file

### ownCloud 10.14
**Fungsi:** Private cloud storage untuk file sharing internal
- Port: 8082
- Data: User files, config, database
- Volume: backup-cloud-server_owncloud_data (5.6 MB)

### Nextcloud 28
**Fungsi:** Collaboration platform dengan file sharing, calendar, contacts
- Port: 8083
- Data: User files, apps, config
- Volume: backup-cloud-server_nextcloud_data (780 MB)
- **Catatan:** Volume terbesar dalam sistem

### PostgreSQL 15
**Fungsi:** Database untuk menyimpan metadata Alfresco
- Port: 5432
- Data: Folder structure, file names, permissions
- Volume: backup-cloud-server_postgres_data (98 MB)
- **Kritis:** Harus direstore bersama Alfresco

---

## 4. ARSITEKTUR YANG DIBANGUN

```
┌─────────────────────────────────────────────┐
│ Ubuntu Host                                 │
│                                             │
│  UrBackup Client (Native)                   │
│  └─ Reads: /backup-sources/*                │
│             ↓                               │
│       (symlinks)                            │
│             ↓                               │
│  Docker Volumes:                            │
│  - backup-cloud-server_alfresco_data        │
│  - backup-cloud-server_nextcloud_data       │
│  - backup-cloud-server_owncloud_data        │
│  - backup-cloud-server_postgres_data        │
│                                             │
│  UrBackup Server (Docker Container)         │
│  - Port 55555: Web UI                       │
│  - Port 55415: Client Connection            │
│  - Storage: /backups/                       │
└─────────────────────────────────────────────┘
```

**Total Data Dibackup:** 984 MB  
**Frekuensi Backup:** Setiap 5 jam (otomatis)  
**Metode:** Incremental dengan hardlink untuk efisiensi

---

## 5. PROBLEM SOLVING - ERROR DAN SOLUSINYA

### Problem #1: Volume Naming Mismatch ❌
**Error:**
```
Backup menunjukkan 0 bytes, folder kosong di Web UI
```

**Root Cause:**
- docker-compose.yml menggunakan `alfresco-data` (dash)
- Volume sebenarnya bernama `backup-cloud-server_alfresco_data` (underscore)

**Cara Mengatasi:**
```bash
# 1. Cek nama volume yang benar
docker volume ls | grep alfresco
# Output: backup-cloud-server_alfresco_data

# 2. Update docker-compose.yml
volumes:
  - /var/lib/docker/volumes/backup-cloud-server_alfresco_data/_data:/backup-sources/alfresco:ro
  - /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data:/backup-sources/nextcloud:ro
  - /var/lib/docker/volumes/backup-cloud-server_owncloud_data/_data:/backup-sources/owncloud:ro
  - /var/lib/docker/volumes/backup-cloud-server_postgres_data/_data:/backup-sources/postgres:ro

# 3. Restart container
docker-compose restart urbackup
```

**Hasil:** ✅ Backup berhasil 765 MB → kemudian 984 MB

---

### Problem #2: Client Configuration Case Sensitivity ❌ ⚠️ PALING SULIT
**Error:**
```
Internet server not configured. Please configure 'internet_server'
Client status: offline
30+ kali konfigurasi gagal
```

**Root Cause:**
Config file menggunakan **UPPERCASE** (`INTERNET_SERVER=10.9.11.138`) tetapi UrBackup client membaca **lowercase** (`internet_server`)

**Percobaan yang Gagal:**
1. ❌ `urbackupclientctl set-settings --internet_server 10.9.11.138` → tidak ada efek
2. ❌ Membuat `server_idents.txt` manual → directory tidak ada
3. ❌ Client reset → kehilangan `pw_change.txt`
4. ❌ Preconfigured installer dari Web UI → auth key salah (UDxtqrlEqK vs pFd6P9Kxhf)

**Cara Mengatasi - SOLUSI FINAL:**
```bash
# 1. Stop client
systemctl stop urbackupclientbackend

# 2. Edit config dengan parameter LOWERCASE
nano /etc/default/urbackupclient

# Content yang BENAR:
LOGFILE=/var/log/urbackupclient.log
LOGLEVEL=debug
RESTORE=server-confirms
internet_only=true                    # lowercase
internet_mode_enabled=true            # lowercase
internet_server=10.9.11.138           # lowercase
internet_server_port=55415            # lowercase
internet_authkey=pFd6P9Kxhf           # lowercase - auth key dari server
COMPUTERNAME=backup-cloud-server

# 3. Restart client
systemctl restart urbackupclientbackend

# 4. Verify connection
urbackupclientctl status
# Output harus menunjukkan: "internet_status": "wait_local" atau "connected"
```

**Hasil:** ✅ Client connected, backup berhasil

**Pelajaran Penting:**
- Parameter harus **lowercase** meskipun biasanya config file menggunakan UPPERCASE
- Auth key harus sama dengan server (cek di Web UI: Settings → Internet)

---

### Problem #3: Preconfigured Installer Auth Key Mismatch ❌
**Error:**
```
Client terinstall tapi tidak connect
Auth key di client: UDxtqrlEqK
Auth key di server: pFd6P9Kxhf
```

**Root Cause:**
Download "Preconfigured installer" dari Web UI menghasilkan auth key baru, bukan menggunakan yang sudah ada

**Cara Mengatasi:**
```bash
# 1. Cek auth key server
# Web UI → Settings → Internet → Authentication key: pFd6P9Kxhf

# 2. Replace auth key di client
nano /etc/default/urbackupclient
# Ubah: internet_authkey=UDxtqrlEqK
# Jadi: internet_authkey=pFd6P9Kxhf

# 3. Restart client
systemctl restart urbackupclientbackend
```

**Hasil:** ✅ Client connect dengan auth key yang benar

---

### Problem #4: Multiple Client Processes / Zombie Process ❌
**Error:**
```
ps aux | grep urbackup
root  1234  ... urbackupclientbackend
root  1235  ... urbackupclientbackend <defunct>
root  1236  ... urbackupclientbackend

Error: bind() failed for 0.0.0.0:35623
```

**Root Cause:**
Client dari percobaan sebelumnya masih running, menyebabkan conflict

**Cara Mengatasi:**
```bash
# 1. Kill semua process
pkill -f urbackupclientbackend

# 2. Verify bersih
ps aux | grep urbackup
# Harus kosong

# 3. Restart service
systemctl restart urbackupclientbackend

# 4. Verify hanya 1 process
ps aux | grep urbackup
# Harus hanya 1 process
```

**Hasil:** ✅ Client running dengan 1 process saja

---

### Problem #5: Tombol Restore Tidak Muncul ❌
**Error:**
```
Backup bisa dibrowse di Web UI
Tapi tidak ada tombol "Restore folder to client"
```

**Root Cause:**
Backup melalui direct volume mount (pseudo-client) tidak mendaftarkan client sebagai active restore target

**Cara Mengatasi:**
Install **native client** di host dengan daemon yang running:
```bash
# 1. Install client
cd /tmp
wget https://hndl.urbackup.org/Client/2.5.26/UrBackup%20Client%20Linux%202.5.26.sh
sh "UrBackup Client Linux 2.5.26.sh"

# 2. Configure dan start
systemctl enable urbackupclientbackend
systemctl start urbackupclientbackend

# 3. Verify status online
# Web UI → Clients → backup-cloud-server → Status: online
```

**Hasil:** ✅ Tombol restore muncul di Web UI

---

### Problem #6: Restore Path Confusion ❌
**Error:**
```
Restore reported success tapi file tidak ketemu
Web UI: "Restore completed"
Tapi data tidak ada di /backup-sources/
```

**Root Cause:**
UrBackup restore ke **original backup path** secara default, bukan custom destination

**Cara Mengatasi:**
```bash
# 1. Stop containers sebelum restore (PENTING)
docker stop alfresco nextcloud owncloud postgres-db

# 2. Lakukan restore via Web UI
# Browse backup → Select folder → Restore folder to client

# 3. Check log untuk confirm
tail -50 /var/log/urbackupclient.log | grep -i restore
# Output: "Restore finished successfully."

# 4. Restart containers
docker start postgres-db
sleep 10
docker start alfresco nextcloud owncloud
```

**Alasan Stop Containers:**
- Mencegah data corruption
- Menghindari file lock conflict
- Menjaga database consistency

**Hasil:** ✅ Restore berhasil ke path yang benar

---

### Problem #7: Folder Testing Tidak Muncul Setelah Restore ❌ ⚠️ INSIGHT PENTING
**Error:**
```
Restore alfresco/contentstore berhasil (271 file .bin)
Tapi folder "testing" tidak muncul di Alfresco UI
```

**Root Cause - Arsitektur Alfresco:**
Alfresco menggunakan **content-addressed storage**:
- **File fisik:** Disimpan sebagai hash .bin di contentstore (contoh: 2026/1/12/10/15/abc123.bin)
- **Metadata:** Nama folder, nama file, struktur disimpan di **PostgreSQL**

```
Alfresco Architecture:
┌──────────────────────────────────────┐
│ User melihat: /Sites/testing/       │ ← Metadata dari PostgreSQL
│                                      │
│ File sebenarnya:                     │
│ contentstore/2026/1/12/.../abc.bin  │ ← File fisik
└──────────────────────────────────────┘
```

**Cara Mengatasi - SOLUSI LENGKAP:**
```bash
# LANGKAH 1: Stop containers
docker stop alfresco postgres-db

# LANGKAH 2: Restore contentstore
# Web UI → Backup → ubuntu → alfresco → Restore folder to client
# Tunggu sampai selesai

# LANGKAH 3: Restore database PostgreSQL
# Web UI → Backup → ubuntu → postgres → Restore folder to client
# Tunggu sampai selesai

# LANGKAH 4: Start database dulu
docker start postgres-db
sleep 15  # Tunggu database ready

# LANGKAH 5: Start Alfresco
docker start alfresco
sleep 30  # Tunggu Alfresco ready

# LANGKAH 6: Verify di browser
# http://10.9.11.138:8080/share
# Login → Sites → Folder "testing" harus muncul
```

**Hasil:** ✅ Folder "testing" muncul dengan struktur lengkap

**Pelajaran Kritis:**
- Untuk content management system seperti Alfresco: **SELALU restore database DAN file**
- File saja tidak cukup → metadata hilang
- Database saja tidak cukup → file fisik hilang
- **Kedua-duanya harus direstore**

---

### Problem #8: Backup Directories Tidak Terdaftar ❌
**Error:**
```
Backup hanya 25-26 MB padahal seharusnya 984 MB
urbackupclientctl list-backupdirs → empty
```

**Root Cause:**
Symlink /backup-sources/ ada tapi tidak didaftarkan ke client

**Cara Mengatasi:**
```bash
# 1. Register backup directories
urbackupclientctl add-backupdir -d /backup-sources/alfresco -f
urbackupclientctl add-backupdir -d /backup-sources/nextcloud -f
urbackupclientctl add-backupdir -d /backup-sources/owncloud -f
urbackupclientctl add-backupdir -d /backup-sources/postgres -f

# 2. Verify registration
urbackupclientctl list-backupdirs
# Output:
# PATH                      NAME      FLAGS
# /backup-sources/alfresco  alfresco  follow_symlinks,symlinks_optional
# /backup-sources/owncloud  owncloud  follow_symlinks,symlinks_optional
# /backup-sources/nextcloud nextcloud follow_symlinks,symlinks_optional
# /backup-sources/postgres  postgres  follow_symlinks,symlinks_optional

# 3. Trigger backup
# Web UI → Clients → backup-cloud-server → Start incremental backup
```

**Hasil:** ✅ Backup penuh 984 MB berhasil

---

## 6. PROSEDUR RESTORE STEP-BY-STEP

### Restore Testing (Folder Tertentu)
```bash
# 1. Stop containers yang akan direstore
docker stop alfresco postgres-db

# 2. Restore via Web UI
# - Browse backup
# - Select folder (contoh: alfresco/contentstore/testing/)
# - Click "Restore folder to client"
# - Tunggu sampai selesai

# 3. Jika service butuh database (seperti Alfresco):
#    Restore database juga
# - Browse backup
# - Select postgres folder
# - Click "Restore folder to client"

# 4. Start containers
docker start postgres-db
sleep 15
docker start alfresco
sleep 30

# 5. Verify di browser
# http://10.9.11.138:8080/share
```

### Restore Production (Full Restore)
```bash
# 1. Stop SEMUA containers
docker stop alfresco nextcloud owncloud postgres-db redis solr

# 2. Restore via Web UI untuk SETIAP service
# Restore order:
#   a. postgres (database dulu)
#   b. alfresco
#   c. nextcloud
#   d. owncloud

# 3. Start dengan urutan:
docker start postgres-db
sleep 20
docker start redis solr
sleep 10
docker start alfresco nextcloud owncloud

# 4. Verify semua service
docker ps  # Semua harus running
curl http://10.9.11.138:8080/share  # Alfresco
curl http://10.9.11.138:8083  # Nextcloud
curl http://10.9.11.138:8082  # ownCloud
```

---

## 7. KONFIGURASI PENTING

### File /etc/default/urbackupclient (YANG BENAR)
```ini
LOGFILE=/var/log/urbackupclient.log
LOGLEVEL=debug
RESTORE=server-confirms
internet_only=true
internet_mode_enabled=true
internet_server=10.9.11.138
internet_server_port=55415
internet_authkey=pFd6P9Kxhf
COMPUTERNAME=backup-cloud-server
```

**Catatan:**
- Parameter internet_* harus **lowercase**
- Auth key harus sama dengan server
- COMPUTERNAME bisa UPPERCASE

### Symlinks untuk Backup
```bash
# /backup-sources/ → Docker volumes
ln -s /var/lib/docker/volumes/backup-cloud-server_alfresco_data/_data /backup-sources/alfresco
ln -s /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/_data /backup-sources/nextcloud
ln -s /var/lib/docker/volumes/backup-cloud-server_owncloud_data/_data /backup-sources/owncloud
ln -s /var/lib/docker/volumes/backup-cloud-server_postgres_data/_data /backup-sources/postgres
```

### Backup Directories Registration
```bash
urbackupclientctl add-backupdir -d /backup-sources/alfresco -f
urbackupclientctl add-backupdir -d /backup-sources/nextcloud -f
urbackupclientctl add-backupdir -d /backup-sources/owncloud -f
urbackupclientctl add-backupdir -d /backup-sources/postgres -f
```

---

## 8. HASIL AKHIR

### Status Sistem
✅ **UrBackup Server:** Running di Docker (uroni/urbackup-server:latest)  
✅ **UrBackup Client:** Connected via LAN mode (172.18.0.10)  
✅ **Backup Size:** 984 MB total  
✅ **Backup Frequency:** Setiap 5 jam (otomatis)  
✅ **Restore:** Tested dan berfungsi dengan Web UI  
✅ **Database Integrity:** PostgreSQL + Alfresco restore berhasil

### Breakdown Backup
| Service | Size | Status |
|---------|------|--------|
| Alfresco | 23 MB | ✅ Backed up |
| Nextcloud | 780 MB | ✅ Backed up |
| ownCloud | 5.6 MB | ✅ Backed up |
| PostgreSQL | 98 MB | ✅ Backed up |
| **TOTAL** | **984 MB** | ✅ |

### Testing Validation
✅ Folder "testing" dibuat di Alfresco  
✅ Folder dihapus  
✅ Restore dilakukan (contentstore + database)  
✅ Folder "testing" kembali dengan struktur lengkap  

**User Confirmation:** "nice banget folder berhasil kembali untuk folder testing nya"

---

## 9. COMMAND CHEATSHEET

### Client Management
```bash
# Status client
urbackupclientctl status

# List backup directories
urbackupclientctl list-backupdirs

# Add backup directory
urbackupclientctl add-backupdir -d /path/to/dir -f

# View logs
tail -f /var/log/urbackupclient.log

# Restart client
systemctl restart urbackupclientbackend
```

### Backup Verification
```bash
# Check backup size
du -sh /var/lib/docker/volumes/backup-cloud-server_alfresco_data/
du -sh /var/lib/docker/volumes/backup-cloud-server_nextcloud_data/
du -sh /var/lib/docker/volumes/backup-cloud-server_owncloud_data/
du -sh /var/lib/docker/volumes/backup-cloud-server_postgres_data/

# Check symlinks
ls -lh /backup-sources/

# Docker volume list
docker volume ls | grep backup-cloud-server
```

### Troubleshooting
```bash
# Kill stuck client processes
pkill -f urbackupclientbackend

# Check for zombie processes
ps aux | grep urbackup | grep defunct

# Verify client config
cat /etc/default/urbackupclient | grep internet_

# Test client connection
urbackupclientctl status | grep internet_status
```

---

## 10. PELAJARAN PENTING

### ✅ DO's
1. **Selalu gunakan lowercase** untuk parameter internet_* di config client
2. **Stop containers** sebelum restore untuk menghindari corruption
3. **Restore database DAN file** untuk content management system (Alfresco)
4. **Verify backup directories** terdaftar dengan `urbackupclientctl list-backupdirs`
5. **Check logs** di `/var/log/urbackupclient.log` saat troubleshooting
6. **Test restore** secara berkala untuk memastikan backup berfungsi

### ❌ DON'Ts
1. **Jangan gunakan UPPERCASE** untuk internet_server, internet_authkey, dll
2. **Jangan restore saat container running** → risk data corruption
3. **Jangan hanya restore file tanpa database** untuk Alfresco → metadata hilang
4. **Jangan gunakan dash** dalam nama volume di docker-compose.yml → use underscore
5. **Jangan pakai preconfigured installer** → auth key akan berbeda, edit manual lebih reliable

### 🎯 Key Takeaways
- **Client config case sensitivity** adalah masalah tersembunyi yang sulit didiagnosa
- **Alfresco memerlukan database + file** untuk restore lengkap
- **Symlinks dengan follow_symlinks flag** bekerja sempurna untuk Docker volumes
- **Native client lebih reliable** daripada pseudo-client untuk restore capability
- **Web UI restore** jauh lebih mudah daripada command line

---

## 11. TROUBLESHOOTING CEPAT

| Gejala | Kemungkinan Penyebab | Solusi Cepat |
|--------|---------------------|--------------|
| Client offline | Config case sensitivity | Edit `/etc/default/urbackupclient`, lowercase parameters |
| Backup 0 bytes | Volume name salah | Check `docker volume ls`, fix docker-compose.yml |
| Auth failed | Auth key mismatch | Match dengan server di Web UI Settings |
| Multiple processes | Zombie process | `pkill -f urbackupclientbackend`, restart service |
| No restore button | Pseudo-client | Install native client dengan daemon |
| Restore tidak muncul | Path confusion | Check `/var/log/urbackupclient.log` untuk actual path |
| Folder hilang (Alfresco) | Database tidak direstore | Restore postgres DAN alfresco |
| Backup kecil | Directory tidak terdaftar | `urbackupclientctl add-backupdir -d /path -f` |

---

## 12. KESIMPULAN

Sistem backup dan restore menggunakan UrBackup berhasil diimplementasikan setelah mengatasi 8 masalah utama. Tantangan terbesar adalah **client configuration case sensitivity** yang memakan waktu paling lama untuk troubleshooting.

**Highlight:**
- ✅ 984 MB data ter-backup otomatis setiap 5 jam
- ✅ Restore via Web UI user-friendly dan tested
- ✅ Alfresco, Nextcloud, ownCloud, PostgreSQL fully protected
- ✅ Database integrity terjaga dengan restore procedure yang benar

**Total Waktu Troubleshooting:** ~16 jam  
**Waktu Paling Lama:** Client configuration (4 jam)  
**Status Final:** ✅ Production Ready

---

**Prepared by:** UrBackup Integration Team  
**Date:** Januari 12, 2026  
**Version:** 1.0
