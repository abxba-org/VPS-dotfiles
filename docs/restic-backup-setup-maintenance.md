# Restic Backup Setup & Maintenance

This document describes the local encrypted Restic backup system configured on the Ubuntu VPS `vps-1558ffa7`.

It covers the initial setup, configuration, automated backups, retention, verification, restoration, troubleshooting, and ongoing maintenance.

## 1. System Overview

| Item | Configuration |
|---|---|
| Operating System | Ubuntu Linux |
| Hostname | vps-1558ffa7 |
| User | ubuntu |
| Shell | Zsh |
| Backup Software | Restic |
| Backup Repository | /home/ubuntu/restic/restic-repo |
| Source 1 | /home/ubuntu/CODE |
| Source 2 | /home/ubuntu/docs |
| Password File | /home/ubuntu/.restic_password |
| Backup Script | /home/ubuntu/scripts/restic_backup.sh |
| Backup Log | /home/ubuntu/scripts/backup.log |
| Schedule | Daily at 02:00 |
| Retention | 7 daily, 4 weekly, 12 monthly |

The repository is stored locally on the VPS and is encrypted by Restic.

> **Important:** A local Restic repository protects against accidental deletion or file corruption, but it does not protect against VPS disk failure, VPS loss, or a compromised server. A separate remote/off-site repository should be considered for a complete backup strategy.

## 2. Directory Structure

The backup system uses the following structure:

```text
/home/ubuntu/
├── CODE/
├── docs/
├── restic/
│   └── restic-repo/
├── scripts/
│   ├── restic_backup.sh
│   └── backup.log
└── .restic_password
```

The important components are:

- `CODE/` — application/source files being backed up.
- `docs/` — documentation/data being backed up.
- `restic/restic-repo/` — encrypted Restic repository.
- `scripts/restic_backup.sh` — automated backup script.
- `scripts/backup.log` — output from automated/manual backup runs.
- `.restic_password` — password used to encrypt/decrypt the Restic repository.

## 3. Install Restic

If Restic is not already installed:

```bash
sudo apt update
sudo apt install restic
```

Verify the installation:

```bash
restic version
```

Example:

```text
restic 0.x.x
```

## 4. Create the Restic Password File

The repository requires a password.

Create the password file:

```bash
nano ~/.restic_password
```

Put the Restic encryption password inside the file.

Then restrict permissions:

```bash
chmod 600 ~/.restic_password
```

Verify the permissions:

```bash
ls -l ~/.restic_password
```

The file should only be readable/writable by the ubuntu user.

Example:

```text
-rw------- 1 ubuntu ubuntu ... /home/ubuntu/.restic_password
```

### Security

- Never commit `.restic_password` to Git.
- Never put the password directly into the backup script.
- Never publish the password in documentation, screenshots, logs, or shell history.
- If the password is ever exposed, create a new repository/password strategy and migrate the backups as appropriate. Restic cannot change the encryption password of an existing repository in the way a normal account password can be rotated; treat the repository password as critical recovery information.

## 5. Create the Repository

Create the directory:

```bash
mkdir -p ~/restic/restic-repo
```

Initialize the repository:

```bash
restic init \
  --repo /home/ubuntu/restic/restic-repo \
  --password-file /home/ubuntu/.restic_password
```

A successful initialization produces a message similar to:

```text
created restic repository ...
```

The repository now contains Restic's encrypted repository structure.

> **Important**
>
> `restic init` is only required when creating a new repository.
>
> Do not repeatedly run:
>
> ```bash
> restic init
> ```
>
> against an existing repository.
>
> If you see:
>
> ```text
> config file already exists
> ```
>
> the repository has already been initialized.

## 6. Verify the Repository

Before creating the automation script, verify that Restic can open the repository:

```bash
restic \
  -r /home/ubuntu/restic/restic-repo \
  --password-file /home/ubuntu/.restic_password \
  snapshots
```

If the repository has no backups yet, an empty snapshot list is expected.

After backups exist, the command will show something similar to:

```text
ID        Time                 Host              Paths
--------------------------------------------------------------
xxxxxxxx  2026-09-28 ...      vps-1558ffa7      /home/ubuntu/CODE
                                                 /home/ubuntu/docs
```

## 7. Create the Backup Script

