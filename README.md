# Cloudflare, IIS & ScreenConnect Setup Guide

A beginner-friendly, step-by-step guide for hosting a company website
and ScreenConnect behind IIS on a Windows Server, using Cloudflare DNS
and an Origin Certificate.

## What this setup provides

  Hostname                         Purpose

---

  `swiftsharelinkhub.com`          Company website
  `www.swiftsharelinkhub.com`      Company website
  `screen.swiftsharelinkhub.com`   ScreenConnect web client

### Example values used in this guide

Replace these with your own values when following the instructions.

  Setting                                       Example

---

  Domain                                        `swiftsharelinkhub.com`
  Server public IP                              `104.251.180.46`
  ScreenConnect web port                        `8040`
  ScreenConnect relay port                      `8041`
  Relay hostname in the example configuration   `relay.greenbeam.cfd`

> **Important:** The relay hostname above is the exact example used in
> the supplied configuration. Change it to a hostname you control if
> that is not your intended relay domain.

---

## Prerequisites

Before starting, make sure you have:

- A Windows Server with Administrator access.
- A domain managed through Cloudflare.
- ScreenConnect installed and running on the server.
- The public IP address of the server.
- Permission to change your VPS/provider firewall or cloud security
  group.
- A backup of important server configuration files.

## Part 1 --- Configure Cloudflare DNS

In Cloudflare, open your domain and go to **DNS → Records**.

Create these records:

  Type   Name       IPv4 address       Proxy status

---

  A      `@`        `104.251.180.46`   Proxied
  A      `www`      `104.251.180.46`   Proxied
  A      `screen`   `104.251.180.46`   Proxied

The relay hostname needs separate consideration. The standard Cloudflare
orange-cloud proxy does not proxy arbitrary TCP traffic on port `8041`.
Configure the relay hostname using a supported DNS-only or TCP proxy
arrangement. If using DNS-only, the hostname should resolve to the
server's public IP and port `8041` must be reachable.

## Part 2 --- Install IIS on Windows Server

Open **PowerShell as Administrator**.

Install IIS:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

Start IIS:

```powershell
Start-Service W3SVC
```

## Part 3 --- Open firewall ports

Allow inbound TCP ports 80 and 443 in Windows Firewall:

```powershell
New-NetFirewallRule -DisplayName "HTTP Port 80" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
```

```powershell
New-NetFirewallRule -DisplayName "HTTPS Port 443" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
```

Allow the ScreenConnect relay port 8041 if your relay configuration
requires inbound connections to this server:

```powershell
New-NetFirewallRule -DisplayName "ScreenConnect Relay Port 8041" -Direction Inbound -Protocol TCP -LocalPort 8041 -Action Allow
```

Also configure the equivalent inbound rules in your VPS provider
firewall or cloud security group. Only expose ports that are required by
your deployment.

## Part 4 --- Create website and certificate folders

Run:

```powershell
New-Item -ItemType Directory -Force C:\inetpub\company
New-Item -ItemType Directory -Force C:\inetpub\screen-proxy
New-Item -ItemType Directory -Force C:\inetpub\ssl
```

Create a simple test page:

```powershell
Set-Content C:\inetpub\company\index.html '<h1>My Website Is Working</h1>'
```

Place your actual company website files in:

```text
C:\inetpub\company
```

## Part 5 --- Install IIS URL Rewrite and ARR

Install:

