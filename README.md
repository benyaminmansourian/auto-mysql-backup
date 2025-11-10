# auto-mysql-backup
Reliable MySQL backup automation with multi-destination uploads (FTP / FTPS / SFTP), AES-256 GPG encryption, Telegram notifications (supports multiple Chat IDs), and an intelligent failover retry system.

## Table of Contents
- [What this script does](#what-this-script-does)
- [Features](#features)
- [How it works (at a glance)](#how-it-works-at-a-glance)
- [Prerequisites](#prerequisites)
  - [Debian / Ubuntu](#debian--ubuntu)
  - [RHEL / CentOS / Rocky / AlmaLinux](#rhel--centos--rocky--almalinux)
- [Installation](#installation)
- [Configuration](#configuration)
  - [The `.env` file (high level)](#the-env-file-high-level)
  - [Secure the `.env`](#secure-the-env)
  - [Destination format](#destination-format)
- [Usage](#usage)
  - [Manual run](#manual-run)
  - [Scheduling with cron](#scheduling-with-cron)
- [Encryption & Decryption](#encryption--decryption)
- [Telegram notifications](#telegram-notifications)
- [Log file & permissions](#log-file--permissions)
- [Operational notes & hardening](#operational-notes--hardening)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## What this script does
- Backs up **all non-system MySQL databases**, compresses them (`.sql.gz`), optionally **encrypts** them to AES-256 (`.gpg`), then **uploads** the results to **one or more** destinations across FTP, FTPS, and SFTP.
- Uploads use **retry with backoff** (3 attempts, 30-second delay) and won’t stop the run if a destination fails.
- Produces detailed logs and (optionally) sends a concise **Telegram report** plus the log file as a document.

## Features
- **Config via `.env`** (no code edits required)
- **Multi-destination** uploads (FTP / FTPS / SFTP)
- **Failover & retry** logic (3×, 30s delay)
- **Optional AES-256 GPG encryption**
- **Telegram** alerts + **log upload (supports multiple Chat IDs)**
- **Auto cleanup** of old backups
- **Verbose logging** to `backup.log`

## How it works (at a glance)
1. Reads configuration from an external `.env` file.
2. Enumerates MySQL databases (excludes `mysql`, `sys`, etc.).
3. Dumps each DB → `mysqldump | gzip` → `YYYY-MM-DD/dbname.sql.gz`.
4. (Optional) Encrypts each `.gz` to `.gz.gpg` and deletes the plaintext `.gz`.
5. Uploads each file to all configured destinations with retry logic.
6. Cleans up older backup directories per retention policy.
7. Writes a summary (and optional details) to Telegram.

---

## Prerequisites
> The following packages must be installed before running the script.

### Debian / Ubuntu
```bash
sudo apt update
sudo apt install -y gzip curl gnupg sshpass
```

### RHEL / CentOS / Rocky / AlmaLinux
```bash
sudo dnf install -y gzip curl gnupg2 sshpass
# For older CentOS systems:
# sudo yum install -y gzip curl gnupg2 sshpass
```

---

## Installation
```bash
# 1) Clone the repository
git clone https://github.com/benyaminmansourian/auto-mysql-backup.git
cd auto-mysql-backup

# 2) Move your environment file to a secure system path
# Make sure to edit it before moving if needed
sudo mv mysql_backup.env /etc/

# 3) Install the backup script globally so it can be run from anywhere
chmod 755 mysql_backup
sudo mv mysql_backup /usr/local/bin/

# 4) Cleanup - remove the cloned repository
cd ..
sudo rm -rf auto-mysql-backup
```
> Ensure the path inside the script matches where you store the `.env` file (default: `/etc/mysql_backup.env`).

---

## Configuration
### The `.env` file (high level)
The script reads configuration such as:
- Backup location, MySQL credentials, retention period.
- Optional encryption (AES-256 GPG).
- Multi-destination upload targets for FTP, FTPS, and SFTP.
- Optional Telegram settings for notifications.

> A ready-to-edit `mysql_backup.env` file is provided separately.

### Secure the `.env`
Since `.env` contains credentials, restrict permissions:
```bash
sudo chown root:root /etc/mysql_backup.env
sudo chmod 600 /etc/mysql_backup.env
```
- `chmod 600` ensures only the owner (root) can read/write the file.

### Destination format
Each destination entry follows this format:
```
host|user|password|remote_path
```
Example:
```
ftp1.example.com|ftpuser|ftppass|server1/backups
sftp1.example.com|sftpuser|sftppass|/remote/backups
```
> Prefer **SFTP** or **FTPS** for secure transfers.

---

## Usage
### Manual run
```bash
sudo mysql_backup
```
> For the first run, keep destination arrays empty to verify backups locally.

### Scheduling with cron
Run daily at 2:00 AM:
```bash
sudo crontab -e
# Add this line:
0 2 * * * /usr/local/bin/mysql_backup >> /var/log/mysql_backup_cron.log 2>&1
```

---

## Encryption & Decryption
If encryption is enabled, backups become `.gz.gpg` files.
To decrypt:
```bash
gpg --batch --yes --passphrase "YOUR_PASSWORD" -o database.sql.gz -d database.sql.gz.gpg
gunzip database.sql.gz
```

---

## Telegram notifications
1. Create a bot via **@BotFather**.
2. Get your **Bot Token** and **Chat IDs**.
3. Set `TELEGRAM_ENABLED=true` in `.env` and configure `TELEGRAM_BOT_TOKEN`.
4. To send notifications to multiple users, define an array:

```bash
TELEGRAM_CHAT_IDS=("123456789" "987654321")
```
> If only one chat is needed, you can still use:
> ```bash
> TELEGRAM_CHAT_ID="123456789"
> ```

The script sends start and completion summaries and can also attach the log file to **all Chat IDs**.

---

## Log file & permissions
The script automatically writes logs to:
```
$BACKUP_BASE_DIR/backup.log
```
Example default path (from `.env`):
```
/backup/dbbackup/backup.log
```
### Log structure
- Backup start and end timestamps
- Success/failure for each database
- Upload attempts (with retries)
- Encryption results and destination statuses

### Restrict log access
Because the log may contain sensitive paths or errors:
```bash
sudo chown root:root /backup/dbbackup/backup.log
sudo chmod 600 /backup/dbbackup/backup.log
```
This limits read/write access to root only, improving data security.

---

## Operational notes & hardening
- Backup directory must be writable.
- Prefer SFTP/FTPS for encrypted transfers.
- Consider log rotation for `backup.log`.
- Limit access to the `.env` file strictly with `chmod 600`.
- Restrict log access using `chmod 600` as shown above.

---

## Troubleshooting
| Issue | Solution |
|-------|-----------|
| `❌ Config file not found` | Ensure the `.env` path matches your setup. |
| Upload fails repeatedly | Verify credentials, remote path, or firewall. |
| No backups produced | Check MySQL credentials; some setups allow root without a password. |
| Telegram not sending | Confirm Bot Token and Chat IDs; ensure outbound Internet. |

---

## License
Released under the **MIT License**. See `LICENSE` for details.
