+++
title = "THM - Publisher writeup"
date = "2024-07-25T23:23:30+02:00"
tags = ["tryhackme", "linux", "spip", "apparmor", "nmap", "gobuster", "CVE-2023-27372", "shebang"]
aliases = ["/2024/07/25/publisher/"]
categories = ["Pentesting", "Writeup", "CTF"]
showTableOfContents = true
showHeadingAnchors = true
heroStyle = "background"
showHero = true
+++

Hi,

This is my writeup about the [TryHackMe](https://tryhackme.com) box ["Publisher"](https://tryhackme.com/r/room/publisher).

## Information gathering

As always we have to know what is in front of us:

```bash
nmap -Pn TARGET_MACHINE_IP -oN ports && nmap -Pn -sC -sV -p $(grep -Po '.*(?=/tcp)' ports | tr '\n' ',') TARGET_MACHINE_IP -oN services
```



This scan reveals only two ports:

{{<figure-dynamic
dark-src="/images/publisher/gobuster_scan_dark.png"
light-src="/images/publisher/gobuster_scan_light.png"
>}}

I have checked the ssh service for password authentication and it accepts, but bruteforcing ssh is not attractive currently without knowing any usernames.
I also take a look at the website, without luck.

![website](/images/publisher/publisher_website_dark.png)

All links are empty, except those on "Related Blogs", but this is out of scope. Don't attack them!

Next I run gobuster against the machine.

{{<figure-dynamic
light-src="/images/publisher/gobuster_scan_light.png"
dark-src="/images/publisher/gobuster_scan_dark.png"
>}}

And there is a new page to take a look at! As always, search for version number.

{{<figure-dynamic
light-src="/images/publisher/spip_version_light.png"
dark-src="/images/publisher/spip_version_dark.png"
>}}

## Exploit SPIP

This version of spip is vulnerable to [CVE-2023-27372](https://www.exploit-db.com/exploits/51536).

{{<figure-dynamic
light-src="/images/publisher/searchsploit_light.png"
dark-src="/images/publisher/searchsploit_dark.png"
>}}

I copied the script to my local directory and open it in nvim. The script is not very useful by default. I had to change one thing.

> If you got some error because of `urllib3<2`, then look at this [comment](https://github.com/psf/requests/issues/6443#issuecomment-1535667256) on github, it solved my problem with the requests module.


I've changed the script by adding this line to the `send_payload` function, right before the return command:

```python
print(r.content)
```


After that I just run the script as it was. But the result is not as handy as I wish for...

{{<figure-dynamic
light-src="/images/publisher/exploit_mess_light.png"
dark-src="/images/publisher/exploit_mess_dark.png"
>}}

Playing around with [regex101](https://regex101.com), I was able to create a shell oneliner, which makes the output more handy or "humen readable".

```shell
python ./51536.py -u "http://$target/spip" -c "cat /home/think/user.txt" | ggrep -Po '(?<=value\\="s\\:[0-9]{2}:\\")[^";]*' | awk '{gsub(/\\\n/,"\n")}1'
```


Result:

{{<figure-dynamic
light-src="/images/publisher/exploit_clean_light.png"
dark-src="/images/publisher/exploit_clean_dark.png"
>}}

This looks way better now! To get the first flag, take a look into the users home directory:

{{<figure-dynamic
light-src="/images/publisher/user_flag_light.png"
dark-src="/images/publisher/user_flag_dark.png"
>}}

## SSH

To get ssh access, I just copied the ssh private key of user `think` to my local directory.

```shell
python ./51536.py -u "http://$target/spip" -c "cat /home/think/.ssh/id_rsa" | ggrep -Po '(?<=value\\="s\\:[0-9]{2}:\\")[^";]*' | awk '{gsub(/\\\n/,"\n")}1' > think_pk
```


And ssh into it!

{{<figure-dynamic
light-src="/images/publisher/ssh_access_light.png"
dark-src="/images/publisher/ssh_access_dark.png"
>}}

## PrivEsc

While looking around I found something suspicious:

{{<figure-dynamic
light-src="/images/publisher/rbash_light.png"
dark-src="/images/publisher/rbash_dark.png"
>}}

This is strange, because as `ls -la` shows, I should be able to read the content of the directory. This discovery was accidental because I was looking for a script or something similar.
But with that, I'm on the right track. We remember the instruction to the task:

> Attempts to escalate privileges using a custom binary are hindered by restricted access to critical system files and directories, necessitating a deeper exploration into the system's security profile to ultimately exploit a loophole that enables the execution of an unconfined bash shell and achieve privilege escalation.

Because the permissions from the file system doesn't match the observed behaviour, I decided to check for [ACL](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#acls) and [SELinux](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#selinux) and also for [AppArmor](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#apparmor).

{{<figure-dynamic
light-src="/images/publisher/apparmor_light.png"
dark-src="/images/publisher/apparmor_dark.png"
>}}

AppArmor is enabled, this must be the reason for the strange behaviour. AppArmor policies lays in the `/etc/apparmor.d/`

One lookup on the directory reveals the problem. The shell of the user `think` is [ash](https://en.wikipedia.org/wiki/Almquist_shell), which has a profile on `/etc/apparmor.d/`

When looking at the profile for the ash shell, I observed a hole.

{{<figure-dynamic
light-src="/images/publisher/aa_policy_light.png"
dark-src="/images/publisher/aa_policy_dark.png"
>}}

The missing wildcard's are the problem here. Without that, the deny rule only prevents me from changing a file with that name. But the content of the directory, is not affected by the deny rule.
With this in mind and the fact that only the ash shell is restricted by apparmor, the solution is obviously.

The trick is explained on hacktricks: [AppArmor Shebang Bypass](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/apparmor?ref=benheater.com#apparmor-shebang-bypass)



I created a file in the directory of `/dev/shm/`

```shell
#!/bin/bash
/bin/bash -pi
```

By executing the script, I got a restrictionfree shell.

{{<figure-dynamic
light-src="/images/publisher/free_shell_light.png"
dark-src="/images/publisher/free_shell_dark.png"
>}}

The content of `/opt`
After my shell was freed from the apparmor profile, I decided to redo some of my previous actions. For example searching for SUID's. The search didn't show any useful Information before. But now, without apparmor restrictions, there is a SUID file `/usr/sbin/run_container`

How do I know that, you ask?

Yes there is no symbolic link, which could be revealed by just `ls -la` it, but like in many reverse engineering CTF's, the command `strings` should not be overlooked!

{{<figure-dynamic
light-src="/images/publisher/strings_run_container_light.png"
dark-src="/images/publisher/strings_run_container_dark.png"
>}}

Now there all the needed things to get privilege escalation.

- A script file which can be rewritten by our user: `/opt/run_container`
- A executable which runs as root and calls our script: `/usr/sbin/run_container`

After that, the root flag can be found in the `/root`

Good luck! :)

---

{{<figure
src="/images/darksoulsMeme.png"
nozoom=true
>}}
