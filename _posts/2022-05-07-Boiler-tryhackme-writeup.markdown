---
title: Boiler | Tryhackme writeup
date: 2022-05-05 18:30:00 +0200
categories: [Blog, Writeup]
tags: [writeup, tryhackme, pentesting, pen, testing, hacking, ethical, Boiler, ftp]
comments: false

---



I've recently completed BoilerCTF room from tryhackme, i hope it will be useful to someone, it was a nice room including:
- Enumeration
- Fuzzing
- A lot of rabbit holes :)
- sar2html exploitation
- SUID privilege escalation.

Here we go:

# Scanning
Like in any other CTF, i start by doing a scan of any open port with **nmap** followed by another scan with **-sCV** options for service name and version.

![Nmap scan for open ports](/assets/img/boil/001.png)
_First scan with Nmap_
![Nmap scan for versions](/assets/img/boil/002.png)
_Versions scan_

This time there's 4 open ports of which ftp is the most interesting to me because boilymous ftp login is enabled.
Knowking this i just login as boilymous with no password and there's a .txt file inside, but it seems to be encrypted.

![Txt file contents](/assets/img/boil/003.png)
_Txt file contents_
It seems like a letter substitution to me so the first thing i test is ROT13. I use CyberChef to decrypt it and it's in fact ROT13, but there's nothing useful:

```
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```
# Discovery
So now i have another 3 ports to work on, and i decide to follow the order given by the room flags. To answer the third, i check the web inside the port 10000 http service.
There's a webmin page there, and i know that the version is 1.930 thanks to the previous version scan i did. Knowing this i check searchsploit for vulnerabilities but the three showing up are for previous versions.
With this i answer third flag with nay.

Now i try to fuzz over port 80 route for answering question 5 and i quickly find /joomla that matches with question 5. When searching for the route the default joomla page is showing.
Fuzzing again showed some more routes, i can see the admin login panel /administrator and some more joomla routes.
![Fuzzing](/assets/img/boil/007.png)
_Fuzzing port 80_

When i check the ones with "_" i can see some encrypted messages but all of them are rabbit holes except "_test" that is a sar2html tool.
# Gaining access
I've worked with sar2html before so i know that there's an easy vulnerability around the _plot_ variable.
Just by assigning ;**command** to _plot_ we can see the output of the command in the select host dropdown. 
![sar2html exploit](/assets/img/boil/008.png)
_sar2html exploit_ 


As the tryhackme page hints that i don't need to reverse shell, i do **ls** and there's a log.txt file that catches my attention.
And inside this file i find a log of connections with an user and password.

![log.txt contents](/assets/img/boil/009.png)
_User and password inside log.txt_ 

Now i can connect with ssh to port 55007 with that credentials.

![Access gained](/assets/img/boil/012.png)
_Access gained as low privilege user_
# Privilege escalation

When going to /home i can see that there's another user called stoner.
Then there's a file called backup.sh in my user's home directory, and when i check the code to see what the script does
i see a comment with a string very similar to the password i got before.
![backup script contents](/assets/img/boil/010.png)
_Comment inside backup.sh_


So i try to change to stoner with that password and it works!

![Access gained](/assets/img/boil/011.png)
_Access gained as stoner_

I find a file with a flag in stoner's home.

![Flag](/assets/img/boil/013.png)
_Flag found_

Stoner doesn't have sudo privileges so i search for SUID files, and this is what i get:

![SUID files](/assets/img/boil/014.png)
_SUID files_

I know that **find** as SUID is a privesc vector from previous experience, so i can just execute this and privesc to root:

```bash
./find . -exec /bin/bash -p \; -quit
```

![Privesc root](/assets/img/boil/015.png)
_Privilege escalated to root_

Last flag is inside /root folder and now the room is done!

It was an interesting room involving enumeration, fuzzing and a cool RCE with sar2html. Have a good time with it!

