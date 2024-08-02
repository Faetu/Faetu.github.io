---
title: "THM - Publisher writeup"
date: 2024-07-25
image: /assets/images/publisher/publisher.png
tags:
  [
    tryhackme,
    writeups,
    linux,
    spip,
    apparmor,
    nmap,
    gobuster,
    CVE-2023-27372,
    shebang,
  ]
categories: [TryHackMe, Writeup]
redirect_from: /2024/07/25/publisher/
---

Hi,

This is my writeup about the [TryHackMe](https://tryhackme.com) box ["Publisher"](https://tryhackme.com/r/room/publisher).

## Information gathering

As always we have to know what is in front of us:

```bash
nmap -Pn TARGET_MACHINE_IP -oN ports && nmap -Pn -sC -sV -p $(grep -Po '.*(?=/tcp)' ports | tr '\n' ',') TARGET_MACHINE_IP -oN services
```

<!--more-->

This scan reveals only two ports:

![light mode only](/assets/images/publisher/nmap_scan_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/nmap_scan_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

I have checked the ssh service for password authentication and it accepts, but bruteforcing ssh is not attractive currently without knowing any usernames.
I also take a look at the website, without luck.

![website](/assets/images/publisher/publisher_website_dark.png){: .w-75 .rounded-10 w='1212' h='668' }

All links are empty, except those on "Related Blogs", but this is out of scope. Don't attack them!

Next I run gobuster against the machine.

![light mode only](/assets/images/publisher/gobuster_scan_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/gobuster_scan_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

And there is a new page to take a look at! As always, search for version number.

![light mode only](/assets/images/publisher/spip_version_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/spip_version_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

## Exploit SPIP

This version of spip is vulnerable to [CVE-2023-27372](https://www.exploit-db.com/exploits/51536).

![light mode only](/assets/images/publisher/searchsploit_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/searchsploit_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

I copied the script to my local directory and open it in nvim. The script is not very useful by default. I had to change one thing.

> If you got some error because of `urllib3<2`, then look at this [comment](https://github.com/psf/requests/issues/6443#issuecomment-1535667256) on github, it solved my problem with the requests module.
{: .prompt-info}

I've changed the script by adding this line to the `send_payload` function, right before the return command:

```python
print(r.content)
```


After that I just run the script as it was. But the result is not as handy as I wish for...

![light mode only](/assets/images/publisher/exploit_mess_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/exploit_mess_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

Playing around with [regex101](https://regex101.com), I was able to create a shell oneliner, which makes the output more handy or "humen readable".

```shell
python ./51536.py -u "http://$target/spip" -c "cat /home/think/user.txt" | ggrep -Po '(?<=value\\="s\\:[0-9]{2}:\\")[^";]*' | awk '{gsub(/\\\n/,"\n")}1'
```

Result:

![light mode only](/assets/images/publisher/exploit_clean_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/exploit_clean_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

This looks way better now! To get the first flag, take a look into the users home directory:

![light mode only](/assets/images/publisher/user_flag_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/user_flag_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

## SSH

To get ssh access, I just copied the ssh private key of user `think` to my local directory.

```shell
python ./51536.py -u "http://$target/spip" -c "cat /home/think/.ssh/id_rsa" | ggrep -Po '(?<=value\\="s\\:[0-9]{2}:\\")[^";]*' | awk '{gsub(/\\\n/,"\n")}1' > think_pk
```

And ssh into it!

![light mode only](/assets/images/publisher/ssh_access_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/ssh_access_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

## PrivEsc

While looking around I found something suspicious:

![light mode only](/assets/images/publisher/rbash_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/rbash_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

This is strange, because as `ls -la` shows, I should be able to read the content of the directory. This discovery was accidental because I was looking for a script or something similar.
But with that, I'm on the right track. We remember the instruction to the task:

> Attempts to escalate privileges using a custom binary are hindered by restricted access to critical system files and directories, necessitating a deeper exploration into the system's security profile to ultimately exploit a loophole that enables the execution of an unconfined bash shell and achieve privilege escalation.

Because the permissions from the file system doesn't match the observed behaviour, I decided to check for [ACL](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#acls) and [SELinux](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#selinux) and also for [AppArmor](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#apparmor).

![light mode only](/assets/images/publisher/apparmor_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/apparmor_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

AppArmor is enabled, this must be the reason for the strange behaviour. AppArmor policies lays in the `/etc/apparmor.d/`{: .filepath} directory.

One lookup on the directory reveals the problem. The shell of the user `think` is [ash](https://en.wikipedia.org/wiki/Almquist_shell), which has a profile on `/etc/apparmor.d/`{: .filepath}.

When looking at the profile for the ash shell, I observed a hole.

![light mode only](/assets/images/publisher/aa_policy_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/aa_policy_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

The missing wildcard's are the problem here. Without that, the deny rule only prevents me from changing a file with that name. But the content of the directory, is not affected by the deny rule.
With this in mind and the fact that only the ash shell is restricted by apparmor, the solution is obviously.

The trick is explained on hacktricks: [AppArmor Shebang Bypass](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/apparmor?ref=benheater.com#apparmor-shebang-bypass)



I created a file in the directory of `/dev/shm/`{: .filepath} which contains a bit of shell code.

```shell
#!/bin/bash
/bin/bash -pi
```

By executing the script, I got a restrictionfree shell.

![light mode only](/assets/images/publisher/free_shell_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/free_shell_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

The content of `/opt`{: .filepath} is now readable! And yes, this is a big thing because there is a shell script which can be modified by our user.
After my shell was freed from the apparmor profile, I decided to redo some of my previous actions. For example searching for SUID's. The search didn't show any useful Information before. But now, without apparmor restrictions, there is a SUID file `/usr/sbin/run_container`{: .filepath} which is directly linked to the `/opt/run_container`{: .filepath} shell script file.

How do I know that, you ask?

Yes there is no symbolic link, which could be revealed by just `ls -la` it, but like in many reverse engineering CTF's, the command `strings` should not be overlooked!

![light mode only](/assets/images/publisher/strings_run_container_light.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }
![dark mode only](/assets/images/publisher/strings_run_container_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

Now there all the needed things to get privilege escalation.

- A script file which can be rewritten by our user: `/opt/run_container`{: .filepath}
- A executable which runs as root and calls our script: `/usr/sbin/run_container`{: .filepath}

After that, the root flag can be found in the `/root`{: .filepath} directory.

Good luck! :)

---

![dark mode only](/assets/images/darksoulsMeme.png){: width="972" height="589" }
