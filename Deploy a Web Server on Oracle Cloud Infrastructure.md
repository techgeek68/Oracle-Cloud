# Deploy a Web Server on Oracle Cloud Infrastructure

---

**Tenancy:** <orcle_cloud_username> | **Region:** India South (Hyderabad) | 

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
11. [Recommendations](#recommendations)

---

## 1. Prerequisites

### What You Need

| Requirement | Details |
|---|---|
| OCI Account | Already active (Always Free) |
| SSH Client | Windows: PuTTY or Windows PowerShell / Mac and Linux: built-in `ssh` command |
| SSH Key Pair | Generated locally before starting (instructions below) |
| Web Browser | Any modern browser |

---

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
cat ~/.ssh/<Key_Name>.pub
```

The output will look something like this:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... oci-webserver-key
```

---

## Step 1: Set Up a Virtual Cloud Network (VCN)

A VCN is your private network inside OCI. Think of it as a virtual data center network you control. You need one with a public subnet before you can launch a VM that is reachable from the internet.


### 1.1 Navigate to Networking

Log in to [cloud.oracle.com](https://cloud.oracle.com). In the top left corner, click the hamburger menu (three horizontal lines). From the menu, select **Networking**, then **Virtual Cloud Networks**.

>Alternatively, from your main dashboard, look for the **Build** section and click **"Set up a network with a wizard"** to jump straight to the wizard.


### 1.2 Launch the VCN Wizard

On the Virtual Cloud Networks page, click **Action** > **"Start VCN Wizard"**. When prompted to select a configuration, choose **"Create VCN with Internet Connectivity"** and click **"Start VCN Wizard"** to proceed.

>This option handles the bulk of the networking configuration for you and is recommended for beginners.

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
<img width="1470" height="712" alt="Screenshot 2026-06-13 at 7 33 08 PM" src="https://github.com/user-attachments/assets/d1de5881-be62-4ed0-a235-636414104298" />

---

## Step 2: Create a VM Instance

### 2.1 Navigate to Compute Instances

Click the hamburger menu (☰) in the top left corner, go to **Compute**, then select **Instances**. Click **"Create instance"**.

### 2.2 Configure the Instance

**Basic Information**

| Field | Value |
|---|---|
| Name | `webserver-vm` |
| Compartment | `<orcle_cloud_username> (root)` |
| Availability Domain | AD-1 (or whichever is available in Hyderabad) |

**Image and Shape**

Click **Edit** under "Image and shape".

For the image, click **"Change image"**, select **CentOS 8 Stream**, and pick **Oracle Linux. Click **"Select image"**.

> If you want to change the shape, click **"Change shape"**, select compatible shape and click **"Select shape"**.

> **Always Free note:** for example The A1.Flex shape gives you a shared pool of 4 OCPUs and 24 GB RAM total, which you can distribute across up to 4 Ampere instances. The other Always Free x86 option is VM.Standard.E2.1.Micro, but it is limited to 1 OCPU and 1 GB RAM per instance, making A1.Flex the better choice for most workloads.

Click on **Next**

**Networking**

| Field | Value |
|---|---|
| Primary network | Select existing virtual cloud network `webserver-vcn` |
| Subnet | Select existing subnet `public subnet-webserver-vcn` |
| Public IPv4 address assignment| Enable "Assign a public IPv4 address" |

> Make sure the public subnet is selected. If you choose the private subnet by mistake, the instance will not get a public IP and will be unreachable from the internet.

**Add SSH Keys**

Under "Add SSH keys", select **"Paste public keys"** and paste the full contents of your `~/.ssh/oci_key.pub` file.

Click on **Next**

**Boot Volume**

Specify a custom boot volume size and performance setting > Enable

The default boot volume size is 50 GB. Leave this as is. The Always Free tier includes 200 GB of total block storage shared across all your free instances, so 50 GB per instance is well within that limit.

### 2.3 Review and Launch the Instance

Click **Create** at the bottom of the page. The instance will pass through these states:

```
PROVISIONING  >  STARTING  > RUNNING
```

> This typically takes 2 to 4 minutes. Do not proceed until the status badge shows green **RUNNING**.

---
<img width="1467" height="743" alt="Screenshot 2026-06-13 at 8 00 43 PM" src="https://github.com/user-attachments/assets/9367d595-07db-4da3-b1dc-d05a688fba4f" />

---
<img width="1463" height="717" alt="Screenshot 2026-06-13 at 8 01 25 PM" src="https://github.com/user-attachments/assets/7e5fd860-1170-4caa-bb42-8e8e5ca774b4" />

---

### 2.4 Record the Public IP Address

Click on **`webserver-vm`** to open the instance details page. Select the **Networking** tab and look under **"Primary VNIC"**. Note the **Public IP address** (for example, `129.154.X.X`). You will use this in every subsequent step.

---

## Step 3: Configure Security Lists and Open Ports

OCI Security Lists are firewall rules at the cloud network level. By default, only port 22 (SSH) is open. You need to add rules for HTTP (port 80) or HTTPS (port 443) before your web server is reachable from a browser.

### 3.1 Navigate to the Default Security List

1. Click the hamburger menu (☰), go to **Networking**, then **Virtual Cloud Networks**.
2. Click **`webserver-vcn`**.
3. Click **Subnets**, then click **`public subnet-webserver-vcn`**.
4. Click **Security Lists**, then click **`Default Security List for webserver-vcn`** and Select **Security rules**

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

or 

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

> **Security note:** Setting `0.0.0.0/0` as the SSH source allows connections from any IP on the internet. This is fine for learning, but in a production setup, you should restrict the SSH rule's source CIDR to your own IP address (for example, `203.0.113.10/32`).
---
<img width="1468" height="718" alt="Screenshot 2026-06-13 at 8 10 24 PM" src="https://github.com/user-attachments/assets/be47a6c2-6c20-4916-b542-6709a7681c5c" />

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

---
<img width="1151" height="263" alt="Screenshot 2026-06-13 at 8 13 26 PM" src="https://github.com/user-attachments/assets/738df1ce-4a42-4abe-939a-347777fde431" />

---

### 4.3 Update the System

Always update packages before installing anything new:

```bash
sudo dnf update -y
```

This may take a few minutes, depending on how many updates are pending.

### 4.4 Verify the OS Version

Confirm you are running Oracle Linux as expected:

```bash
cat /etc/os-release
```
---
> If the repository didn't work, follow the method below

> CentOS Stream 8 hit End of Life on May 31, 2024. It receives no further security patches. Do not use this for anything production-facing. Recreating the instance with Oracle Linux 9 is the proper fix.

Step 1: Back Up the Repo Files

```bash
sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.backup
```

Step 2: Redirect All Repos to the CentOS Vault Archive

```bash
sudo sed -i \
  -e 's|^mirrorlist=|#mirrorlist=|g' \
  -e 's|^#baseurl=|baseurl=|g' \
  -e 's|mirror.centos.org/\$contentdir|vault.centos.org|g' \
  -e 's|\$stream|CentOS-Stream8|g' \
  /etc/yum.repos.d/*.repo
```

Step 3: Clean Cache and Test

```bash
sudo dnf clean all
sudo dnf makecache
```

If `makecache` completes without errors, run the update:

```bash
sudo dnf update -y
```
---
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

### 5.4 Test Apache Locally on the VM

```bash
curl -I http://localhost
```

Expected response:

```
HTTP/1.1 200 OK
Date: Sat, 13 Jun 2026 15:00:46 GMT
Server: Apache/2.4.37 (CentOS Stream)
Last-Modified: Sat, 13 Jun 2026 14:56:15 GMT
ETag: "1980-65423cb7fd41a"
Accept-Ranges: bytes
Content-Length: 6528
Content-Type: text/html; charset=UTF-8
```

---

>A `200 OK` here means Apache is working before you open it to the internet.

>A 403 Forbidden error confirms Apache is running and responding, but it cannot serve the requested content. The cause can be incorrect file or directory permissions, a wrong SELinux security context, or a missing index file with directory listing disabled. Check the Apache error log first to identify which one applies before making any changes.

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
    <title>My OCI Web Server</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', sans-serif;
            background-color: #0d1117;
            color: #e6edf3;
            min-height: 100vh;
        }

        header {
            background-color: #161b22;
            border-bottom: 1px solid #30363d;
            padding: 18px 40px;
            display: flex;
            align-items: center;
            justify-content: space-between;
        }

        header .logo {
            font-size: 1rem;
            font-weight: 600;
            color: #e6edf3;
            letter-spacing: 0.3px;
        }

        header .logo span {
            color: #f05a2a;
        }

        header nav a {
            color: #8b949e;
            text-decoration: none;
            font-size: 0.875rem;
            margin-left: 24px;
        }

        header nav a:hover {
            color: #e6edf3;
        }

        .hero {
            max-width: 760px;
            margin: 80px auto 60px;
            padding: 0 24px;
        }

        .tag {
            display: inline-block;
            font-size: 0.75rem;
            font-weight: 600;
            text-transform: uppercase;
            letter-spacing: 1px;
            color: #f05a2a;
            border: 1px solid #f05a2a;
            border-radius: 4px;
            padding: 3px 10px;
            margin-bottom: 20px;
        }

        .hero h1 {
            font-size: 2.4rem;
            font-weight: 700;
            line-height: 1.3;
            color: #e6edf3;
            margin-bottom: 16px;
        }

        .hero p {
            font-size: 1rem;
            color: #8b949e;
            line-height: 1.8;
            max-width: 580px;
        }

        .status-bar {
            max-width: 760px;
            margin: 0 auto 60px;
            padding: 0 24px;
        }

        .status-bar .inner {
            background-color: #161b22;
            border: 1px solid #30363d;
            border-radius: 6px;
            padding: 14px 20px;
            display: flex;
            align-items: center;
            gap: 10px;
            font-size: 0.875rem;
            color: #8b949e;
        }

        .status-bar .dot {
            width: 8px;
            height: 8px;
            border-radius: 50%;
            background-color: #3fb950;
            flex-shrink: 0;
        }

        .status-bar .inner b {
            color: #3fb950;
            font-weight: 600;
        }

        .status-bar .inner .divider {
            color: #30363d;
            margin: 0 6px;
        }

        .section {
            max-width: 760px;
            margin: 0 auto 60px;
            padding: 0 24px;
        }

        .section h2 {
            font-size: 0.7rem;
            font-weight: 600;
            text-transform: uppercase;
            letter-spacing: 1.2px;
            color: #8b949e;
            margin-bottom: 16px;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            font-size: 0.875rem;
        }

        table tr {
            border-bottom: 1px solid #21262d;
        }

        table tr:last-child {
            border-bottom: none;
        }

        table td {
            padding: 12px 16px;
            background-color: #161b22;
        }

        table td:first-child {
            color: #8b949e;
            width: 40%;
            border-right: 1px solid #21262d;
        }

        table td:last-child {
            color: #e6edf3;
            font-weight: 500;
        }

        table tr:first-child td:first-child { border-radius: 6px 0 0 0; }
        table tr:first-child td:last-child  { border-radius: 0 6px 0 0; }
        table tr:last-child td:first-child  { border-radius: 0 0 0 6px; }
        table tr:last-child td:last-child   { border-radius: 0 0 6px 0; }

        .notice {
            max-width: 760px;
            margin: 0 auto 60px;
            padding: 0 24px;
        }

        .notice .inner {
            background-color: #1f1610;
            border: 1px solid #6e3a1e;
            border-radius: 6px;
            padding: 14px 20px;
            font-size: 0.875rem;
            color: #c9742a;
            line-height: 1.7;
        }

        .notice .inner b {
            color: #e6825a;
        }

        .notice .inner a {
            color: #e6825a;
            text-decoration: underline;
        }

        .tags-row {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
        }

        .badge {
            font-size: 0.8rem;
            color: #8b949e;
            background-color: #161b22;
            border: 1px solid #30363d;
            border-radius: 4px;
            padding: 4px 12px;
        }

        footer {
            border-top: 1px solid #21262d;
            padding: 24px 40px;
            font-size: 0.8rem;
            color: #484f58;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        footer a {
            color: #484f58;
            text-decoration: none;
        }

        footer a:hover {
            color: #8b949e;
        }

        @media (max-width: 600px) {
            header { padding: 16px 20px; }
            header nav { display: none; }
            .hero, .status-bar, .section, .notice { padding: 0 16px; }
            .hero h1 { font-size: 1.8rem; }
            .status-bar .inner { flex-direction: column; align-items: flex-start; gap: 6px; }
            footer { flex-direction: column; gap: 8px; text-align: center; }
        }
    </style>
</head>
<body>

    <header>
        <div class="logo">Oracle <span>Cloud</span> &mdash; webserver-vm</div>
        <nav>
            <a href="https://cloud.oracle.com" target="_blank">Console</a>
            <a href="https://docs.oracle.com" target="_blank">Docs</a>
        </nav>
    </header>

    <div class="hero">
        <div class="tag">Oracle Cloud Infrastructure</div>
        <h1>Welcome to My OCI Web Server</h1>
        <p>This instance is running on Oracle Cloud Infrastructure in the India South (Hyderabad) region. Apache HTTP Server is installed and serving this page over port 80.</p>
    </div>

    <div class="status-bar">
        <div class="inner">
            <div class="dot"></div>
            <span>Server status: <b>Online</b></span>
            <span class="divider">|</span>
            <span>Region: ap-hyderabad-1</span>
            <span class="divider">|</span>
            <span>Web server: Apache/2.4.37</span>
        </div>
    </div>

    <div class="notice">
        <div class="inner">
            <b>Note:</b> This server is currently running <b>CentOS Stream 8</b>, which reached
            End of Life on May 31, 2024. It is recommended to migrate to
            <a href="https://docs.oracle.com/en/operating-systems/oracle-linux/" target="_blank">Oracle Linux 9</a>
            as soon as possible to continue receiving security updates.
        </div>
    </div>

    <div class="section">
        <h2>Instance Details</h2>
        <table>
            <tr>
                <td>Operating System</td>
                <td>CentOS Stream 8 (EOL)</td>
            </tr>
            <tr>
                <td>Instance Shape</td>
                <td>VM.Standard.A1.Flex (1 OCPU, 6 GB RAM)</td>
            </tr>
            <tr>
                <td>Cloud Region</td>
                <td>India South &mdash; ap-hyderabad-1</td>
            </tr>
            <tr>
                <td>Account Type</td>
                <td>Always Free</td>
            </tr>
            <tr>
                <td>Web Server</td>
                <td>Apache/2.4.37 (httpd)</td>
            </tr>
            <tr>
                <td>SSH Access</td>
                <td>Key-based authentication (RSA 4096)</td>
            </tr>
            <tr>
                <td>Tenancy</td>
                <td>anuppyakurel</td>
            </tr>
        </table>
    </div>

    <div class="section">
        <h2>Stack</h2>
        <div class="tags-row">
            <span class="badge">CentOS Stream 8</span>
            <span class="badge">Apache/2.4.37</span>
            <span class="badge">OCI Compute</span>
            <span class="badge">OCI VCN</span>
            <span class="badge">firewalld</span>
            <span class="badge">SELinux</span>
            <span class="badge">SSH Key Auth</span>
        </div>
    </div>

    <footer>
        <span>Deployed on <a href="https://cloud.oracle.com" target="_blank">Oracle Cloud Infrastructure</a></span>
        <span>June 2026</span>
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

```bash
sudo chown -R apache:apache /var/www/html
```

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

---
<img width="1468" height="118" alt="Screenshot 2026-06-13 at 8 55 02 PM" src="https://github.com/user-attachments/assets/1d2edf1e-af07-4be8-83d3-2287a7ae3d09" />

---

### 8.2 Open the Website in a Browser

On your local machine, open a browser and navigate to:

```
http://YOUR_PUBLIC_IP
```

For example: `http://129.154.X.X`

---
<img width="1470" height="926" alt="Screenshot 2026-06-13 at 9 11 30 PM" src="https://github.com/user-attachments/assets/1f13bd19-a59a-4099-989d-2f590eef0f11" />

---

### 8.3 Test from the Command Line on Your Local Machine

```bash
curl -I http://YOUR_PUBLIC_IP
```

Expected output:

```
HTTP/1.1 200 OK
Date: Sat, 13 Jun 2026 15:28:55 GMT
Server: Apache/2.4.37 (CentOS Stream)
Last-Modified: Sat, 13 Jun 2026 15:24:50 GMT
ETag: "22a0-6542431bffbef"
Accept-Ranges: bytes
Content-Length: 8864
Content-Type: text/html; charset=UTF-8
```

---

## Troubleshooting

### Browser shows "Connection timed out" when accessing the website

Work through these checks in order:

```bash
sudo systemctl status httpd                            # Confirm Apache is running


sudo ss -tlnp | grep :80                                # Confirm port 80 is actively listening


sudo firewall-cmd --list-services                        # Confirm the OS firewall has HTTP open
```

If the OS level checks look fine, go back to the OCI Console and verify:
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

## Recommendations

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

---
## Cleanup

## OCI Cleanup

### 1. Inside the VM

```bash
sudo systemctl stop httpd && sudo systemctl disable httpd
sudo dnf remove httpd -y
sudo dnf clean all
exit
```

### 2. OCI Console (in this order)

**Terminate the instance:**
Compute > Instances > `webserver-vm` > More Actions > Terminate
Check "Permanently delete the attached boot volume" > Confirm.

**Delete the VCN:**
Networking > Virtual Cloud Networks > `webserver-vcn` > More Actions > Delete > Confirm.

> If the VCN delete fails, go inside the VCN, delete subnets individually, then retry.


### 3. Local Machine

```bash
rm ~/.ssh/oci_key ~/.ssh/oci_key.pub
ssh-keygen -R YOUR_PUBLIC_IP
```

---
