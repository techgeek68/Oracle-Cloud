# Deploy a Web Server on Oracle Linux

---

**Tenancy:** anuppyakurel | **Region:** India South (Hyderabad) | 

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Step 1: Set Up a Virtual Cloud Network (VCN)](#step-1-set-up-a-virtual-cloud-network-vcn)
3. [Step 2: Create a VM Instance](#step-2-create-a-vm-instance)
4. [Step 3: Configure Security Lists and Open Ports](#step-3-configure-security-lists-and-open-ports)
5. [Step 4: Connect via SSH](#step-4-connect-via-ssh)
6. [Step 5: Install Apache Web Server](#step-5-install-apache-web-server)
7. [Step 6: Configure the OS Firewall](#step-6-configure-the-os-firewall)
8. [Step 7: Deploy Your HTML Website](#step-7-deploy-your-html-website)
9. [Step 8: Verify Website Access](#step-8-verify-website-access)
10. [Troubleshooting](#troubleshooting)
11. [Best Practices](#best-practices)
12. [Quick Reference](#quick-reference)

---

## 1. Prerequisites

### What You Need

| Requirement | Details |
|---|---|
| OCI Account | Already active (Always Free) |
| SSH Client | Windows: PuTTY or Windows Terminal / Mac and Linux: built-in `ssh` command |
| SSH Key Pair | Generated locally before starting (instructions below) |
| Web Browser | Any modern browser |

### Generate an SSH Key Pair

Run this command on your **local machine**, not inside OCI:

```bash
ssh-keygen -t rsa -b 4096 -C "oci-webserver-key"
```

When prompted:
- **File location:** Press Enter to accept the default path, or type a custom one such as `~/.ssh/oci_key`
- **Passphrase:** Enter one for extra security, or press Enter twice to skip

Two files will be created:
- `~/.ssh/oci_key` — Your private key. Keep this secure and never share it.
- `~/.ssh/oci_key.pub` — Your public key. This gets uploaded to OCI.

Print the public key contents so you can copy them later:

```bash
cat ~/.ssh/oci_key.pub
```

The output will look something like this:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... oci-webserver-key
```

---

## Step 1: Set Up a Virtual Cloud Network (VCN)

A VCN is your private network inside OCI. Think of it as a virtual data center network you control. You need one with a public subnet before you can launch a VM that is reachable from the internet.

### 1.1 Navigate to Networking

Log in to [cloud.oracle.com](https://cloud.oracle.com). In the top-left corner, click the hamburger menu (three horizontal lines). From the menu, select **Networking**, then **Virtual Cloud Networks**.

Alternatively, from your main dashboard, look for the **Build** section and click **"Set up a network with a wizard"** to jump straight to the wizard.

### 1.2 Launch the VCN Wizard

On the Virtual Cloud Networks page, click **"Start VCN Wizard"**. When prompted to select a configuration, choose **"Create VCN with Internet Connectivity"** and click **"Start VCN Wizard"** to proceed.

This option handles the bulk of the networking configuration for you and is recommended for beginners.

### 1.3 Fill In VCN Details

| Field | Value |
|---|---|
| VCN Name | `webserver-vcn` |
| Compartment | `anuppyakurel (root)` |
| VCN IPv4 CIDR Block | `10.0.0.0/16` |
| Public Subnet IPv4 CIDR Block | `10.0.0.0/24` |
| Private Subnet IPv4 CIDR Block | `10.0.1.0/24` |

Click **Next**, review the configuration summary, then click **Create**.

The wizard provisions the following resources automatically:

| Resource | Purpose |
|---|---|
| Public Subnet (`10.0.0.0/24`) | Where your VM will live, reachable from the internet |
| Private Subnet (`10.0.1.0/24`) | For internal resources not exposed publicly |
| Internet Gateway | Enables inbound and outbound internet traffic for the public subnet |
| NAT Gateway | Allows outbound-only internet access for resources in the private subnet |
| Service Gateway | Lets your instances reach Oracle Cloud services without going over the public internet |
| Route Tables and Security List | Control traffic routing and access rules |

Click **"View VCN"** once the wizard finishes.

---

## Step 2: Create a VM Instance

### 2.1 Navigate to Compute Instances

Click the hamburger menu (☰) in the top-left corner, go to **Compute**, then select **Instances**. Click **"Create instance"**.

### 2.2 Configure the Instance

**Basic Information**

| Field | Value |
|---|---|
| Name | `webserver-vm` |
| Compartment | `anuppyakurel (root)` |
| Availability Domain | AD-1 (or whichever is available in Hyderabad) |

**Image and Shape**

Click **Edit** under "Image and shape".

For the image, click **"Change image"**, select **Oracle Linux**, and pick **Oracle Linux 8** or **Oracle Linux 9** (choose the latest available). Click **"Select image"**.

For the shape, click **"Change shape"**, select **Ampere**, then choose **VM.Standard.A1.Flex**. This is an ARM-based shape and the best Always Free option available.

Set:
- OCPUs: `1`
- Memory: `6 GB`

Click **"Select shape"**.

> **Always Free note:** The A1.Flex shape gives you a shared pool of 4 OCPUs and 24 GB RAM total, which you can distribute across up to 4 Ampere instances. The other Always Free x86 option is VM.Standard.E2.1.Micro, but it is limited to 1 OCPU and 1 GB RAM per instance, making A1.Flex the better choice for most workloads.

**Networking**

Click **Edit** under "Networking".

| Field | Value |
|---|---|
| Primary network | `webserver-vcn` |
| Subnet | `public subnet-webserver-vcn` |
| Public IPv4 address | Enable "Assign a public IPv4 address" |

Make sure the public subnet is selected. If you choose the private subnet by mistake, the instance will not get a public IP and will be unreachable from the internet.

**Add SSH Keys**

Under "Add SSH keys", select **"Paste public keys"** and paste the full contents of your `~/.ssh/oci_key.pub` file.

**Boot Volume**

The default boot volume size is 50 GB. Leave this as-is. The Always Free tier includes 200 GB of total block storage shared across all your free instances, so 50 GB per instance is well within that limit.

### 2.3 Launch the Instance

Click **Create** at the bottom of the page. The instance will pass through these states:

```
PROVISIONING  →  STARTING  →  RUNNING
```

This typically takes 2 to 4 minutes. Do not proceed until the status badge shows green **RUNNING**.

### 2.4 Record the Public IP Address

Click on **`webserver-vm`** to open the instance details page. Scroll to the **"Instance information"** tab and look under **"Primary VNIC"**. Note the **Public IP address** (for example, `129.154.X.X`). You will use this in every subsequent step.

---

## Step 3: Configure Security Lists and Open Ports

OCI Security Lists are firewall rules at the cloud network level. By default, only port 22 (SSH) is open. You need to add rules for HTTP (port 80) and HTTPS (port 443) before your web server is reachable from a browser.

### 3.1 Navigate to the Default Security List

1. Click the hamburger menu (☰), go to **Networking**, then **Virtual Cloud Networks**.
2. Click **`webserver-vcn`**.
3. Click **Subnets**, then click **`public subnet-webserver-vcn`**.
4. Click **Security Lists**, then click **`Default Security List for webserver-vcn`**.

### 3.2 Add Ingress Rules for HTTP and HTTPS

Click **"Add Ingress Rules"** and fill in the details below. You can add both rules in a single form by clicking **"+ Another Ingress Rule"** after the first one.

**Rule 1: HTTP (Port 80)**

| Field | Value |
|---|---|
| Source Type | CIDR |
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | TCP |
| Destination Port Range | `80` |
| Description | Allow HTTP traffic |

**Rule 2: HTTPS (Port 443)**

| Field | Value |
|---|---|
| Source Type | CIDR |
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | TCP |
| Destination Port Range | `443` |
| Description | Allow HTTPS traffic |

Click **"Add Ingress Rules"** to save.

### 3.3 Confirm All Required Rules Are Present

Your security list should now contain at least these three ingress rules:

| Source CIDR | Protocol | Port | Purpose |
|---|---|---|---|
| `0.0.0.0/0` | TCP | `22` | SSH |
| `0.0.0.0/0` | TCP | `80` | HTTP |
| `0.0.0.0/0` | TCP | `443` | HTTPS |

> **Security note:** Setting `0.0.0.0/0` as the SSH source allows connections from any IP on the internet. This is fine for learning, but in a production setup you should restrict the SSH rule's source CIDR to your own IP address (for example, `203.0.113.10/32`).

---

## Step 4: Connect via SSH

### 4.1 Set Correct Permissions on Your Private Key

SSH rejects private keys with overly permissive file permissions. Fix this before attempting to connect.

On Mac or Linux:
```bash
chmod 400 ~/.ssh/oci_key
```

On Windows (run PowerShell as Administrator):
```powershell
icacls "$env:USERPROFILE\.ssh\oci_key" /inheritance:r /grant:r "$env:USERNAME:R"
```

### 4.2 Connect to the VM

```bash
ssh -i ~/.ssh/oci_key opc@YOUR_PUBLIC_IP
```

> The default username for Oracle Linux instances on OCI is `opc` (Oracle Public Cloud). This account is created automatically on all Oracle Linux images, has full `sudo` privileges, and uses SSH key authentication with no password.

The first time you connect, SSH will display a fingerprint verification prompt. Type `yes` and press Enter to continue. You should land at a prompt like this:

```
[opc@webserver-vm ~]$
```

### 4.3 Update the System

Always update packages before installing anything new:

```bash
sudo dnf update -y
```

This may take a few minutes depending on how many updates are pending.

### 4.4 Verify the OS Version

Confirm you are running Oracle Linux as expected:

```bash
cat /etc/oracle-release
```

Expected output:
```
Oracle Linux Server release 8.x
```

---

## Step 5: Install Apache Web Server

### 5.1 Install Apache

```bash
sudo dnf install httpd -y
```

### 5.2 Start Apache and Enable It at Boot

```bash
sudo systemctl enable --now httpd
```

This single command both starts Apache immediately and configures it to start automatically whenever the VM reboots.

### 5.3 Verify Apache Is Running

```bash
sudo systemctl status httpd
```

Look for the `Active:` line in the output. It should read:

```
Active: active (running) since ...
```

### 5.4 Test Apache Locally on the VM

```bash
curl -I http://localhost
```

Expected response:

```
HTTP/1.1 200 OK
Server: Apache/2.4.xx (Oracle Linux)
...
```

A `200 OK` here means Apache is working before you open it to the internet.

---

## Step 6: Configure the OS Firewall

An OCI instance has two independent firewall layers. The first is the OCI Security List you configured in Step 3, which operates at the cloud network level. The second is `firewalld`, running inside the operating system. Both layers must allow a port for traffic to reach your web server.

### 6.1 Confirm the Firewall Is Running

```bash
sudo systemctl status firewalld
```

### 6.2 Open HTTP and HTTPS in the OS Firewall

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### 6.3 Verify the Rules Took Effect

```bash
sudo firewall-cmd --list-all
```

The output should include `http` and `https` in the services line:

```
services: cockpit dhcpv6-client http https ssh
```

---

## Step 7: Deploy Your HTML Website

### 7.1 About the Apache Document Root

Apache serves files from `/var/www/html/` by default. Any file placed in that directory is accessible via your public IP address.

```bash
ls -la /var/www/html
```

The directory should be empty on a fresh installation.

### 7.2 Create the Website Directory Structure

```bash
sudo mkdir -p /var/www/html/assets/css
sudo mkdir -p /var/www/html/assets/images
```

### 7.3 Check SELinux Status

Oracle Linux enforces SELinux by default. Files with the wrong SELinux context will be blocked from being served by Apache even if the file permissions look correct.

```bash
sestatus
```

You should see both `SELinux status: enabled` and `Current mode: enforcing`. If this is the case, the `restorecon` step in 7.5 is important and must not be skipped.

### 7.4 Create the Main HTML Page

```bash
sudo nano /var/www/html/index.html
```

Paste the following HTML:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to My OCI Web Server</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            color: #ffffff;
        }
        .hero { text-align: center; padding: 60px 20px 40px; }
        .oracle-badge {
            display: inline-block;
            background: linear-gradient(90deg, #c74634, #f80000);
            color: white;
            font-size: 0.85rem;
            font-weight: 700;
            letter-spacing: 2px;
            text-transform: uppercase;
            padding: 8px 24px;
            border-radius: 30px;
            margin-bottom: 30px;
        }
        h1 {
            font-size: 3.5rem;
            font-weight: 800;
            margin-bottom: 20px;
            background: linear-gradient(90deg, #ffffff, #a8d8ff);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }
        .subtitle {
            font-size: 1.2rem;
            color: #a0b9d9;
            max-width: 600px;
            margin: 0 auto 40px;
            line-height: 1.7;
        }
        .status-card {
            background: rgba(255,255,255,0.07);
            border: 1px solid rgba(255,255,255,0.15);
            border-radius: 16px;
            padding: 24px 40px;
            margin-bottom: 50px;
            display: flex;
            align-items: center;
            gap: 16px;
        }
        .status-dot {
            width: 14px;
            height: 14px;
            background: #00e676;
            border-radius: 50%;
            animation: pulse 2s infinite;
            flex-shrink: 0;
        }
        @keyframes pulse {
            0%, 100% { box-shadow: 0 0 0 0 rgba(0,230,118,0.4); }
            50% { box-shadow: 0 0 0 10px rgba(0,230,118,0); }
        }
        .status-text { font-size: 1rem; color: #d0e8ff; }
        .status-text strong { color: #00e676; }
        .info-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
            gap: 20px;
            max-width: 900px;
            width: 100%;
            padding: 0 20px;
            margin-bottom: 50px;
        }
        .info-card {
            background: rgba(255,255,255,0.06);
            border: 1px solid rgba(255,255,255,0.12);
            border-radius: 14px;
            padding: 28px 24px;
            text-align: center;
            transition: transform 0.3s ease;
        }
        .info-card:hover { transform: translateY(-6px); }
        .info-card .icon { font-size: 2.5rem; margin-bottom: 14px; }
        .info-card h3 {
            font-size: 0.8rem;
            text-transform: uppercase;
            letter-spacing: 1.5px;
            color: #8ab4d4;
            margin-bottom: 10px;
        }
        .info-card p { font-size: 1rem; font-weight: 600; color: #e0f0ff; }
        .tech-stack {
            display: flex;
            gap: 12px;
            flex-wrap: wrap;
            justify-content: center;
            margin-bottom: 50px;
        }
        .tech-badge {
            background: rgba(255,255,255,0.1);
            border: 1px solid rgba(255,255,255,0.2);
            padding: 8px 18px;
            border-radius: 20px;
            font-size: 0.85rem;
            font-weight: 600;
            color: #c9e0ff;
        }
        footer {
            text-align: center;
            padding: 30px 20px;
            color: #506070;
            font-size: 0.85rem;
        }
        footer a { color: #6ea8d8; text-decoration: none; }
        @media (max-width: 600px) {
            h1 { font-size: 2.2rem; }
            .status-card { flex-direction: column; text-align: center; padding: 20px; }
        }
    </style>
</head>
<body>
    <section class="hero">
        <div class="oracle-badge">Oracle Cloud Infrastructure</div>
        <h1>Welcome to My OCI Web Server</h1>
        <p class="subtitle">
            Your web server is live on Oracle Cloud Infrastructure,
            running Oracle Linux with Apache HTTP Server.
        </p>
    </section>

    <div class="status-card">
        <div class="status-dot"></div>
        <div class="status-text">
            Status: <strong>Online</strong> &nbsp;|&nbsp;
            Region: <strong>India South (Hyderabad)</strong> &nbsp;|&nbsp;
            Web Server: <strong>Apache HTTP Server</strong>
        </div>
    </div>

    <div class="info-grid">
        <div class="info-card">
            <div class="icon">🖥️</div>
            <h3>Operating System</h3>
            <p>Oracle Linux 8</p>
        </div>
        <div class="info-card">
            <div class="icon">⚡</div>
            <h3>Instance Shape</h3>
            <p>VM.Standard.A1.Flex</p>
        </div>
        <div class="info-card">
            <div class="icon">🌍</div>
            <h3>Cloud Region</h3>
            <p>ap-hyderabad-1</p>
        </div>
        <div class="info-card">
            <div class="icon">🛡️</div>
            <h3>Account Type</h3>
            <p>Always Free Tier</p>
        </div>
        <div class="info-card">
            <div class="icon">🌐</div>
            <h3>Web Server</h3>
            <p>Apache (httpd)</p>
        </div>
        <div class="info-card">
            <div class="icon">🔒</div>
            <h3>SSH Access</h3>
            <p>Key-Based Auth</p>
        </div>
    </div>

    <div class="tech-stack">
        <span class="tech-badge">Oracle Linux</span>
        <span class="tech-badge">Apache HTTP Server</span>
        <span class="tech-badge">OCI Compute</span>
        <span class="tech-badge">OCI VCN</span>
        <span class="tech-badge">SSH Key Auth</span>
        <span class="tech-badge">firewalld</span>
    </div>

    <footer>
        <p>
            Deployed on
            <a href="https://cloud.oracle.com" target="_blank">Oracle Cloud Infrastructure</a>
            &nbsp;|&nbsp; Tenancy: <strong>anuppyakurel</strong>
        </p>
        <p style="margin-top: 8px;">Deployed: June 2026</p>
    </footer>
</body>
</html>
```

Save and exit nano:
- Press `Ctrl + X`
- Press `Y` to confirm saving
- Press `Enter` to keep the filename

### 7.5 Set File Permissions and Fix the SELinux Context

Set the correct read permissions on the web directory:

```bash
sudo find /var/www/html/ -type d -exec chmod 755 {} \;
sudo find /var/www/html/ -type f -exec chmod 644 {} \;
```

> **Note on ownership:** The `/var/www/html` directory is owned by `root` by default, and Apache runs as the `apache` user. For serving static HTML files, Apache only needs read access, which the 644/755 permissions above provide. Changing ownership to `apache` is only necessary if Apache needs to write to the directory (for example, for CMS uploads).

Now restore the correct SELinux context on your web files. The default context for files under `/var/www/html` is `httpd_sys_content_t`, which allows Apache to read them. The `restorecon` command applies this automatically:

```bash
sudo restorecon -Rv /var/www/html/
```

Confirm the context is correctly applied:

```bash
ls -Z /var/www/html/index.html
```

The output should include `httpd_sys_content_t`:

```
system_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html
```

### 7.6 Test the Apache Configuration

Before reloading Apache, run a syntax check on its configuration files:

```bash
sudo apachectl configtest
```

Expected output:

```
Syntax OK
```

Do not reload if you see any errors. Fix them first by reviewing `/etc/httpd/conf/httpd.conf` or the relevant virtual host file.

### 7.7 Reload Apache

```bash
sudo systemctl reload httpd
```

This applies changes gracefully without dropping active connections.

---

## Step 8: Verify Website Access

### 8.1 Test Locally on the VM

```bash
curl -s http://localhost | grep -o "<title>.*</title>"
```

Expected output:

```html
<title>Welcome to My OCI Web Server</title>
```

### 8.2 Open the Website in a Browser

On your local machine, open a browser and navigate to:

```
http://YOUR_PUBLIC_IP
```

For example: `http://129.154.X.X`

You should see your welcome page with the gradient background, status indicator, and server detail cards.

### 8.3 Test from the Command Line on Your Local Machine

```bash
curl -I http://YOUR_PUBLIC_IP
```

Expected output:

```
HTTP/1.1 200 OK
Date: ...
Server: Apache/2.4.xx (Oracle Linux)
Content-Type: text/html; charset=UTF-8
```

### 8.4 Look Up the Public IP from Inside the VM

If you need to check the instance's public IP while logged in, query the OCI Instance Metadata Service (IMDS). This is a link-local endpoint (`169.254.169.254`) accessible only from within the instance, provided by Oracle's metadata service (IMDSv1):

```bash
curl -s http://169.254.169.254/opc/v1/vnics/ | \
  python3 -c "import sys, json; data=json.load(sys.stdin); print(data[0].get('publicIp', 'Not assigned'))"
```

---

## Troubleshooting

### Browser shows "Connection timed out" when accessing the website

Work through these checks in order:

```bash
# Confirm Apache is running
sudo systemctl status httpd

# Confirm port 80 is actively listening
sudo ss -tlnp | grep :80

# Confirm the OS firewall has HTTP open
sudo firewall-cmd --list-services
```

If the OS-level checks look fine, go back to the OCI Console and verify:
- The Security List for your public subnet has an ingress rule allowing TCP port 80 from `0.0.0.0/0`
- The instance is in the **public subnet**, not the private subnet
- A public IPv4 address is assigned to the instance

### SSH returns "Permission denied (publickey)"

```bash
# Check the permissions on your local private key file
ls -la ~/.ssh/oci_key

# Fix permissions if needed
chmod 400 ~/.ssh/oci_key

# Run SSH with verbose output to trace the exact failure
ssh -v -i ~/.ssh/oci_key opc@YOUR_PUBLIC_IP
```

The most common causes are using the wrong username (it must be `opc` for Oracle Linux), pointing to the wrong key file, or having key permissions too open (644 instead of 400).

### Apache returns "403 Forbidden"

```bash
# Check the SELinux context on the file
ls -Z /var/www/html/index.html

# Restore the correct SELinux context if it is wrong
sudo restorecon -Rv /var/www/html/

# Check that file permissions allow reading
ls -la /var/www/html/index.html
# Should show: -rw-r--r-- (644)

# Look at Apache's error log for a specific cause
sudo tail -50 /var/log/httpd/error_log
```

### Apache will not start

```bash
# View the systemd error output
sudo journalctl -xe | grep httpd

# Test the Apache config for syntax problems
sudo apachectl configtest

# Check if another process is already using port 80
sudo ss -tlnp | grep :80
```

### Website shows old content after editing index.html

Do a hard browser refresh: `Ctrl + Shift + R` on Windows/Linux, or `Cmd + Shift + R` on Mac. If the content still looks stale, reload Apache:

```bash
sudo systemctl reload httpd
```

---

## Best Practices

### Security

Restrict SSH access in the OCI Security List to your own IP address rather than leaving it open to `0.0.0.0/0`. Change the SSH ingress rule source CIDR from `0.0.0.0/0` to `YOUR.IP.ADDRESS/32`.

```bash
# Keep the system up to date
sudo dnf update -y

# Review failed SSH login attempts regularly
sudo journalctl -u sshd | grep "Failed password" | tail -20

# List all listening ports to audit what is exposed
sudo ss -tlnp
```

### Monitoring

```bash
# Watch incoming requests in real time
sudo tail -f /var/log/httpd/access_log

# Watch errors in real time
sudo tail -f /var/log/httpd/error_log

# Check system resource usage
top

# Install htop for a more readable interface
sudo dnf install htop -y

# Check disk space
df -h
```

### Backup

```bash
# Create a compressed, timestamped backup of all website files
sudo tar -czf /home/opc/website_backup_$(date +%Y%m%d).tar.gz /var/www/html/

# Confirm the backup file was created
ls -lh /home/opc/website_backup_*.tar.gz
```

### Apache Management Reference

```bash
sudo systemctl start httpd       # Start the service
sudo systemctl stop httpd        # Stop the service
sudo systemctl restart httpd     # Full restart (brief downtime)
sudo systemctl reload httpd      # Graceful reload without downtime
sudo apachectl configtest        # Check configuration syntax before reloading
```

---
