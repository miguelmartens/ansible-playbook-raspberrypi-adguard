## Summary

This guide explains how to provision and configure a Raspberry Pi 4 Model B headlessly, with all heavy I/O (logs, AdGuard Home data) offloaded to a USB‑SSD, unattended OS and AdGuard updates, and secure networking. It covers two control-machine scenarios:

* **Linux (Ubuntu/Debian)**: native Ansible installation.
* **Windows**: using Windows Subsystem for Linux (WSL) to run Ansible.

Follow the steps in your preferred environment to clone the project, prepare SSH keys, flash the OS, and execute the Ansible playbook.

---

## Prerequisites

### Raspberry Pi Preparation

Your friend will need to flash Raspberry Pi OS Lite onto the microSD card and preconfigure headless SSH (and optionally Wi‑Fi) before inserting it into the Pi:

1. **Download Raspberry Pi Imager**

   * Go to the official downloads page: [https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/)
   * Choose "Raspberry Pi Imager" for your OS.

2. **Flash OS with Headless Configuration**

   * Insert the microSD card into your control machine.
   * Open Raspberry Pi Imager.
   * Under **Raspberry Pi Device**, select **Raspberry Pi 4**.
   * Under **Operating System**, navigate to **Raspberry Pi OS (other)** → **Raspberry Pi OS Lite (64-bit)**.
   * Click **Next**, then click **Edit Settings**:

     * **Set Username**: `pi`.
     * **Set Password**: choose a secure password.
     * **Set Hostname**: `raspberrypi`.
   * Switch to the **Services** tab and **enable SSH**.
   * Click **Save**, then **Yes** to confirm settings, then **Yes** to write the image.
   * After writing, confirm an empty file named `ssh` exists in the **boot** partition (Imager will do this for you).

3. **First Boot**

   * Insert the flashed microSD into the Pi, power it on.
   * Wait ~1 minute for SSH (and Wi‑Fi if configured) to come up.

### On the Raspberry Pi

* A Raspberry Pi 4 Model B with the flashed microSD card.
* USB‑SSD connected for AdGuard data.
* Power supply and DHCP‑enabled network.

### On the Control Machine

* Git installed.
* SSH client available.

---

## Linux Setup (Ubuntu/Debian)

1. **Install dependencies**

   ```bash
   sudo apt update
   sudo add-apt-repository --yes --update ppa:ansible/ansible
   sudo apt install -y ansible
   ```

2. **SSH Key Preparation**

   First, generate a new SSH key pair:

   ```bash
   ssh-keygen -t ed25519 -f ssh/id_ed25519 -C "pi@raspberrypi"
   ```

   This will create:
   - A private key at `ssh/id_ed25519`
   - A public key at `ssh/id_ed25519.pub`

   To install the public key on the Pi:

   ```bash
   ssh-copy-id -i ssh/id_ed25519.pub pi@raspberrypi.local
   ```

   Enter the password of the "pi" user when prompted.

3. **Run the Playbook**

   Ensure you are in the project root (where `site.yml`, `hosts.ini`, and `ansible.cfg` live). Then execute:

   ```bash
   ansible-playbook -i hosts.ini site.yml
   ```

   * **Inventory**: `hosts.ini` defines your Pi's address and user.
   * **Config**: `ansible.cfg` points to the private key and disables host key checking.
   * **Playbook**: `site.yml` runs all roles in sequence.

   You can verify progress in the output; look for tasks like `Partition SSD`, `Install AdGuard Home`, and `Enable UFW`.

---

## Windows Setup (WSL)

1. **Enable WSL and install Ubuntu**

   * Open PowerShell as Administrator:

     ```powershell
     wsl --install -d Ubuntu
     ```
   * Restart if prompted.

2. **Inside WSL (Ubuntu)**

   Follow the same **Linux Setup** steps above, starting with `sudo apt update`.

---

## What the Playbook Does

* **Partitions & Formats** the USB‑SSD for AdGuard data and configuration.
* **Enables unattended‑upgrades** for OS patches.
* **Installs AdGuard Home** with:
  * Automatic updates every Sunday at 3 AM
  * Update logs in `/var/log/adguard-update.log`
  * Data and configuration stored on SSD
  * Web interface on port 3000
  * DNS service on port 53
* **Hardens SSH** with:
  * Disabled root login
  * Strong cipher configurations
  * Connection timeouts and limits
  * Proper logging
* **Configures UFW** to allow:
  * SSH (port 22)
  * DNS (port 53)
  * AdGuard Web UI (port 3000)
  * HTTP/HTTPS (ports 80/443)

---

## Accessing the Pi

* **SSH**: `ssh pi@raspberrypi.local`
* **AdGuard UI**: `http://raspberrypi.local:3000`
* **DNS**: point your client's DNS to the Pi's IP (DHCP assigned).

---

## Troubleshooting

* If `raspberrypi.local` doesn't resolve, add to `/etc/hosts`:

  ```
  <IP ADDRESS> raspberrypi.local
  ```
* Ensure the Pi's SSH public key is installed: the playbook's `pre_tasks` handle this automatically.
* If AdGuard Home fails to start, check:
  * Port 53 is not in use: `sudo lsof -i :53`
  * Service logs: `sudo journalctl -u adguardhome`
  * Update logs: `cat /var/log/adguard-update.log`

---

## Maintenance

* **Backups**: snapshot `/mnt/ssd/adguard/data` and `/mnt/ssd/adguard/config` regularly.
* **Updates**: AdGuard Home updates automatically every Sunday at 3 AM.
* **Manual Updates**: run `/usr/local/bin/update-adguard.sh` to update manually.
* **Logs**: check `/var/log/adguard-update.log` for update history.

Enjoy your self‑maintaining, headless Raspberry Pi with network‑wide ad blocking!
