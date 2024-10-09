# Guide for Creating a systemd Service to Mount an SMB Share

This guide explains how to create a systemd service on Debian that automatically mounts an SMB share, checks the connection, and restores it if disconnected. Possible causes of errors and their solutions are also covered.

## Table of Contents

1. [Step 1: Creating the systemd Service File](#step-1-creating-the-systemd-service-file)
2. [Step 2: Creating a Timer to Monitor the Connection](#step-2-creating-a-timer-to-monitor-the-connection)
3. [Step 3: Enabling and Starting the Service and Timer](#step-3-enabling-and-starting-the-service-and-timer)
4. [Step 4: Checking the Status](#step-4-checking-the-status)
5. [Guide to Securely Storing the Password](#guide-to-securely-storing-the-password)
    - [Approach 1: Using a Credentials File](#approach-1-using-a-credentials-file)
    - [Approach 2: Using systemd Secrets](#approach-2-using-systemd-secrets)
6. [Conclusion](#conclusion)

---

## Step 1: Creating the systemd Service File

Create a new systemd service file to mount the SMB share and restore the connection if needed.

Open a new file in `/etc/systemd/system`:

```bash
sudo nano /etc/systemd/system/smb-mount.service
```

Add the following content to the file:

```ini
[Unit]
Description=Mount SMB Share and Monitor Connection
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'mount -t cifs -o username=YOUR_USERNAME,password=YOUR_PASSWORD //???.???.???.???/folder /media/example/folder'
ExecStop=/bin/bash -c 'umount /media/example/folder'

[Install]
WantedBy=multi-user.target
```

### Explanation:

- **After=network-online.target** and **Wants=network-online.target**: Ensures the service is started after the network is up.
- **Type=oneshot**: The service is executed only once to mount the SMB share.
- **RemainAfterExit=yes**: Keeps the service marked as active even after the mounting is done.
- **ExecStart**: The command to mount the SMB share.
- **ExecStop**: The command to unmount the share when the service is stopped.
- **WantedBy=multi-user.target**: Ensures the service runs in multi-user mode.

---

## Step 2: Creating a Timer to Monitor the Connection

To periodically check the connection and remount the SMB share if needed, create a systemd timer.

Create a new timer file:

```bash
sudo nano /etc/systemd/system/smb-mount.timer
```

Add the following content:

```ini
[Unit]
Description=Check and Remount SMB Share

[Timer]
OnBootSec=30s
OnUnitActiveSec=1min

[Install]
WantedBy=timers.target
```

### Explanation:

- **OnBootSec=30s**: Starts the timer 30 seconds after boot.
- **OnUnitActiveSec=1min**: The timer runs every minute to check and remount the SMB share if necessary.

---

## Step 3: Enabling and Starting the Service and Timer

Reload systemd to recognize the new service and timer files:

```bash
sudo systemctl daemon-reload
```

Enable and start the service:

```bash
sudo systemctl enable smb-mount.service
sudo systemctl start smb-mount.service
```

Enable and start the timer:

```bash
sudo systemctl enable smb-mount.timer
sudo systemctl start smb-mount.timer
```

---

## Step 4: Checking the Status

You can check the status of the service and timer using the following commands:

**Service Status:**

```bash
systemctl status smb-mount.service
```

**Timer Status:**

```bash
systemctl status smb-mount.timer
```

---

## Guide to Securely Storing the Password

It’s important not to store the password in plain text within the service file. Here are two approaches to securely managing the password.

### Approach 1: Using a Credentials File

Instead of writing the password in the mount command, you can store it in a separate file and ensure only root can access it.

#### Steps:

1. **Create the credentials file:**

   ```bash
   sudo nano /root/.smbcredentials
   ```

2. **Add the following content:**

   ```ini
   username=YOUR_USERNAME
   password=YOUR_PASSWORD
   ```

3. **Set file permissions:**

   ```bash
   sudo chmod 600 /root/.smbcredentials
   ```

   This ensures only the root user has read and write access to the file.

4. **Update the mount command in the systemd service:**

   ```ini
   [Unit]
   Description=Mount SMB Share and Monitor Connection
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=oneshot
   RemainAfterExit=yes
   ExecStart=/bin/bash -c 'mount -t cifs -o credentials=/root/.smbcredentials //???.???.???.???/folder /media/example/folder'
   ExecStop=/bin/bash -c 'umount /media/example/folder'

   [Install]
   WantedBy=multi-user.target
   ```

   This replaces the password with the path to the **credentials file**. The mount command will now read the credentials from this file.

5. **Test the mount:**

   ```bash
   sudo systemctl restart smb-mount.service
   ```

---

### Approach 2: Using systemd Secrets (Supported in Newer Versions)

Systemd provides a way to securely manage sensitive data like passwords through **systemd Secrets**, but this feature is only available in newer versions.

#### Steps:

1. **Create a secret in systemd:**

   ```bash
   sudo systemctl import-environment
   sudo systemctl set-environment SMB_PASS=YOUR_PASSWORD
   ```

2. **Update the systemd service file:**

   ```ini
   [Unit]
   Description=Mount SMB Share and Monitor Connection
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=oneshot
   RemainAfterExit=yes
   Environment=SMB_PASS
   ExecStart=/bin/bash -c 'mount -t cifs -o username=YOUR_USERNAME,password=${SMB_PASS} //???.???.???.???/folder /media/example/folder'
   ExecStop=/bin/bash -c 'umount /media/example/folder'

   [Install]
   WantedBy=multi-user.target
   ```

   This uses the **SMB_PASS** environment variable to pass the password, so it is not visible in plain text in the service file.

3. **Provide the environment for systemd units:**

   Add this line to the `/etc/environment` file:

   ```bash
   SMB_PASS=YOUR_PASSWORD
   ```

   This makes the password available to systemd services without exposing it in plain text.

---

## Conclusion

- **The first approach** using a **credentials file** is the simplest and most common way to securely manage the SMB password.
- **The second approach** using **systemd Secrets** is a more advanced solution but is not always available in all distributions.

With these steps, the SMB share will be automatically mounted at boot and the connection will be periodically checked. If the connection is lost, the share will be automatically remounted.

**Note:** In all examples, replace `YOUR_PASSWORD` with your actual password, but never store it in plain text in public files.

---

**License:** This project is licensed under the MIT License – see the [LICENSE](LICENSE) file for details.
