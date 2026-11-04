
# SwiftShareLinkHub — Website & ScreenConnect Setup Guide

A beginner-friendly, step-by-step guide to configuring Cloudflare DNS, Windows Server, IIS, HTTPS certificates, and ScreenConnect behind an IIS reverse proxy.

## Overview

This guide configures the following services:

| Service | Domain |
|---|---|
| Company website | `swiftsharelinkhub.com` |
| Company website (www) | `www.swiftsharelinkhub.com` |
| ScreenConnect | `screen.swiftsharelinkhub.com` |

### Example Configuration

| Setting | Value |
|---|---|
| Domain | `swiftsharelinkhub.com` |
| Server IP | `104.251.180.46` |
| ScreenConnect web port | `8040` |
| ScreenConnect relay port | `8041` |
| Operating system | Windows Server |
| Web server | IIS |
| DNS/CDN/SSL | Cloudflare |

> **Important:** Replace the example domain, server IP, and certificate details with your own when using this guide for another server.

---

# PART 1 — CLOUDFLARE DNS CONFIGURATION

## Step 1: Add DNS Records

1. Log in to [Cloudflare](https://dash.cloudflare.com/).
2. Select your domain.
3. Navigate to **DNS → Records**.
4. Add the following DNS records.

| Type | Name | IPv4 Address | Proxy Status |
|---|---|---|---|
| A | `@` | `104.251.180.46` | Proxied |
| A | `www` | `104.251.180.46` | Proxied |
| A | `screen` | `104.251.180.46` | Proxied |

Save all records.

---

# PART 2 — WINDOWS SERVER CONFIGURATION

Open **PowerShell as Administrator**.

## Step 2: Install IIS

Run:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

Start the IIS service:

```powershell
Start-Service W3SVC
```

## Step 3: Open Windows Firewall Ports

Allow incoming HTTP traffic on port 80:

```powershell
New-NetFirewallRule -DisplayName "HTTP Port 80" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
```

Allow incoming HTTPS traffic on port 443:

```powershell
New-NetFirewallRule -DisplayName "HTTPS Port 443" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
```

Also open TCP ports **80 and 443** in your VPS firewall or AWS Security Group.

## Step 4: Create Required Folders

```powershell
New-Item -ItemType Directory -Force C:\inetpub\company
New-Item -ItemType Directory -Force C:\inetpub\screen-proxy
New-Item -ItemType Directory -Force C:\inetpub\ssl
```

## Step 5: Create a Test Website

```powershell
Set-Content C:\inetpub\company\index.html '<h1>My Website Is Working</h1>'
```

Your actual company website files should eventually be placed inside:

```text
C:\inetpub\company
```

---

# PART 3 — INSTALL IIS ADD-ONS

## Step 6: Download and Install Required Components

Install the following:

1. [IIS URL Rewrite Module](https://www.iis.net/downloads/microsoft/url-rewrite)
2. [Application Request Routing (ARR)](https://www.iis.net/downloads/microsoft/application-request-routing)

### Enable ARR Proxy

After installing both components:

1. Open **IIS Manager**.
2. Click your server name in the left panel.
3. Open **Application Request Routing Cache**.
4. Click **Server Proxy Settings**.
5. Check **Enable proxy**.
6. Click **Apply**.

---

# PART 4 — CREATE IIS WEBSITES

## Step 7: Configure the Company Website

1. Open IIS Manager.
2. Click **Sites**.
3. Select **Default Web Site**.
4. Click **Basic Settings**.
5. Set the physical path to:

```text
C:\inetpub\company
```

6. Click **Bindings**.
7. Add the following HTTP bindings.

### Root Domain

```text
Type: http
Port: 80
Host name: swiftsharelinkhub.com
```

### WWW Domain

```text
Type: http
Port: 80
Host name: www.swiftsharelinkhub.com
```

## Step 8: Create the ScreenConnect Proxy Website

In IIS Manager:

1. Right-click **Sites**.
2. Select **Add Website**.
3. Enter the following configuration.

```text
Site name: ScreenConnectProxy
Physical path: C:\inetpub\screen-proxy
Type: http
IP address: All Unassigned
Port: 80
Host name: screen.swiftsharelinkhub.com
```

4. Click **OK**.

---

# PART 5 — CONNECT SCREENCONNECT TO IIS

## Step 9: Verify ScreenConnect Is Running

Run:

```powershell
curl.exe -I http://127.0.0.1:8040/
```

Expected response:

```text
HTTP/1.1 200 OK
```

If you do not receive a successful response, stop and verify:

- ScreenConnect is running.
- Port `8040` is correct.
- The local ScreenConnect web service is accessible.

## Step 10: Create the Reverse Proxy Configuration

Run the following entire command in PowerShell:

```powershell
@'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="ScreenConnect Reverse Proxy" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite"
                  url="http://127.0.0.1:8040/{R:1}"
                  appendQueryString="true" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
'@ | Set-Content C:\inetpub\screen-proxy\web.config -Encoding UTF8
```

### Test the Reverse Proxy

```powershell
curl.exe -I -H "Host: screen.swiftsharelinkhub.com" http://127.0.0.1/
```

Expected result:

```text
HTTP/1.1 200 OK
```

---

# PART 6 — CREATE A CLOUDFLARE ORIGIN CERTIFICATE

## Step 11: Generate the Origin Certificate

In Cloudflare:

1. Select your domain.
2. Navigate to **SSL/TLS → Origin Server**.
3. Click **Create Certificate**.
4. Include the following hostnames:

```text
swiftsharelinkhub.com
*.swiftsharelinkhub.com
```

5. Select **PEM** format.
6. Create the certificate.

Copy both generated files:

- Origin Certificate
- Private Key

Save them on your Windows Server as:

```text
C:\inetpub\ssl\origin.pem
C:\inetpub\ssl\private.key
```

> **Security warning:** Never share, publish, or commit your private key to GitHub. Protect it carefully.

---

# PART 7 — INSTALL OPENSSL AND CREATE A PFX CERTIFICATE

## Step 12: Verify OpenSSL Installation

Install OpenSSL for Windows if it is not already installed.

Example installation path:

```text
C:\Program Files\OpenSSL-Win64\bin\openssl.exe
```

Check the installed version:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version
```

## Step 13: Verify Certificate Hostnames

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" x509 -in "C:\inetpub\ssl\origin.pem" -noout -subject -ext subjectAltName
```

Expected output should include:

```text
DNS:*.swiftsharelinkhub.com
DNS:swiftsharelinkhub.com
```

## Step 14: Locate legacy.dll

Run:

```powershell
Get-ChildItem "C:\Program Files\OpenSSL-Win64" -Filter legacy.dll -Recurse -ErrorAction SilentlyContinue |
Select-Object -ExpandProperty FullName
```

Example location:

```text
C:\Program Files\OpenSSL-Win64\bin\legacy.dll
```

## Step 15: Create the PFX Certificate

Run:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -export `
-provider-path "C:\Program Files\OpenSSL-Win64\bin" `
-out "C:\inetpub\ssl\swiftsharelinkhub-compat.pfx" `
-inkey "C:\inetpub\ssl\private.key" `
-in "C:\inetpub\ssl\origin.pem" `
-name "SwiftShareLinkHub" `
-legacy
```

OpenSSL will request a password.

Create a strong password and store it securely. Do not share it.

---

# PART 8 — IMPORT THE CERTIFICATE INTO WINDOWS

## Step 16: Import the PFX Certificate

Enter your PFX password:

```powershell
$password = Read-Host "Enter your PFX password" -AsSecureString
```

Import the certificate:

```powershell
Import-PfxCertificate `
-FilePath "C:\inetpub\ssl\swiftsharelinkhub-compat.pfx" `
-CertStoreLocation "Cert:\LocalMachine\My" `
-Password $password `
-Exportable
```

## Step 17: Retrieve the Certificate Thumbprint

Run:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
Where-Object {$_.Subject -like "*CloudFlare Origin Certificate*"} |
Format-List Subject,Thumbprint,NotAfter,HasPrivateKey
```

Confirm that the certificate shows:

```text
HasPrivateKey : True
```

Copy the certificate **Thumbprint**. You will need it in the next section.

---

# PART 9 — ENABLE HTTPS IN IIS

## Step 18: Import the IIS PowerShell Module

```powershell
Import-Module WebAdministration
```

## Step 19: Enter Your Certificate Thumbprint

Replace the placeholder with the actual thumbprint obtained in Step 17.

```powershell
$thumb = "PASTE_YOUR_CERTIFICATE_THUMBPRINT_HERE"
```

## Step 20: Create HTTPS Bindings

### Root Domain

```powershell
New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -HostHeader "swiftsharelinkhub.com" -SslFlags 1
```

### WWW Domain

```powershell
New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -HostHeader "www.swiftsharelinkhub.com" -SslFlags 1
```

### ScreenConnect

```powershell
New-WebBinding -Name "ScreenConnectProxy" -Protocol https -Port 443 -HostHeader "screen.swiftsharelinkhub.com" -SslFlags 1
```

## Step 21: Attach the SSL Certificate

```powershell
Get-WebBinding -Protocol https | ForEach-Object {
    $_.AddSslCertificate($thumb, "My")
}
```

## Step 22: Verify HTTPS Bindings

```powershell
Get-WebBinding -Protocol https |
Format-Table protocol,bindingInformation,sslFlags
```

Confirm that all three hostnames are listed.

---

# PART 10 — TEST ALL WEBSITES

Run the following tests from PowerShell on the Windows Server.

## Step 23: Test the Company Website

```powershell
curl.exe -k -I --resolve swiftsharelinkhub.com:443:127.0.0.1 https://swiftsharelinkhub.com/
```

## Step 24: Test the WWW Website

```powershell
curl.exe -k -I --resolve www.swiftsharelinkhub.com:443:127.0.0.1 https://www.swiftsharelinkhub.com/
```

## Step 25: Test ScreenConnect

```powershell
curl.exe -k -I --resolve screen.swiftsharelinkhub.com:443:127.0.0.1 https://screen.swiftsharelinkhub.com/
```

Expected response for all three:

```text
HTTP/1.1 200 OK
```

The `-k` option skips certificate validation for these local tests only. It does not change the certificate configuration.

---

# PART 11 — FINAL CLOUDFLARE CONFIGURATION

## Step 26: Set SSL/TLS Mode

1. Log in to Cloudflare.
2. Select your domain.
3. Navigate to **SSL/TLS → Overview**.
4. Set the encryption mode to:

```text
Full (strict)
```

## Step 27: Test the Websites Publicly

Open the following URLs from your normal computer:

- https://swiftsharelinkhub.com
- https://www.swiftsharelinkhub.com
- https://screen.swiftsharelinkhub.com

Confirm that all websites load correctly over HTTPS.

---

# IMPORTANT — SCREENCONNECT REMOTE SESSIONS

The configuration above exposes the ScreenConnect web interface through HTTPS.

ScreenConnect also requires a separate relay service, which in this example listens on:

```text
Port: 8041
```

The standard Cloudflare orange-cloud proxy does not automatically support arbitrary TCP traffic on port `8041`.

The relay service requires its own supported configuration and testing.

> **Important:** Successfully opening the ScreenConnect login page does not guarantee that remote sessions will connect. The relay service must be configured and tested separately.

---

# FINAL CHECKLIST

Use this checklist to confirm the setup is complete.

- [ ] Domain points to the correct server IP.
- [ ] TCP ports 80 and 443 are open.
- [ ] IIS is installed and running.
- [ ] IIS URL Rewrite is installed.
- [ ] Application Request Routing is installed.
- [ ] ARR proxy is enabled.
- [ ] ScreenConnect works locally on port 8040.
- [ ] Company website directory is created.
- [ ] ScreenConnect proxy website is created.
- [ ] Cloudflare Origin Certificate is generated.
- [ ] PFX certificate is created successfully.
- [ ] PFX certificate is imported into Windows.
- [ ] HTTPS bindings are configured in IIS.
- [ ] SSL certificate is attached to the HTTPS bindings.
- [ ] All three websites return HTTP 200 during local tests.
- [ ] Cloudflare SSL mode is set to Full (strict).
- [ ] All three websites work publicly over HTTPS.
- [ ] ScreenConnect relay service on port 8041 is tested separately.

---

## SECURITY REMINDERS

1. Use your own domain and server IP.
2. Never reuse another person's private key.
3. Never publish your private key or PFX password.
4. Do not commit private certificates, private keys, or passwords to GitHub.
5. Secure your Windows Server administrator account.
6. Keep Windows Server, IIS, and ScreenConnect updated.
7. Verify ScreenConnect relay connectivity separately from its web interface.

---

**End of Guide**

