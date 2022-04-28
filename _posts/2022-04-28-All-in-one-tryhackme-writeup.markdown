---
title: All in one tryhackme writeup
date: 2022-04-27 22:26:00 +0200
categories: [Blog, Writeup]
tags: [writeup, tryhackme, pentesting, pen, testing, hacking, ethical, all, in, one]
render_with_liquid: false
comments: false
description: this is a test

---

Writeup of me, **Bltz**, resolving tryhackme's easy room 'All in One', this is my first writeup of many. I'm a begginer so i might make some mistakes!

![AllinOne info](/assets/allinone.png)



### This CTF room include the next topics:
- Basic decrypt
- Wordpress user enumeration and file upload
- Fuzzing
- Double privilege escalation
- Abusing sudo privileges

# Scanning

I start the room by making the usual scanning. I always **ping** the ip to check that is running and the TTL so i can determine if it's windows or linux
and then i use **nmap** with syn scan script.
![Nmap scan for open ports](/assets/img/allinone/1.png)
_First scan with Nmap_

Now i continue with **nmap -sCV** on the previous open ports i got to check the services running on that ports and the version of them.

![Nmap service and version scripts](/assets/img/allinone/2.png)
_Nmap service and version scan_


I can see that anonymous ftp login is allowed, so i try to login and see if there's something usefull but its empty.
![Ftp anonymous login](/assets/img/allinone/3.png)
_Ftp anonymous login empty folder_


# Discovery

## Fuzzing
Then i take a look at the website and the default apache login is shown, i take a look at the source code but there's nothing out of the ordinary.
Fuzzing with gobuster makes the trick and 2 routes were found.
![Gobuster fuzzing](/assets/img/allinone/fuzz.png)
_Fuzzing with gobuster_

## Website
When browsing for **hackathons** route i encountered this:
![Hackathons route](/assets/img/allinone/4.png)
_hackathons route_

The hackathons route seems like this, i check the source code too and i dont see anything weird, so i continue with the **wordpress** route.

![wordpress route](/assets/img/allinone/5.png)
_wordpress page_



Here i can see that there’s a post with an username and i find the usual wordpress login page too 
in which i try to login as this user with the password _Vinegar_, but the password is invalid.

After this i realize that there's an uncommon header when i analyze a request to the wordpress page with burpsuite.
![Burpsuite](/assets/img/allinone/6.png)
_Burpsuite request of wordpress page_

There's a link in it and it happens that the wordpress wp-json resource is public, 
i check there if there is any user i dont know yet but no, there’s only one user and is the one i already have.:
I stucked a bit here, trying to fuzzing hackathons route and trying to find something in wordpress route, but i got nothing.
Then I go back to the Hackathons path and inspect the code again, seeing that there were two comments I didn't see.
![Hidden commentaries](/assets/img/allinone/7.png)
_Hidden comments in hackathon's source code_

# Gaining Access
The text of the first comment seems encrypted, while the other one doesn’t, so i try to use
[cyberchef’s](https://gchq.github.io/CyberChef/) magic module and [dcrypt.fr](http://dcrypt.fr) cipher identifier but none worked.
Then I think that maybe the second comment is a key for the cipher, so I google 'cipher with key' and the first result is Vigenere Cipher,
now it all makes sense and I see that the hackathons route was hiding a clue.

A quick decryption with cyberchef works and i got a password that works with the username i had.

![Login successfull, wordpress panel](/assets/img/allinone/8.png)
_Login successfull into the wordpress panel_

Then i use the theme editor to inject some php code into a .php file from the wordpress theme.

![Theme editor injection](/assets/img/allinone/9.png)
_Injecting php via Theme Editor_

With this i can reverse shell or just upload a .php file to bypass the usual wordpress file upload. I decide to send a reverse shell to my terminal through netcat and turn it into a tty.

![Access gained as www-data](/assets/img/allinone/10.png)
_Access gained_

# Privilege escalation

## Escalation to low privilege user

First i check the sudoers and suid files and there was nothing interesting, then i checked elyanas home directory and there was this hint:

```
Elyana's user password is hidden in the system. Find it ;)
```

Seeing that, i went to /var/www/html/wordpress/wp-config.php but the password was the same we got earlier and there was nothing in the database neither.
I continue by trying to find files owned by elyana and i find this:

![Untitled](/assets/img/allinone/11.png)
_Files found with elyana as owner_
Now we are logged as elyana 🙂

![We logged as elyana!](/assets/img/allinone/12.png)
_We manage to get logged as elyana!_
## Escalation to root

I check again sudoers and i see that we can execute socat as sudo without password, and [gtfobins](https://gtfobins.github.io/gtfobins/socat/) has this oneliner that let me privesc to root.

```jsx
sudo socat stdin exec:/bin/sh
```

![Bash as root](/assets/img/allinone/13.png)
_Shell as root_
We privesc to root but the flags from user.txt and root.txt seems to be encrypted.

![Encrypted flags](/assets/img/allinone/14.png)
_Both flags were encrypted_

A quick search in cyberchef’s magic module tells me that it’s base64, so i decrypt it and we are done.

![Decrypted flags](/assets/img/allinone/15.png)
_Decrypted flags_


Thank you for reading me, have a good day and see you soon in future writeups! 😎