Create the scripts directory:

```bash
mkdir -p ~/scripts
```

Create the backup script:

```bash
nano ~/scripts/restic_backup.sh
```

Use:

```bash
#!/bin/bash

# Stop the script if a command fails.
set -e

# Restic repository
export RESTIC_REPOSITORY="/home/ubuntu/restic/restic-repo"

# Restic encryption password
export RESTIC_PASSWORD_FILE="/home/ubuntu/.restic_password"

echo "Starting backup..."

# Create a new backup
restic backup \
    /home/ubuntu/CODE \
    /home/ubuntu/docs

echo "Cleaning up old snapshots..."

# Retention policy:
# - Keep 7 daily snapshots
# - Keep 4 weekly snapshots
# - Keep 12 monthly snapshots
restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune

echo "Checking repository..."

# Verify repository integrity
restic check

echo "Backup process complete!"
```

Save the file.

Make it executable:

```bash
chmod +x ~/scripts/restic_backup.sh
```

Verify:

```bash
ls -l ~/scripts/restic_backup.sh
```

## 8. Why Absolute Paths Are Used

The backup script intentionally uses:

- `/home/ubuntu/CODE`
- `/home/ubuntu/docs`

rather than:

- `home/ubuntu/CODE`
- `home/ubuntu/docs`

and rather than relying on the current working directory.

This is particularly important because the script is executed by cron, where the working environment can differ from an interactive Zsh session.

The same principle is used for:

- `RESTIC_REPOSITORY`
- `RESTIC_PASSWORD_FILE`

Using absolute paths makes the automation more predictable.

## 9. Test the Backup Manually

Before configuring cron, run the script manually:

```bash
~/scripts/restic_backup.sh
```

or:

```bash
/home/ubuntu/scripts/restic_backup.sh
```

A successful backup should contain output similar to:

```text
Starting backup...
repository ... opened
Files: ...
Dirs: ...
snapshot ... saved

Cleaning up old snapshots...
Applying Policy: keep 7 daily, 4 weekly, 12 monthly

Checking repository...
no errors were found

Backup process complete!
```

The important success indicators are:

- `snapshot ... saved`
- `no errors were found`

## 10. Check Backup Snapshots

List all snapshots:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

This is one of the most useful maintenance commands.

It shows:

- Snapshot ID
- Date/time
- Host
- Paths
- Tags, if used

## 11. Automated Backups With Cron

Edit the user's crontab:

```bash
crontab -e
```

Add:

```cron
0 2 * * * /home/ubuntu/scripts/restic_backup.sh > /home/ubuntu/scripts/backup.log 2>&1
```

This means:

- Minute: `0`
- Hour: `2`
- Day: `*`
- Month: `*`
- Weekday: `*`

Therefore: Every day at `02:00`.

The output is redirected to:

`/home/ubuntu/scripts/backup.log`

## 12. Verify the Cron Configuration

Show the current user's crontab:

```bash
crontab -l
```

Expected:

```cron
0 2 * * * /home/ubuntu/scripts/restic_backup.sh > /home/ubuntu/scripts/backup.log 2>&1
```

Make sure the script is executable:

```bash
ls -l /home/ubuntu/scripts/restic_backup.sh
```

## 13. Check Backup Logs

View the latest backup output:

```bash
cat ~/scripts/backup.log
```

For easier monitoring:

```bash
tail -f ~/scripts/backup.log
```

Look for:

- `snapshot ... saved`
- `no errors were found`

If the backup fails, the error should normally appear in:

`/home/ubuntu/scripts/backup.log`

## 14. Retention Policy

The configured retention policy is:

- 7 daily
- 4 weekly
- 12 monthly

It is implemented with:

```bash
restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune
```

The policy means Restic automatically removes snapshots that are no longer required by the retention rules.

`--prune` then removes repository data that is no longer referenced by retained snapshots.

This keeps the repository from growing indefinitely.

## 15. Check Repository Integrity

Run:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  check
```

A successful check ends with:

```text
no errors were found
```

The automated backup script also runs this check after pruning.

## 16. Restore Files

Backups are only useful if they can actually be restored.

Create a temporary restore directory:

```bash
mkdir -p ~/restic-restore-test
```

Restore the latest snapshot:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  restore latest \
  --target ~/restic-restore-test
```

