+++
title = "THM - Intermediate Nmap"
date = "2025-09-05T15:00:30+02:00"
tags = ["tryhackme", "nmap", "netcat", "sudo"]
aliases = []
categories = ["Pentesting", "Writeup", "CTF"]
showTableOfContents = true
showHeadingAnchors = true
heroStyle = "background"
showHero = true
+++

Hi,

This is my writeup about the [TryHackMe](https://tryhackme.com) box ["Intermediate Nmap"](https://tryhackme.com/room/intermediatenmap).

## Information gathering

As always, lets start by scanning ports and services:
```bash
nmap -Pn TARGET_MACHINE_IP -oN ports && nmap -Pn -sC -sV -p $(grep -Po '.*(?=/tcp)' ports | tr '\n' ',') TARGET_MACHINE_IP -oN services
```

That scan reveals three open ports:

- 22 which is running ssh
- 2222 which is running EtherNetIP-1
- 31337 which is running Elite

## Credentials

As the Room description suggests, Netcat plays an important role here. So, I decided to just play around and try connecting to the high port:

```bash
nc $target 31337
```

{{<figure-dynamic
light-src="/images/intermediatenmap/light/the_hint.png"
dark-src="/images/intermediatenmap/dark/the_hint.png"
>}}

I mean... ye, this room is labeled as `easy`, but come on, this feels like a joke.
"Anyways, the cool part will be afterwards" I thought.

## Afterwards

With those credentials, I was able to connect over ssh – but only on port `22`.
Aaaaaand... it's done.

{{<figure-dynamic
light-src="/images/intermediatenmap/light/flag.png"
dark-src="/images/intermediatenmap/dark/flag.png"
>}}

You're seeing it right, the flag has read permissions for `others`, which includes our user.
So technically, the room ends here with this flag.

Now, you may ask *"But why write a writeup for such a disappointing room?"*

Because, my dear reader, I didn't stop there. It felt so unsatisfying that I just had to go for root!

## The Cool Part

I started by looking for `suid` binaries and checked `sudo -l`.
Our user *ubuntu* isn't allowed to use sudo on this machine.
```bash
find / -perm -4000 2>/dev/null
```
No exploitable `suid` binaries showed up.

Next I searched for cron jobs or writable files that might help – no luck.
The running processes caught my attention:
{{<figure-dynamic 
light-src="/images/intermediatenmap/light/dockerfiles.png"
dark-src="/images/intermediatenmap/dark/dockerfiles.png"
>}}
I wasted nearly two hours chasing this death end.

I was about to give up when I had an idea:
*"I may not run any command with sudo, but is sudo itself vulnerable?"*

The target machine was running sudo version `1.8.31`, which is vulnarable to privilege escalation:
{{<figure-dynamic
light-src="/images/intermediatenmap/light/searchsploit.png"
dark-src="/images/intermediatenmap/dark/searchsploit.png"
>}}
As you can see, my search string was `sudo 1.8.` instead of `sudo 1.8.31`. That's because vulnerabilities often apply to a whole range of versions, not just one specific release.

Unfortunately, this exploit didn't work. I hadn't noticed the dependency: the user must be able to run sudoedit as root.

So, I checked the kernel version with `uname -a` and searched for kernel exploits:
```bash
searchsploit -t linux kernel 5.1 linux local
```
I had to include all those keywords, because otherwise there were just too many results.
One exploit caught my eye, it matched the kernel version of the TARGET_MACHINE_IP: [DirtyPipe](https://www.exploit-db.com/exploits/50808)

To use this exploit, I needed any binary with `suid` flag. So I re-run the search for `suid` and picked the sudo binary.
{{<figure-dynamic
light-src="/images/intermediatenmap/light/dirtypipe.png"
dark-src="/images/intermediatenmap/dark/dirtypipe.png"
>}}
Yes! Root access achieved!
Sadly, there is no hidden easter egg or something else. But I needed this kick after that unsatifying task.

Good luck! :)

---

{{<figure
src="/images/darksoulsMeme.png"
nozoom=true
>}}
