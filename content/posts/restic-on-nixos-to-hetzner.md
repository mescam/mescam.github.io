+++
title = 'Off-site Backups from NixOS to Hetzner Storage Box with Restic'
date = 2025-11-02T16:03:57+01:00
draft = false
+++


I run a small NAS at home built on NixOS and ZFS. While ZFS gives me local redundancy and snapshots, I wanted true off-site protection in case of disasters like fire, theft, or hardware failure. After evaluating a few options, I settled on Hetzner's Storage Box: it's affordable, reliable, and supports SFTP out of the box. Using Restic for encrypted, deduplicated backups seemed like the perfect match.

I initially considered using ZFS snapshots as the backup source, but ultimately decided against it—I needed straightforward file-level backups of specific directories, and snapshots added complexity I didn't require for this use case.

***

## Prerequisites

You'll need:

- **Hetzner Storage Box** with a subaccount created:
    - Main account: `u123456`
    - Subaccount: `u123456-sub1`
    - Hostname: `u123456-sub1.your-storagebox.de`
    - SFTP/SSH runs on port 23 (not the standard 22)
- **SSH keypair** generated on your NixOS machine for passwordless authentication
- **Restic password** stored securely (we'll use `/run/keys/restic-password.txt`)
- **Directories to back up**, for example:
    - `/storage/photos`
    - `/storage/files`

***

## Step 1: Configure SSH for the Storage Box subaccount

Hetzner Storage Box uses port 23 for SFTP, and subaccounts are chrooted into their home directory. Since the NixOS Restic service runs as root by default, we'll configure SSH in root's profile.

Create or edit `/root/.ssh/config`:

```
Host u123456-sub1.your-storagebox.de
    User u123456-sub1
    Port 23
    IdentityFile /root/.ssh/id_storagebox
    StrictHostKeyChecking yes
```

**Generate an SSH key if you don't have one:**

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_storagebox -C "Backup to Hetzner"
chmod 600 /root/.ssh/id_storagebox
```

**Add the public key to your Storage Box subaccount:**

1. Log into the Hetzner Console
2. Navigate to your Storage Box
3. Select the subaccount
4. Add the contents of `/root/.ssh/id_storagebox.pub` as an authorized key (OpenSSH format for port 23)

***

## Step 2: Trust the Storage Box host key

To enable non-interactive backups via systemd, we need to trust the Storage Box's SSH host key. Connect once as root:

```bash
sudo -i
ssh -p 23 u123456-sub1@u123456-sub1.your-storagebox.de
```

Type `yes` when prompted to save the host key to `/root/.ssh/known_hosts`.

Alternatively, use `ssh-keyscan` (but verify the fingerprint matches what Hetzner provides):

```bash
ssh-keyscan -p 23 u123456-sub1.your-storagebox.de >> /root/.ssh/known_hosts
```


***

## Step 3: Create the repository directory

Subaccounts on Hetzner Storage Box are restricted to their home directory. The repository must be created inside `/home`, not at the filesystem root.

Connect to your Storage Box and create the directory:

```bash
ssh -p 23 u123456-sub1@u123456-sub1.your-storagebox.de
mkdir -p /home/nas-backup
exit
```

**Important:** Paths like `/nas-backup` (at root) will fail with `SSH_FX_FAILURE` errors. Always use `/home/<directory>` for subaccounts.

***

## Step 4: Configure Restic in NixOS

Add the following to your `configuration.nix`:

```nix
services.restic.backups.nas-backup = {
  # Repository location on Hetzner Storage Box (subaccount)
  repository = "sftp:u123456-sub1@u123456-sub1.your-storagebox.de:/home/nas-backup";
  
  # File containing your Restic repository password (not SSH password)
  passwordFile = "/run/keys/restic-password.txt";
  
  # Paths to back up (file-level, not snapshots)
  paths = [
    "/storage/photos"
    "/storage/files"
  ];
  
  # Automatically initialize the repository if it doesn't exist
  initialize = true;
  
  # Backup schedule: daily at midnight, with persistence for missed runs
  timerConfig = {
    OnCalendar = "daily";
    Persistent = true;
  };
  
  # Retention policy: keep 7 daily, 4 weekly, and 12 monthly snapshots
  pruneOpts = [
    "--keep-daily 7"
    "--keep-weekly 4"
    "--keep-monthly 12"
  ];
};
```

**Create the password file:**

```bash
echo "your-secure-restic-password" > /run/keys/restic-password.txt
chmod 600 /run/keys/restic-password.txt
```

For best practices, consider using a proper secrets management solution like `agenix` or `sops-nix`.
Make sure to keep a copy of your Restic password in a safe place, as losing it means losing access to your backups.

***

## Step 5: Apply configuration and test

Rebuild your NixOS configuration:

```bash
sudo nixos-rebuild switch
```

Start the backup service manually to test:

```bash
sudo systemctl start restic-backups-nas-backup
```
Warning: this is going to take some time, it's not asynchronous. Do not kill the process.

Monitor the logs:

```bash
journalctl -u restic-backups-nas-backup -f
```

If everything is configured correctly, you should see Restic initializing the repository (if needed) and starting the first backup.

***

## Restoring data

**List available snapshots:**

```bash
restic -r sftp:u123456-sub1@u123456-sub1.your-storagebox.de:/home/nas-backup snapshots
```

**Restore the latest snapshot:**

```bash
restic -r sftp:u123456-sub1@u123456-sub1.your-storagebox.de:/home/nas-backup restore latest --target /restore-target
```

**Restore a specific file:**

```bash
restic -r sftp:u123456-sub1@u123456-sub1.your-storagebox.de:/home/nas-backup restore latest --target /restore-target --include /storage/photos/vacation.jpg
```


***

## Common issues and solutions

**"Host key verification failed"**

- Solution: Connect manually once as root to add the host key, or use `ssh-keyscan`

**"SSH_FX_FAILURE" when creating repository**

- Solution: Ensure your repository path is under `/home` (e.g., `/home/nas-backup`), not at root

**Restic can't connect on the correct port**

- Solution: Verify `/root/.ssh/config` has `Port 23` set for the host. Restic relies on SSH config, not URL syntax

**"Permission denied" errors**

- Solution: Verify your SSH key is correctly added to the subaccount in Hetzner Console, and the private key has correct permissions (600)

**Service uses wrong SSH key**

- Solution: The NixOS Restic service runs as root by default. Keys must be in `/root/.ssh/`

***

## Why not ZFS snapshots?

ZFS snapshots provide point-in-time consistency, which is valuable for databases and other rapidly-changing data. For my use case—backing up relatively static file directories like photos and documents—file-level backups were simpler and sufficient. If you need crash-consistent backups of live databases or VMs, consider backing up from ZFS snapshot paths like `/storage/data/.zfs/snapshot/daily`.

***

## Conclusion

With this setup, I have automated, encrypted, off-site backups running daily from my NixOS NAS to a Hetzner Storage Box. The configuration is fully declarative, easy to reproduce, and leverages NixOS's built-in Restic service module. The key gotchas—port 23, host key trust, and subaccount home directory restrictions—are all handled cleanly in the SSH config and repository path.

Total setup time: about 15 minutes. Peace of mind: priceless.