After restoration, inspect the files:

```bash
ls -la ~/restic-restore-test
```

The restored directory structure will contain the backed-up paths.

## 17. Restore a Specific Snapshot

First list snapshots:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

Example:

```text
ID        Time                 Host
------------------------------------------------
dcb2917f  2026-09-28 18:41    vps-1558ffa7
58c8224a  2026-09-28 18:43    vps-1558ffa7
```

Restore a specific snapshot:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  restore 58c8224a \
  --target ~/restic-restore
```

## 18. Browse a Snapshot Before Restoring

You can inspect a snapshot without restoring everything:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  ls latest
```

This is useful when you only need to find a particular file.

## 19. Restore a Specific File

First find the file:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  find filename
```

Then restore the required snapshot/path.

For example:

```bash
restic \
  -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  restore latest \
  --target ~/restic-restore \
  --include /home/ubuntu/CODE/example.txt
```

## 20. Useful Operational Commands

### List snapshots

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

### Check repository

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  check
```

### View repository statistics

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  stats
```

### List files in the latest snapshot

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  ls latest
```

### Run a backup manually

```bash
~/scripts/restic_backup.sh
```

### View backup log

```bash
cat ~/scripts/backup.log
```

### Follow backup log live

```bash
tail -f ~/scripts/backup.log
```

### View cron configuration

```bash
crontab -l
```

## 21. Recommended `do-backup` Zsh Alias

A convenient shortcut can be added to:

`~/.zshrc`

Add:

```bash
alias do-backup="/home/ubuntu/scripts/restic_backup.sh"
```

Then reload Zsh configuration:

```bash
source ~/.zshrc
```

Now the backup can be started with:

```bash
do-backup
```

Verify the alias:

```bash
alias do-backup
```

Expected:

```text
do-backup='/home/ubuntu/scripts/restic_backup.sh'
```

## 22. Zsh Configuration Notes

The correct Zsh configuration file is:

`~/.zshrc`

which resolves to:

`/home/ubuntu/.zshrc`

For example:

```bash
nano ~/.zshrc
```

A common mistake is:

```bash
nano ~./zshrc
```

This is incorrect.

Use:

```bash
nano ~/.zshrc
```

After modifying it:

```bash
source ~/.zshrc
```

## 23. Backup Maintenance Procedure

Periodically perform the following checks.

### Step 1 — Check that snapshots exist

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

### Step 2 — Check the latest backup log

```bash
tail -100 ~/scripts/backup.log
```

### Step 3 — Check repository integrity

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  check
```

### Step 4 — Verify cron

```bash
crontab -l
```

### Step 5 — Periodically perform a test restore

```bash
mkdir -p ~/restic-restore-test

restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  restore latest \
  --target ~/restic-restore-test
```

Then verify that the expected files are present.

## 24. Troubleshooting

### Repository does not exist

If you see:

```text
repository does not exist
```

verify the repository:

```bash
ls -la ~/restic/restic-repo
```

It should contain Restic repository files/directories including:

- `config`
- `data`
- `index`
- `keys`
- `snapshots`

Then test:

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

### `config file already exists`

If:

```bash
restic init ...
```

returns:

```text
config file already exists
```

the repository has already been initialized.

Do not run `restic init` again.

Instead, use:

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

### Source directory does not exist

If the backup reports:

```text
/home/ubuntu/docs does not exist, skipping
```

verify:

```bash
ls -ld /home/ubuntu/CODE
ls -ld /home/ubuntu/docs
```

If a directory is intentionally not present, remove it from the backup script.

### Backup works manually but not with cron

Cron has a much smaller environment than an interactive Zsh session.

Use absolute paths in the script.

Check:

```bash
crontab -l
```

Check:

```bash
cat ~/scripts/backup.log
```

Make sure the script is executable:

```bash
chmod +x ~/scripts/restic_backup.sh
```

## 25. Important Security Considerations

The Restic password is extremely important.

The repository is encrypted, and knowledge of the password is required to decrypt the backup.

Protect:

`/home/ubuntu/.restic_password`

with:

```bash
chmod 600 ~/.restic_password
```

Check:

```bash
stat ~/.restic_password
```

Do not:

- Commit the password to Git.
- Put it inside `restic_backup.sh`.
- Put it in public documentation.
- Share it in screenshots.
- Put it into public shell history.
- Store it in an unsecured shared location.

Also ensure the Restic repository itself has appropriate filesystem permissions.

## 26. Local Repository vs Off-Site Backup

The current configuration is:

```text
VPS
 │
 ├── CODE
 ├── docs
 │
 └── Restic repository
       └── /home/ubuntu/restic/restic-repo
