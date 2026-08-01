# auto-mysql-backup
Reliable MySQL backup automation with multi-destination uploads (FTP / FTPS / SFTP), AES-256 GPG encryption, Telegram notifications (supports multiple Chat IDs, optional proxy), and an intelligent failover retry system.

## Table of Contents
- [What this script does](#what-this-script-does)
- [Features](#features)
- [How it works](#how-it-works)
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
  - [Single-database backup](#single-database-backup)
  - [Help](#help)
  - [Scheduling with cron](#scheduling-with-cron)
- [Encryption & Decryption](#encryption--decryption)
- [Telegram notifications](#telegram-notifications)
  - [Optional Telegram proxy](#optional-telegram-proxy)
- [Log file & permissions](#log-file--permissions)
- [Retention & cleanup](#retention--cleanup)
- [Operational notes & hardening](#operational-notes--hardening)
- [Troubleshooting](#troubleshooting)
- [Changelog](#changelog)
- [License](#license)
- [Support This Project](#support-this-project)

---

## What this script does
- Backs up **all non-system MySQL databases** (or a single one via `--only-db`), compresses them (`.sql.gz`), verifies archive integrity, optionally **encrypts** them to AES-256 (`.gpg`), then **uploads** the results to **one or more** destinations across FTP, FTPS, and SFTP.
- Uploads use **retry with backoff** (3 attempts, 30-second delay) and won't stop the run if a destination fails.
- Produces detailed logs and (optionally) sends a concise **Telegram report** plus the log file as a document — optionally routed through a proxy.
- Prevents overlapping runs with a **file lock** and aborts safely if MySQL is unreachable.

## Features
- **Config via `.env`** (no code edits required)
- **Multi-destination** uploads (FTP / FTPS / SFTP)
- **Failover & retry** logic (3×, 30s delay)
- **Optional AES-256 GPG encryption**
- **Archive integrity check** (`gzip -t`) before upload
- **Telegram** alerts + **log upload** (supports multiple Chat IDs, optional proxy)
- **Auto cleanup** of old backups — both **locally** and on **remote destinations**
- **Concurrency lock** (`flock`) to prevent overlapping runs
- **`--only-db`** flag for single-database backup/debugging
- **`--help`** flag for built-in usage reference
- **Verbose logging** to `backup_${DATE}.log`

## How it works
1. Reads configuration from an external `.env` file.
2. Acquires a lock (`flock`) to prevent concurrent runs.
3. Verifies MySQL connectivity before doing anything else.
4. Enumerates MySQL databases (excludes `mysql`, `sys`, etc.), or backs up only the database given via `--only-db`.
5. Dumps each DB → `mysqldump | gzip` → `YYYY-MM-DD/dbname.sql.gz`, using accurate pipe exit-code detection.
6. Verifies each archive's integrity with `gzip -t`; corrupt archives are discarded and reported.
7. (Optional) Encrypts each `.gz` to `.gz.gpg` and deletes the plaintext `.gz` only on success.
8. Deletes local backup folders older than the retention period (based on folder name, not mtime).
9. Deletes matching old backup folders from remote FTP/FTPS/SFTP destinations too.
10. Uploads each file to all configured destinations with retry logic (no `eval`, fully array-based).
11. Writes a summary (with real start/end timestamps and total backup size) to Telegram.

---

## Prerequisites
> The following packages must be installed before running the script.

### Debian / Ubuntu
```bash
sudo apt update
sudo apt install -y gzip curl gnupg sshpass lftp
```

### RHEL / CentOS / Rocky / AlmaLinux
```bash
sudo dnf install -y gzip curl gnupg2 sshpass lftp
# For older CentOS systems:
# sudo yum install -y gzip curl gnupg2 sshpass lftp
```

> **`lftp` is required** for remote cleanup of old backup folders on FTP/FTPS destinations (recursive directory deletion isn't supported by `curl` alone). If `lftp` is missing, the script will skip remote FTP/FTPS cleanup and log a warning — local cleanup and uploads still work normally.

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
- Optional Telegram settings for notifications, including optional proxy settings.

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

### Single-database backup
Useful for quick debugging or testing without backing up everything:
```bash
sudo mysql_backup --only-db=myapp_production
```

### Help
```bash
mysql_backup --help
```
Displays usage, available options, config file path, and log file location — works without root or an existing `.env` file.

### Scheduling with cron
Run daily at 2:00 AM:
```bash
sudo crontab -e
# Add this line:
0 2 * * * /usr/local/bin/mysql_backup >> /var/log/mysql_backup_cron.log 2>&1
```
> Thanks to the built-in `flock` lock, it's safe if a manual run overlaps with a scheduled one — the second invocation exits immediately instead of running concurrently.

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

The script sends start and completion summaries — each with an accurate timestamp reflecting the moment that message was actually sent — and can also attach the log file to **all Chat IDs**. The completion summary also reports total backup size and duration.

### Optional Telegram proxy
If Telegram's API is restricted on your server's network, you can route **only the Telegram notification requests** through an HTTP or SOCKS5 proxy (backups, uploads, and MySQL access are unaffected):

```bash
TELEGRAM_PROXY_ENABLED=true
TELEGRAM_PROXY_TYPE="socks5h"    # socks5h resolves DNS through the proxy too
TELEGRAM_PROXY_ADDR="127.0.0.1:1080"
TELEGRAM_PROXY_USER=""           # optional
TELEGRAM_PROXY_PASS=""           # optional
```
> Use `socks5h` (not `socks5`) if you need `api.telegram.org` itself resolved through the proxy rather than locally.

---

## Log file & permissions
The script automatically writes logs to:
```
$BACKUP_BASE_DIR/$DATE/backup_${DATE}.log
```
Example default path (from `.env`):
```
/backup/dbbackup/2026-08-01/backup_2026-08-01.log
```
### Log structure
- Backup start and end timestamps
- Success/failure for each database, including integrity check results
- Upload attempts (with retries), per destination
- Encryption results and destination statuses
- Local and remote cleanup actions

### Restrict log access
Because the log may contain sensitive paths or errors, the script automatically applies:
```bash
chmod 600 "$LOG_FILE"
```
at the end of every run, limiting read/write access to the owner (root) only.

---

## Retention & cleanup
- Old backup folders are identified **by folder name** (`YYYY-MM-DD`), not by filesystem modification time — this avoids incorrectly keeping or deleting folders due to touched files or clock changes.
- Cleanup runs **locally first**, then (if uploads are enabled) also removes matching old folders from every configured **FTP**, **FTPS**, and **SFTP** destination.
- Remote FTP/FTPS cleanup requires `lftp` (see [Prerequisites](#prerequisites)); if missing, remote cleanup is skipped with a warning, but local cleanup and uploads still proceed normally.
- Remote SFTP cleanup requires shell access on the destination (not SFTP-chroot-only accounts).

---

## Operational notes & hardening
- Backup directory must be writable.
- Prefer SFTP/FTPS for encrypted transfers.
- Consider log rotation for `backup_${DATE}.log`.
- Limit access to the `.env` file strictly with `chmod 600`.
- Log access is automatically restricted with `chmod 600` at the end of every run.
- The script uses `set -o pipefail` and `PIPESTATUS` to correctly detect `mysqldump` failures even when piped into `gzip`.
- Uploads are fully array-based (no `eval`), reducing the risk of command injection from passwords or paths containing special characters.
- A MySQL connectivity check runs before any backup attempt, so credential issues are caught immediately instead of silently producing an empty backup.

---

## Troubleshooting
| Issue | Solution |
|-------|-----------|
| `❌ Config file not found` | Ensure the `.env` path matches your setup. |
| `⚠️ Another instance of mysql_backup is already running` | A previous run (manual or cron) hasn't finished yet; wait for it or check for a stuck process. |
| `❌ Cannot connect to MySQL server` | Verify `MYSQL_USER`/`MYSQL_PASSWORD`/`USE_PASSWORD` and that the MySQL socket/service is reachable. |
| `❌ Integrity check failed ... (corrupt archive)` | The `.sql.gz` failed `gzip -t`; check disk space and MySQL server health during the dump. |
| Upload fails repeatedly | Verify credentials, remote path, or firewall. |
| `⚠️ lftp not installed, skipping remote ftp cleanup` | Install `lftp` (see [Prerequisites](#prerequisites)) to enable remote FTP/FTPS retention cleanup. |
| No backups produced | Check MySQL credentials; some setups allow root without a password. |
| Telegram not sending | Confirm Bot Token and Chat IDs; ensure outbound Internet, or configure `TELEGRAM_PROXY_*` if Telegram is restricted on your network. |

---

## Changelog

### v3 (hardened release)
- **Fixed pipe exit-code bug**: `mysqldump | gzip` failures are now correctly detected via `set -o pipefail` + `PIPESTATUS`, instead of only checking `gzip`'s exit code.
- **Added archive integrity check**: every `.sql.gz` is verified with `gzip -t` before being encrypted/uploaded; corrupt archives are discarded and reported.
- **Added MySQL connectivity check**: the script now aborts early (with a Telegram alert) if it can't connect to MySQL, instead of silently reporting a false "success".
- **Removed `eval`**: all upload commands are now built as bash arrays instead of `eval`-ed strings, closing a potential command-injection risk from special characters in passwords/paths.
- **Added concurrency lock (`flock`)**: prevents two instances (e.g. cron + manual run) from running at the same time.
- **Retention rewritten**: local cleanup now matches folders by name (`YYYY-MM-DD`) instead of filesystem mtime; old folders are also now removed from remote FTP/FTPS (via `lftp`) and SFTP destinations, not just locally.
- **Added `--only-db=DBNAME`**: back up a single database, useful for debugging.
- **Added `--help` / `-h`**: built-in usage reference, works without root or an existing `.env`.
- **Added optional Telegram proxy support**: route only Telegram API calls through an HTTP/SOCKS5 proxy via `TELEGRAM_PROXY_*` variables.
- **Added backup size reporting**: Telegram completion summary now includes total backup size (`du -sh`).
- **Fixed Telegram timestamp bug**: the completion message now shows the actual completion time instead of reusing the start timestamp.
- **Quoted `$LOG_FILE`**: fixed a potential failure when the log path contains spaces.
- **Added `nullglob`**: prevents literal glob patterns (e.g. `*.gz`) from being treated as real filenames when no files match.

### v2
- Introduced multi-destination FTP/FTPS/SFTP uploads with retry/backoff.
- Introduced Telegram notifications with multi-Chat-ID support.
- Introduced optional AES-256 GPG encryption.

---

## License
Released under the **MIT License**. See `LICENSE` for details.

---

## Support This Project

If this project saved you time or helped your team, consider supporting its
development with a small crypto donation:

| Currency | Address |
|---|---|
| USDT (BEP-20) | `0xD18e0A300a758bdb9d64e321D3A2c80D76Ee27fd` |
| USDT (TRC-20) | `TUD2UytTuw3KsWSJfy73LHZ3HrCwFWtBSN` |
| Bitcoin | `bc1qrkxh9nw7ntru33687qwr36j9nq5nzky2eny5gg` |

> ⚠️ Double-check the network before sending — BEP-20 and TRC-20 addresses are
> **not interchangeable**. Sending USDT on the wrong network may result in
> permanent loss of funds.

Every bit of support helps keep this project maintained and free for everyone.
⭐ Starring the repo also helps a lot — it's free and takes two seconds.
