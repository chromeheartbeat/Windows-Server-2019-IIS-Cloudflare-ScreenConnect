# Windows Server 2019 + IIS + Cloudflare + ScreenConnect

## Step-by-step setup guide

This guide documents the setup used to host:

- `https://swiftsharelinkhub.com` --- company website
- `https://www.swiftsharelinkhub.com` --- company website alias
- `https://screen.swiftsharelinkhub.com` --- ScreenConnect web
  interface

The example server public IPv4 address is `104.251.180.46`. Replace it
with your own server IP and domain when following this guide.

> **Important:** This guide gets the company website and ScreenConnect
> web interface working through IIS HTTPS. It does **not** by itself
> complete or verify the ScreenConnect relay/client connection on port
> `8041`. Relay access is a separate networking task explained near the
> end.

---

## 1. Prerequisites

You need:

- Windows Server 2019 (or a compatible Windows Server version).
- Administrator access through RDP.
- A domain managed in Cloudflare.
- A public/static IPv4 address for the server.
- ScreenConnect installed and running locally.
- The ScreenConnect web service listening on port `8040` and relay
  service on port `8041` in this example.
- A Cloudflare Origin Certificate and its matching private key.
- IIS, URL Rewrite, and Application Request Routing (ARR).

### Example values used in this guide

  Setting                        Example

---

  Domain                         `swiftsharelinkhub.com`
  Server public IP               `104.251.180.46`
  Company website IIS site       `Default Web Site`
  ScreenConnect proxy IIS site   `ScreenConnectProxy`
  Company website folder         `C:\\inetpub\\company`
  ScreenConnect proxy folder     `C:\\inetpub\\screen-proxy`
  ScreenConnect web backend      `http://127.0.0.1:8040`
  ScreenConnect relay port       `8041`
  Certificate folder             `C:\\inetpub\\ssl`

Change these values to match your environment.

---

## 2. Prepare Cloudflare DNS

In Cloudflare, select your domain and open **DNS → Records**.

Create these records:

  Type   Name       IPv4 address            Proxy status

---

  A      `@`        Your server public IP   Proxied
  A      `www`      Your server public IP   Proxied
  A      `screen`   Your server public IP   Proxied

For this example, the IP is `104.251.180.46`.