```

This protects against some forms of accidental deletion and allows historical restores.

However, if the VPS itself is destroyed, the local repository may be lost at the same time.

A stronger architecture is:

```text
                 ┌── Local Restic Repository
                 │
VPS ── Restic ───┤
                 │
                 └── Remote/Off-site Restic Repository
```

For important data, maintain at least one backup copy on separate infrastructure.

Possible remote targets include:

- Another VPS
- S3-compatible object storage
- Backblaze B2
- A NAS
- Another physical server
- Other Restic-supported remote storage

The remote repository should have its own credentials and appropriate access restrictions.

## 27. Disaster Recovery Information

For disaster recovery, preserve the following information securely:

- Restic repository location
- Restic repository password
- Source directories
- Backup script
- Cron schedule
- Retention policy
- Remote repository credentials, if applicable

The most important recovery information is the Restic repository password.

Without the repository password, an encrypted Restic repository cannot be recovered.

## 28. Current Configuration

The current working setup is:

- Host: `vps-1558ffa7`
- User: `ubuntu`
- Shell: `zsh`
- Source:
  - `/home/ubuntu/CODE`
  - `/home/ubuntu/docs`
- Repository: `/home/ubuntu/restic/restic-repo`
- Password: `/home/ubuntu/.restic_password`
- Backup script: `/home/ubuntu/scripts/restic_backup.sh`
- Log: `/home/ubuntu/scripts/backup.log`
- Cron: `0 2 * * *`
- Retention:
  - 7 daily
  - 4 weekly
  - 12 monthly

The repository has been successfully initialized, backups have been successfully created, pruning has completed successfully, and `restic check` has reported:

```text
no errors were found
```

## 29. Daily/Weekly Maintenance Checklist

### Daily

Automated:

```text
02:00
  ↓
Restic backup
  ↓
Retention/prune
  ↓
Repository check
  ↓
backup.log
```

No manual intervention should normally be required.

### Weekly

Check:

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots
```

and:

```bash
tail -100 ~/scripts/backup.log
```

Confirm recent snapshots exist and there are no errors.

### Monthly

Run:

```bash
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  check
```

Also perform a test restore to a temporary directory.

## 30. Quick Reference

The most important commands are:

```bash
# Run backup
~/scripts/restic_backup.sh

# List snapshots
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  snapshots

# Check repository
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  check

# View latest snapshot
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  ls latest

# Restore latest backup
restic -r ~/restic/restic-repo \
  --password-file ~/.restic_password \
  restore latest \
  --target ~/restic-restore

# View logs
cat ~/scripts/backup.log

# View cron
crontab -l

# Reload Zsh configuration
source ~/.zshrc
```

## Final Backup Architecture

```text
/home/ubuntu/
│
├── CODE/                         ← Data being backed up
│
├── docs/                         ← Data being backed up
│
├── restic/
│   └── restic-repo/              ← Encrypted Restic repository
│
├── scripts/
│   ├── restic_backup.sh          ← Backup automation
│   └── backup.log                ← Backup output
│
└── .restic_password              ← Repository password
                                    chmod 600
```

Automated workflow:

```text
                 Every day at 02:00
                         │
                         ▼
              restic_backup.sh
                         │
                         ▼
              Backup CODE + docs
                         │
                         ▼
                Restic snapshot
                         │
                         ▼
              Apply retention policy
            7 daily / 4 weekly / 12 monthly
                         │
                         ▼
                       Prune
                         │
                         ▼
                  restic check
                         │
                         ▼
                  backup.log
```

This is the baseline configuration to maintain on `vps-1558ffa7`.

One important change from the original documentation is adding `set -e` to the script. That prevents the script from continuing through the prune/check stages after a failed backup and then misleadingly printing `Backup process complete!`.