1. [IIS URL
   Rewrite](https://www.iis.net/downloads/microsoft/url-rewrite)
2. [Application Request Routing
   (ARR)](https://www.iis.net/downloads/microsoft/application-request-routing)

Then:

1. Open IIS Manager.
2. Select the server name.
3. Open **Application Request Routing Cache**.
4. Select **Server Proxy Settings**.
5. Enable **Enable proxy**.
6. Click **Apply**.

## Part 6 --- Create IIS websites

### Company website

In IIS Manager:

1. Open **Sites**.
2. Select **Default Web Site**.
3. Choose **Basic Settings**.
4. Set the physical path to:

```text
C:\inetpub\company
```

5. Open **Bindings** and add:

```text
Type: http
Port: 80
Host name: swiftsharelinkhub.com
```

```text
Type: http
Port: 80
Host name: www.swiftsharelinkhub.com
```

### ScreenConnect reverse-proxy website

In IIS Manager:

1. Right-click **Sites**.
2. Select **Add Website**.
3. Enter:

```text
Site name: ScreenConnectProxy
Physical path: C:\inetpub\screen-proxy
Type: http
IP address: All Unassigned
Port: 80
Host name: screen.swiftsharelinkhub.com
```

Click **OK**.

## Part 7 --- Verify ScreenConnect locally

Check that ScreenConnect is responding on the example web port:

```powershell
curl.exe -I http://127.0.0.1:8040/
```

Confirm that the service is running and that `8040` is the correct web
port for your installation before continuing.

## Part 8 --- Configure the IIS reverse proxy

Create the IIS proxy configuration file at:

```text
C:\inetpub\screen-proxy\web.config
```

Use this content:

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

Test the local IIS host binding:

```powershell
curl.exe -I -H "Host: screen.swiftsharelinkhub.com" http://127.0.0.1/
```

This file is the **IIS reverse-proxy configuration**. It is not the
ScreenConnect application's own `web.config`.

## Part 9 --- Create a Cloudflare Origin Certificate

In Cloudflare:

1. Select your domain.
2. Open **SSL/TLS → Origin Server**.
3. Select **Create Certificate**.
4. Include these hostnames:

```text
swiftsharelinkhub.com
*.swiftsharelinkhub.com
```

5. Choose PEM format.
6. Create the certificate.
7. Save the Origin Certificate and Private Key securely on the server:

```text
C:\inetpub\ssl\origin.pem
C:\inetpub\ssl\private.key
```

**Security:** Never publish, commit, email, or share the private key.
Restrict access to the certificate files.

## Part 10 --- Install OpenSSL and create a PFX

Install OpenSSL for Windows. The example installation path is:

```text
C:\Program Files\OpenSSL-Win64\bin\openssl.exe
```

Check the installation:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version
```

Inspect the certificate names:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" x509 -in "C:\inetpub\ssl\origin.pem" -noout -subject -ext subjectAltName
```

The certificate should cover the root domain and wildcard hostname.

Find `legacy.dll` if your OpenSSL installation requires it:

```powershell
Get-ChildItem "C:\Program Files\OpenSSL-Win64" -Filter legacy.dll -Recurse -ErrorAction SilentlyContinue |
Select-Object -ExpandProperty FullName
```

Create the PFX:

```powershell
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -export `
-provider-path "C:\Program Files\OpenSSL-Win64\bin" `
-out "C:\inetpub\ssl\swiftsharelinkhub-compat.pfx" `
-inkey "C:\inetpub\ssl\private.key" `
-in "C:\inetpub\ssl\origin.pem" `
-name "SwiftShareLinkHub" `
-legacy
```

Set a strong PFX password when prompted. Keep it private.

## Part 11 --- Import the certificate

Enter the PFX password:

```powershell
$password = Read-Host "Enter your PFX password" -AsSecureString
```

Import the PFX:

```powershell
Import-PfxCertificate `
-FilePath "C:\inetpub\ssl\swiftsharelinkhub-compat.pfx" `
-CertStoreLocation "Cert:\LocalMachine\My" `
-Password $password `
-Exportable
```

Find the certificate thumbprint:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
Where-Object {$_.Subject -like "*CloudFlare Origin Certificate*"} |
Format-List Subject,Thumbprint,NotAfter,HasPrivateKey
```

Confirm that `HasPrivateKey` is `True`, then copy the thumbprint.

## Part 12 --- Add HTTPS bindings in IIS

Load the IIS PowerShell module:

```powershell
Import-Module WebAdministration
```

Set the thumbprint you copied:

```powershell
$thumb = "PASTE_YOUR_CERTIFICATE_THUMBPRINT_HERE"
```

Create HTTPS bindings:

```powershell
New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -HostHeader "swiftsharelinkhub.com" -SslFlags 1
```

```powershell
New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -HostHeader "www.swiftsharelinkhub.com" -SslFlags 1
```

```powershell
New-WebBinding -Name "ScreenConnectProxy" -Protocol https -Port 443 -HostHeader "screen.swiftsharelinkhub.com" -SslFlags 1
```

Attach the certificate:

```powershell
Get-WebBinding -Protocol https | ForEach-Object {
    $_.AddSslCertificate($thumb, "My")
}
```

Check the bindings:

```powershell
Get-WebBinding -Protocol https |
Format-Table protocol,bindingInformation,sslFlags
```

Verify that all three hostnames appear. If the server has other HTTPS
bindings, do not blindly attach this certificate to every binding;
select the intended bindings individually.

## Part 13 --- Configure ScreenConnect relay settings

**Important: These settings belong in ScreenConnect's own application
`web.config`, not in the IIS reverse-proxy file** at:

```text
C:\inetpub\screen-proxy\web.config
```

Back up the ScreenConnect application configuration file before editing
it.

Find the ScreenConnect installation directory and open its application
`web.config`. Locate the existing `<appSettings>` section. Add these two
entries inside that section:

```xml
<add key="RelayListenUri" value="relay://+:8041/" />
<add key="RelayAddressableUri" value="relay://relay.greenbeam.cfd:8041/" />
```

Example placement:

```xml
<configuration>
  <appSettings>
    <!-- Keep any existing appSettings entries -->
    <add key="RelayListenUri" value="relay://+:8041/" />
    <add key="RelayAddressableUri" value="relay://relay.greenbeam.cfd:8041/" />
  </appSettings>

  <!-- Keep the other existing configuration sections -->
</configuration>
```

Do not replace the entire existing `web.config` with this abbreviated
example. Preserve its existing settings and sections. Do not add a
second `<appSettings>` section if one already exists.

### Relay hostname and connectivity

- Make sure `relay.greenbeam.cfd` is a hostname you control and that
  it resolves to the correct public server IP.
- The relay hostname must be reachable by ScreenConnect clients on TCP
  port `8041`.
- The normal Cloudflare orange-cloud proxy does not proxy arbitrary
  TCP relay traffic on port `8041`. Use DNS-only for that hostname or
  a supported TCP proxy service such as an appropriately configured
  Spectrum deployment, where available.
- Confirm the ScreenConnect version's supported relay configuration
  and restart requirements before applying changes.
- After saving the configuration, restart the relevant ScreenConnect
  services during a maintenance window.
- Test an actual remote session. A working login page alone does not
  prove that relay connections work.

## Part 14 --- Test the websites locally

Test the root domain:

```powershell
curl.exe -k -I --resolve swiftsharelinkhub.com:443:127.0.0.1 https://swiftsharelinkhub.com/
```

Test `www`:

```powershell
curl.exe -k -I --resolve www.swiftsharelinkhub.com:443:127.0.0.1 https://www.swiftsharelinkhub.com/
```

Test ScreenConnect:

```powershell
curl.exe -k -I --resolve screen.swiftsharelinkhub.com:443:127.0.0.1 https://screen.swiftsharelinkhub.com/
```

Investigate any unexpected status code or connection error before
proceeding.

## Part 15 --- Set Cloudflare SSL mode

In Cloudflare, open **SSL/TLS → Overview** and select:

**Full (strict)**

Then test from a normal computer:

- [https://swiftsharelinkhub.com](https://swiftsharelinkhub.com)
- [https://www.swiftsharelinkhub.com](https://www.swiftsharelinkhub.com)
- [https://screen.swiftsharelinkhub.com](https://screen.swiftsharelinkhub.com)

## Final checklist

- [ ] Domain DNS records point to the server.
- [ ] TCP ports 80 and 443 are allowed.
- [ ] TCP relay port 8041 is configured as required.
- [ ] IIS is installed and running.
- [ ] URL Rewrite is installed.
- [ ] ARR is installed and proxy is enabled.
- [ ] Company website files are in the correct folder.
- [ ] ScreenConnect responds locally on its configured web port.
- [ ] IIS ScreenConnect reverse proxy is configured.
- [ ] Cloudflare Origin Certificate is installed.
- [ ] PFX was imported and has a private key.
- [ ] HTTPS bindings are correct.
- [ ] ScreenConnect's own `web.config` contains the relay settings.
- [ ] Relay hostname resolves and is reachable through the chosen TCP
  configuration.
- [ ] Cloudflare SSL mode is Full (strict).
- [ ] All three HTTPS hostnames have been tested publicly.
- [ ] A real ScreenConnect remote session has been tested.

## Troubleshooting notes

- **Website does not load:** Check DNS, IIS site bindings, Windows
  Firewall, provider firewall, and whether the IIS site is started.
- **502 or 500 response from ScreenConnect proxy:** Check that
  ScreenConnect is listening on `127.0.0.1:8040`, and verify URL
  Rewrite and ARR proxy settings.
- **Certificate error:** Check the certificate names, expiration,
  private-key presence, IIS HTTPS binding, and Cloudflare SSL mode.
- **Relay connection fails:** Check the relay hostname's DNS record,
  TCP port 8041, provider firewall, Cloudflare proxy mode, and
  ScreenConnect service/configuration logs.
- **Configuration error after editing:** Restore the backup if needed
  and validate that the XML is well-formed. Ensure the relay entries
  are inside the existing `<appSettings>` section.

---

## Security reminders

- Never commit `private.key`, PFX files, passwords, or other secrets
  to GitHub.
- Use a private repository if you need to store non-secret deployment
  notes.
- Restrict Windows Server Administrator access.
- Keep Windows Server, IIS, ScreenConnect, and installed components
  updated.
- Open only the ports required by your deployment.
- Test configuration changes during a maintenance window and keep
  backups.
