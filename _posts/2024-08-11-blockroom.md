---
title: "THM - Block"
date: 2024-08-11
image: /assets/images/blockroom/blockroom.png
tags:
  [tryhackme, writeups, windows, wireshark, ntlm, smb, hashcat, pypykatz]
categories: [TryHackMe, Writeup, CTF, Medium]
redirect_from: /2024/08/11/blockroom/
---

Hi,

This is my writeup about the [TryHackMe](https://tryhackme.com) medium room ["Block"](https://tryhackme.com/r/room/blockroom).

## 1. What is the username of the first person who accessed our server?

On this stage, I ignored the `lsass.DMP`{: .filepath} file and focuses on the pcap file.
To get the answer to this question, I just had to open the pcap file with [wireshark](https://www.wireshark.org):

![first username light](/assets/images/blockroom/first_username_light.png){: .light .shadow .w-75}
![first username dark](/assets/images/blockroom/first_username_dark.png){: .dark .shadow .w-75}

At this moment all should know, that we are working with a windows machine. More specifically, with a tcpdump of a Windows machine. So you should be familiar with ntlm. If not, read this [article on NTLMv2](https://book.hacktricks.xyz/windows-hardening/ntlm) and/or do the [Windows Fundamentaks](https://tryhackme.com/module/windows-fundamentals) + [Windows Exploitation Basics](https://tryhackme.com/module/hacking-windows-1) learning module on [TryHackMe](https://tryhackme.com).

## 2. What is the password of the user in question 1?

This question is simple: Bruteforce!
But, what do we need to bruteforce? This is the real question here. Because to be able to bruteforce `NTLMv2`, there are several parts we need to obtain.

1. Username
2. Domain
3. NTLM Server Challenge
4. NTProofStr aka HMAC-MD5
5. NTLMv2 Response

All those information have to be joined together and only separated by `:`, except the separation between username and domain, there has to be two colons.

Take a look at the skeleton of our hash:
```
username::domain:serverChallenge:NTProofStr:NTLMv2Response
```
{: .nolineno}
Without leading spaces, tabs and without spaces, tabs or newlines at the end!

To obtain every part, the `traffic.pcapng`{: .filepath} is all we need.

### 2.1 Domain

The username is known from the previous question. And also the domain! As can be seen on the screenshot of the previous question, the domain is the `WORKGROUP` right before the username.

### 2.2 NTLM Server Challenge

While the username and domain can be found on frame 11, the server challenge is one frame before/above on frame 10.

![server challenge light](/assets/images/blockroom/server_challenge_light.png){: .light .shadow .w-75}
![server challenge dark](/assets/images/blockroom/server_challenge_dark.png){: .dark .shadow .w-75}

Just right click on that line and select `copy`/`value`.

### 2.3 NTProofStr / HMAC-MD5

If you read somewhere about HMAC-MD5 then this is the NTProofStr on wireshark.
To obtain this, go back to frame 11 where the username and domain were found and search for it.

![NTProofStr light](/assets/images/blockroom/ntproofstr_light.png){: .light .shadow .w-75}
![NTProofStr dark](/assets/images/blockroom/ntproofstr_dark.png){: .dark .shadow .w-75}

### 2.4 NTLMv2 Response

This one is also tricky, because the response can be copied one line above the NTProofStr line from before, but it has the NTProofStr merged into it. Gladly it is at the beginning of the response string.

Remove it manually or use a shell. As I use [fish](https://fishshell.com) this command will do the job:
```bash
echo (string replace -r '^NTProofStr' '' "NTLMv2Response")
```
{: .nolineno}
For Bash:
```bash
echo "${'NTProofStr'#'NTLMv2Response'}"
```
{: .nolineno}

### 2.5 Get the password for mrealman

The bruteforce will not take much time, when using [rockyou.txt](https://github.com/praetorian-inc/Hob0Rules/blob/master/wordlists/rockyou.txt.gz) wordlist.

```bash
hashcat -m 5600 -a 0 mrealman_hash /usr/share/SecLists/rockyou.txt -d 2
```
{: .nolineno}
![Bruteforce mrealman light](/assets/images/blockroom/bruteforce_mrealman_light.png){: .light .shadow .w-75}
![Bruteforce mrealman dark](/assets/images/blockroom/bruteforce_mrealman_dark.png){: .dark .shadow .w-75}



## 3. What is the flag that the first user got access to?

The flag was clearly in a file saved and the user download it over smb. To get the file content, the SMB-encrypted frame must be decrypted:

![Encrypted SMB3](/assets/images/blockroom/encrypted_smb.png){: .w-100 .shadow }

To decrypt `smb3` traffic, two things are needed.

1. Session ID as `HEX Stream`
2. Session Key

### 3.1 Session ID

The session id can be obtained from the frame 10:

![Session ID light](/assets/images/blockroom/session_id_light.png){: .light .shadow .w-75}
![Session ID dark](/assets/images/blockroom/session_id_dark.png){: .dark .shadow .w-75}

### 3.2 Session Key

To get the session key more information are needed:

1. Username -> Obtained in question 1.
2. Domain -> Obtained in question 2.
3. Password -> Obtained in question 2.
4. NTProofStr -> Obtained in question 2.
5. Encrypted Session Key

The encrypted session key can be found in frame 11:

![Encrypted Session Key light](/assets/images/blockroom/encrypted_session_key_light.png){: .shadow .w-75 .light}
![Encrypted Session Key dark](/assets/images/blockroom/encrypted_session_key_dark.png){: .shadow .w-75 .dark}

Thanks to [Khris Tolbert](https://medium.com/@khristopher.tolbert) for his article ["Decrypting SMB3 Traffic with just a PCAP? Absolutely (maybe.)"](https://medium.com/maverislabs/decrypting-smb3-traffic-with-just-a-pcap-absolutely-maybe-712ed23ff6a2), where he explains all the steps in good detail.

The last piece to get the session key from all those information, is the python script, which Khris have wrote.
He wrote it in python2, if you are using python2 then get it from the article. I rewrote it for Python 3 because I don't use python2 anymore.

<script src="https://gist.github.com/Faetu/35ff929e64a37980021f19f083939b7e.js"></script>

When running the script with all information provided, it prints out the session key.

### 3.3 Decrypting SMB2

I added the session id with the session key into wireshark (Preferences/Protocol/SMB2):

![Decrypt SMB2 light](/assets/images/blockroom/decrypt_smb2_light.png){: .light .w-75}
![Decrypt SMB2 dark](/assets/images/blockroom/decrypt_smb2_dark.png){: .dark .w-75}

After that I lookup all now decrypted smb3 frames. The file which contains the flag, can be found on frame 54:

![First Flag light](/assets/images/blockroom/first_flag_light.png){: .light .shadow .w-75}
![First Flag dark](/assets/images/blockroom/first_flag_dark.png){: .dark .shadow .w-75}

Or a much simpler way:


![Export files light](/assets/images/blockroom/export_objects_light.png){: .light  .w-100}
![Export files dark](/assets/images/blockroom/export_objects_dark.png){: .dark  .w-100}

The flag lies inside of the `clients156.csv`{: .filepath} file.

## 4. What is the username of the second person who accessed our server?

As easy as question 1:

![Second username light](/assets/images/blockroom/second_username_light.png){: .light .w-75 .shadow}
![Second username dark](/assets/images/blockroom/second_username_dark.png){: .dark .w-75 .shadow}


## 5. What is the hash of the user in question 4?

First of: The `traffic.pcapng`{: .filepath} doesn't contain the hashes. To get hashes, I had to look into the second file.
First I decided to use [Ghidra](https://ghidra-sre.org), a reverse engineering tool developed by the NSA's research directorate. But it takes to long and was not so easy to obtain hashes.
After a short search, I came up with [pypykatz](https://github.com/skelsec/pypykatz). Perfect fitting into my needs, because to run [mimikatz](https://github.com/ParrotSec/mimikatz) I had to run a windows VM.

![NTLM Hash of eshellstrop light](/assets/images/blockroom/ntlm_hash_light.png){: .light .shadow .w-75}
![NTLM Hash of eshellstrop dark](/assets/images/blockroom/ntlm_hash_dark.png){: .dark .shadow .w-75}


## 6. What is the flag that the second user got access to?

This one takes me hours until I got it. I tried every wordlist on `SecLists/Passwords/`{: .filepath} + `rockyou.txt`{: .filepath} until I gave up on the idea to bruteforce the password of user `eshellstrop`.

At that moment, I tried to get the password by using encrypted smb3 blobs and all other information available at that moment. The tool I tried, was [dpapick3](https://pypi.org/project/dpapick3/). After hours of try and errors, I gave up and took a day break.
This break saved me hours of frustration, because I came up with the idea to reread everything up to that point. I also reread the article mentioned in 3.2 and saw something, which solved my problem quickly:

![Reread article light](/assets/images/blockroom/reread_light.png){: .light .shadow .w-75}
![Reread article dark](/assets/images/blockroom/reread_dark.png){: .dark .shadow .w-75}

So the script was not written to support ntlm hash beside plain passwords, and because of that, I missed this little phrase.
Now, beside I rewrote the script from python2 to python3, I also extended it a bit to support NTLM hashes, by adding those options and change some lines:

```python
group = parser.add_mutually_exclusive_group(required=True)
group.add_argument("-p", "--password", help="Plain password of User")
group.add_argument("-a", "--ntlmhash", help="NTLM Hash of User")
```

Now I executed the script with all needed information which can be found in the `traffic.pcapng`{: .filepath} and instead of the password, I used the `-a ESHELLSTROP_NTLM_HASH` option.

After I got the session key, I added it to Wireshark as in 3.3 for the first user and was able to obtain the `clients978.csv`{: .filepath} file, which contains the second flag.


Good luck! :)

---

![dark mode only](/assets/images/darksoulsMeme.png){: width="972" height="589" }
