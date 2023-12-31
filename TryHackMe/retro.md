# Retro

Hi,

this is my Writeup for the [Retro](https://tryhackme.com/room/retro) box of [TryHackMe](https://tryhackme.com).
This box is the final room of the [Offensive Pentesting](https://tryhackme.com/paths) path.

## Information gathering

First I have to know what is in front of me. So I started the machine and after that start with nmap.
```bash
nmap -Pn -p- {{TARGET_MACHINE_IP}} -oN ports
```

That scan reveals two open ports:
- 80 which is running a webserver
- 3389 which is the rdp service

To get the services running, I use always this command:
```bash
nmap -Pn -sC -sV -p $(grep -Po '[0-9]*(?=/tcp)' ports | tr '\n' ', ') {{TARGET_MACHINE_IP}} -oN services
```
## A web server is running on the target. What is the hidden directory which the website lives on?
Tho answer this question, I used my favorit content discovery tool: [feroxbuster](https://github.com/epi052/feroxbuster).

```bash
feroxbuster -w /PATH/TO/SECLISTS/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://{{TARGET_MACHINE_IP}}
```

[feroxbuster result](thm_resources/SCR-20231231-iteb.png)