The orange-cloud proxy is appropriate for normal website traffic on
ports 80 and 443. Do not assume that the standard Cloudflare proxy will
forward ScreenConnect relay traffic on port 8041. See [Section
15](#15-screenconnect-relay-port-8041).

---

## 3. Allow web traffic through the server firewall

In Windows Defender Firewall with Advanced Security, create inbound TCP
rules for:

- TCP `80` --- HTTP
- TCP `443` --- HTTPS

If you manage the server through a cloud provider (AWS, etc.), also
allow inbound TCP 80 and 443 in its network security rules/security
group.

Keep RDP access restricted to trusted IP addresses where possible. Do
not expose administrative ports unnecessarily.

You can check the Windows Firewall rules from PowerShell, but use the
Firewall UI or your organization's approved method to create rules if
you are unsure.

---

## 4. Install IIS

Open **Windows PowerShell as Administrator** and run:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

Check that the IIS service is running:

```powershell
Get-Service W3SVC
```

If it is not running:

```powershell
Start-Service W3SVC
```

IIS Manager should now be available from Server Manager → Tools →
Internet Information Services (IIS) Manager.

---

## 5. Install IIS URL Rewrite and ARR

Install these Microsoft IIS extensions:

1. **IIS URL Rewrite Module 2**
2. **Application Request Routing (ARR)**

Use the official Microsoft IIS download pages or your organization's
approved software source. Install both, then reopen IIS Manager.

### Enable ARR proxy

1. Open IIS Manager.
2. Select the server name at the top of the Connections pane (not an
   individual website).
3. Open **Application Request Routing Cache**.
4. In the Actions pane, select **Server Proxy Settings**.
5. Tick **Enable proxy**.
6. Click **Apply**.

The ARR proxy feature is required for IIS to forward ScreenConnect web
requests to the local service.

---

## 6. Confirm ScreenConnect is running locally

Before configuring IIS, verify the ScreenConnect web service is
reachable on the server itself.

In Administrator PowerShell:

```powershell
curl.exe -I http://127.0.0.1:8040/
```

A successful response such as `HTTP/1.1 200 OK` indicates the local web
interface is responding.

Check listeners:

```powershell
Get-NetTCPConnection -State Listen |
Where-Object { $_.LocalPort -in 8040,8041 } |
Format-Table LocalAddress,LocalPort,OwningProcess
```

The example setup uses:

- `8040` for the ScreenConnect web interface.
- `8041` for the ScreenConnect relay.

Your installation may use different ports. Confirm the actual ports in
your ScreenConnect configuration before proceeding. Do not terminate the
processes that own these ports.

---

## 7. Create the IIS website folders

Create the folders:

```powershell
New-Item -ItemType Directory -Force -Path C:\inetpub\company
New-Item -ItemType Directory -Force -Path C:\inetpub\screen-proxy
New-Item -ItemType Directory -Force -Path C:\inetpub\ssl
```

Put your company website files in:

```text
C:\inetpub\company
```

For a temporary test page, you can use:

```powershell
@'
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Company Website</title></head>
<body><h1>Company website is working</h1></body>
</html>
'@ | Set-Content -Path C:\inetpub\company\index.html -Encoding UTF8
```

The temporary page can be replaced with your actual website files.

---

## 8. Configure the IIS websites and HTTP bindings

Open IIS Manager.

### Company website

1. Select **Default Web Site**.
2. Open **Basic Settings**.
3. Set the Physical path to `C:\inetpub\company`.
4. Open **Bindings**.
5. Add HTTP bindings on port `80` for:
   - `swiftsharelinkhub.com`
   - `www.swiftsharelinkhub.com`

### ScreenConnect proxy website

Create a second IIS website if it does not already exist:

1. Right-click **Sites** → **Add Website**.
2. Site name: `ScreenConnectProxy`.
3. Physical path: `C:\inetpub\screen-proxy`.
4. Type: `http`.
5. IP address: `All Unassigned`.
6. Port: `80`.
7. Host name: `screen.swiftsharelinkhub.com`.

If you already have these sites, do not create duplicates. Check the
current bindings first.

You can inspect the configuration with:

```powershell
Import-Module WebAdministration

Get-Website | Format-Table Name,ID,State,PhysicalPath

Get-WebBinding |
Format-Table protocol,bindingInformation,sslFlags
```

The expected HTTP host bindings are:

```text
*:80:swiftsharelinkhub.com
*:80:www.swiftsharelinkhub.com
*:80:screen.swiftsharelinkhub.com
```

---

## 9. Configure the ScreenConnect reverse proxy

The ScreenConnect proxy website must forward requests to the local
ScreenConnect web service.

Create or edit:

```text
C:\inetpub\screen-proxy\web.config
```

Use this basic URL Rewrite reverse-proxy rule:

```xml
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
```

Save the file as `web.config` (not `web.config.txt`).

**Important:** This is the basic HTTP reverse-proxy rule used for the
web-interface test. ScreenConnect features that rely on WebSockets or
particular request headers may require additional ARR/URL Rewrite
configuration. Test the actual session and management workflows after
deployment; a successful homepage response alone does not prove every
feature works.

### Test the proxy over local HTTP

Run:

```powershell
curl.exe -I -H "Host: screen.swiftsharelinkhub.com" http://127.0.0.1/
```

A response from ScreenConnect (in the example, HTTP 200 and
`X-Powered-By: ARR/3.0`) indicates that the IIS reverse proxy is
forwarding the request.

Also test the company site:

```powershell
curl.exe -I -H "Host: swiftsharelinkhub.com" http://127.0.0.1/
```

---

## 10. Create a Cloudflare Origin Certificate

In the Cloudflare dashboard:

1. Select your domain.
2. Open **SSL/TLS → Origin Server**.
3. Select **Create Certificate**.
4. Include these hostnames:
   - `swiftsharelinkhub.com`
   - `*.swiftsharelinkhub.com`
5. Choose a suitable private-key type (RSA is commonly supported).
6. Choose PEM format.
7. Create the certificate.
8. Copy/download both:
   - Origin Certificate (PEM)
   - Private Key (PEM)

Keep the private key secret. Do not email it, publish it, or share it
with friends. Each server owner should create their own Origin
Certificate and private key.

Copy the files to the server and use these example names:

```text
C:\inetpub\ssl\origin.pem
C:\inetpub\ssl\private.key
```

Confirm the files exist:

```powershell
Get-Item C:\inetpub\ssl\origin.pem
Get-Item C:\inetpub\ssl\private.key
```

---

## 11. Install OpenSSL and create a PFX file

Install a trusted Windows build of OpenSSL. In this example, OpenSSL is
installed at:

```text
C:\Program Files\OpenSSL-Win64\bin\openssl.exe
```

Check that path and version:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version
```

### Check certificate names

Run:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" x509 `
-in "C:\inetpub\ssl\origin.pem" `
-noout -subject -ext subjectAltName
```

For this setup, the output should include:

```text
DNS:*.swiftsharelinkhub.com
DNS:swiftsharelinkhub.com
```

The root domain must be explicitly included. A wildcard alone does not
cover the root domain.

### Find the Legacy Provider DLL if needed

Some OpenSSL 3/4 Windows installations need the legacy provider for
compatibility exports. Locate it:

```powershell
Get-ChildItem "C:\Program Files\OpenSSL-Win64" `
-Filter legacy.dll -Recurse -ErrorAction SilentlyContinue |
Select-Object -ExpandProperty FullName
```

In the tested setup it was located at:

```text
C:\Program Files\OpenSSL-Win64\bin\legacy.dll
```

If the DLL is in that folder, use `-provider-path` with the same folder
when running `-legacy`.

### Create the PFX

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

OpenSSL asks for an export password. Create a strong password and keep
it private. You will need the same password when importing the PFX.

If your OpenSSL installation does not use a legacy provider or does not
support the `-provider-path` option, consult that build's documentation
rather than copying DLLs from an unrelated OpenSSL installation.

---

## 12. Import the PFX into the Windows certificate store

In the **same Administrator PowerShell session**, enter the PFX password
as a secure string:

```powershell
$password = Read-Host "Enter the PFX export password you created" -AsSecureString
```

Import it:

```powershell
Import-PfxCertificate `
-FilePath "C:\inetpub\ssl\swiftsharelinkhub-compat.pfx" `
-CertStoreLocation "Cert:\LocalMachine\My" `
-Password $password `
-Exportable
```

A successful import displays a certificate thumbprint and subject.

### Verify the certificate

Use the thumbprint returned by the import:

```powershell
Get-Item "Cert:\LocalMachine\My\<PASTE_THUMBPRINT_HERE>" |
Format-List Subject,Thumbprint,NotAfter,HasPrivateKey
```

The important result is:

```text
HasPrivateKey : True
```

If import reports a password error, re-enter the exact PFX export
password. Make sure `$password` is populated in the current PowerShell
session. Never send the password or private key to anyone.

---

## 13. Add HTTPS bindings in IIS

Before adding bindings, inspect current sites and bindings:

```powershell
Import-Module WebAdministration

Get-Website | Format-Table Name,ID,State,PhysicalPath

Get-WebBinding |
Format-Table protocol,bindingInformation,sslFlags
```

Use the certificate thumbprint returned by your own import. The
following is the example thumbprint from the documented setup; **replace
it with your own**:

```powershell
$thumb = "D3EDC3D14064994FB98B1A5A2D4A5717B3EA25"
```

> Copy the thumbprint exactly from your certificate store. Do not use
> the example value above for another server.

Create the three SNI HTTPS bindings:

```powershell
New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 `
-HostHeader "swiftsharelinkhub.com" -SslFlags 1

New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 `
-HostHeader "www.swiftsharelinkhub.com" -SslFlags 1

New-WebBinding -Name "ScreenConnectProxy" -Protocol https -Port 443 `
-HostHeader "screen.swiftsharelinkhub.com" -SslFlags 1
```

Attach the certificate to the HTTPS bindings:

```powershell
Get-WebBinding -Protocol https | ForEach-Object {
    $_.AddSslCertificate($thumb, "My")
}
```

`-SslFlags 1` enables Server Name Indication (SNI), allowing multiple
hostnames to share port 443.

If the bindings already exist, do not run the `New-WebBinding` commands
again. Inspect and update the existing bindings instead.

### Verify HTTPS bindings

```powershell
Get-WebBinding -Protocol https |
Format-Table protocol,bindingInformation,sslFlags
```

Expected hostnames:

```text
*:443:swiftsharelinkhub.com
*:443:www.swiftsharelinkhub.com
*:443:screen.swiftsharelinkhub.com
```

---

## 14. Test HTTPS locally before testing Cloudflare

Run these commands on the Windows server:

### Root domain

```powershell
curl.exe -k -I --resolve swiftsharelinkhub.com:443:127.0.0.1 `
https://swiftsharelinkhub.com/
```

### WWW hostname

```powershell
curl.exe -k -I --resolve www.swiftsharelinkhub.com:443:127.0.0.1 `
https://www.swiftsharelinkhub.com/
```

### ScreenConnect hostname

```powershell
curl.exe -k -I --resolve screen.swiftsharelinkhub.com:443:127.0.0.1 `
https://screen.swiftsharelinkhub.com/
```

In the documented setup, all three returned:

```text
HTTP/1.1 200 OK
```

The ScreenConnect response also included:

```text
X-Powered-By: ARR/3.0
```

The `-k` option skips certificate trust verification for this local
test. It does not mean you should disable certificate validation in
normal use.

---

## 15. Configure Cloudflare SSL/TLS and test public access

In Cloudflare:

1. Open **SSL/TLS → Overview**.
2. Select **Full (strict)**.
3. Confirm the DNS records point to the correct server IP and are
   proxied for the website hostnames.
4. Wait for DNS changes to take effect.

From a separate computer or network, test:

- `https://swiftsharelinkhub.com`
- `https://www.swiftsharelinkhub.com`
- `https://screen.swiftsharelinkhub.com`

If a site fails, check: - Cloudflare DNS record and proxy status. -
Cloudflare SSL/TLS mode. - Server firewall inbound TCP 80 and 443. -
Cloud provider security-group/network firewall. - IIS site state and
bindings. - IIS logs and ScreenConnect service status.

A Cloudflare Origin Certificate is intended for the Cloudflare-to-origin
connection. Browsers normally trust the publicly issued Cloudflare edge
certificate, not the origin certificate. Avoid testing the origin IP
directly in a browser and expecting the Origin Certificate to be
publicly trusted.

---

## 16. ScreenConnect relay port 8041 --- separate requirement

The ScreenConnect web interface and the ScreenConnect relay are
different services.

This guide verifies the web interface through:

```text
Browser → Cloudflare HTTPS → IIS on port 443 → ARR → 127.0.0.1:8040
```

It does **not** prove that ScreenConnect clients can reach the relay
service.

The standard Cloudflare orange-cloud proxy supports specific HTTP/HTTPS
ports; it does not automatically proxy arbitrary TCP relay traffic on
port `8041`.

Before exposing relay traffic:

1. Confirm the actual relay port and protocol in your ScreenConnect
   installation.
2. Consult the current ConnectWise ScreenConnect networking and
   firewall documentation for the required client/relay ports.
3. Choose a supported relay exposure method, such as an appropriate
   direct DNS-only relay hostname with a narrowly scoped firewall rule,
   or a suitable TCP proxy service such as Cloudflare Spectrum if
   available for your plan and use case.
4. Configure ScreenConnect's public relay/hostname settings according
   to its current documentation.
5. Test a real remote session from a separate network.

Do not simply orange-cloud a DNS record for port 8041 and assume the
relay will work. Do not expose unnecessary ports.

---

## 17. Final verification checklist

Use this checklist before calling the deployment complete:

- [ ] DNS `@` points to the correct public IP.
- [ ] DNS `www` points to the correct public IP.
- [ ] DNS `screen` points to the correct public IP.
- [ ] Windows Firewall allows inbound TCP 80 and 443.
- [ ] Cloud provider firewall/security group allows inbound TCP 80 and
  443.
- [ ] IIS and W3SVC are running.
- [ ] URL Rewrite and ARR are installed.
- [ ] ARR proxy is enabled.
- [ ] ScreenConnect web service responds locally on the configured
  port.
- [ ] ScreenConnect reverse proxy responds locally.
- [ ] Origin certificate includes the root domain and required
  subdomains.
- [ ] PFX imports successfully and `HasPrivateKey` is `True`.
- [ ] All three IIS HTTPS bindings use SNI and the correct
  certificate.
- [ ] All three local HTTPS tests return the expected response.
- [ ] Cloudflare SSL/TLS mode is Full (strict).
- [ ] All three hostnames work from an external network.
- [ ] ScreenConnect WebSocket/session features have been tested.
- [ ] ScreenConnect relay/client connectivity has been configured and
  tested separately.

---

## 18. Security notes

- Keep the Cloudflare Origin Certificate private key and PFX password
  secret.
- Do not reuse another person's private key or certificate. Each
  server/domain owner should generate their own.
- Use a strong, unique password for the PFX.
- Restrict RDP access to trusted IP addresses where practical.
- Keep Windows Server, IIS, ScreenConnect, URL Rewrite, and ARR
  updated.
- Back up the certificate and private key securely. Protect backups as
  sensitive secrets.
- Do not expose ScreenConnect relay or administrative ports unless
  required and properly secured.
- Use official vendor documentation for current ScreenConnect port
  requirements and relay configuration.

## Useful Microsoft and Cloudflare documentation

- IIS: https://learn.microsoft.com/iis/
- IIS URL Rewrite: https://www.iis.net/downloads/microsoft/url-rewrite
- IIS Application Request Routing:
  https://www.iis.net/downloads/microsoft/application-request-routing
- Cloudflare Origin CA:
  https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/
- Cloudflare SSL modes:
  https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/
- Cloudflare network ports:
  https://developers.cloudflare.com/fundamentals/reference/network-ports/

---

**End of guide**

This document describes the configuration and tests performed in one
Windows Server 2019 environment. Server versions, ScreenConnect
versions, ports, and Cloudflare plan capabilities can differ. Verify
each setting on the server you are configuring.
