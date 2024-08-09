---
title: "The Return of the Yeti - Flag 1 Room"
date: 2023-12-28
image: /assets/images//aoc2023/yeti_banner_sq1.png
tags: [tryhackme, writeups, wireshark]
categories: [TryHackMe, Writeup, Event]
redirect_from: /2023/12/28/AOC-2023-SideQuest-Flag-1/
---

Hi!

Now in this room, you have to use [Wireshark](https://www.wireshark.org/), a network sniffing tool. If you never used it before, here are two rooms on thm to visit before:

- [Wireshark room on thm](https://tryhackme.com/room/wireshark) (learning)
- [Overpass2Hacked room on thm](https://tryhackme.com/room/overpass2hacked)
  (practice)

Or if you're looking for a real world Threat Hunting challenge you can check out
[Case: 001 PCAP Analysis](https://dfirmadness.com/case-001-pcap-analysis/) by
DFIR Madness.

Let's not waste time and dive into the first question!

### What's the name of the WiFi network in the PCAP?

This one is easy because when you open the .pcapng file, it is the first you
see:

![SSID_Wireshark.png](/assets/images/aoc2023/SSID_Wireshark.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

Nothing more to say about this, lets move on to the next question.

### What's the password to access the WiFi network?

This one took me some while to find out. After wasting some time looking for
cleartext hints, I filtered for
«[eapol](https://www.wireshark.org/docs/dfref/e/eapol.html)» in wireshark and
saw that there were two times where a handshake happens:

![EAPOL_Wireshark.png](/assets/images/aoc2023/EAPOL_Wireshark.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

So this means, we should be able to crack the wpa-pwd! But to do this, two tools
are needed:

- [Hashcat](https://hashcat.net/hashcat/)
- [hcxtools](https://github.com/ZerBea/hcxtools) - With this tool, the pcapng
  file can be converted to a hash which can be used in hashcat.

So the first step was to get the hash file for hashcat:

```bash
hcxpcapngtool VanSpy.pcapng -o wifihash
```

After that just run hashcat on it with rockyou wordlist:

```bash
hashcat -m 22000 -a 0 wifihash PATH/TO/rockyou.txt
```

**Explanation:**

- The «-m 22000» is the hash-type. 22000 is the code for
  «WPA-PBKDF2-PMKID+EAPOL». If you don't know which hash-type you have to use,
  then just run hashcat without providing a hash-type:

![Hash_Detection.png](/assets/images/aoc2023/Hash_Detection.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

- The «-a 0» flag is for the attack mode. For bruteforce (or straight) mode you
  have to use the 0 code.
- wifihash is the hash file we generated with hcxpcaptools
- rockyou.txt is the wordlist used to bruteforce the wpa-pwd

Hashcat should not take much time to crack it and then you see this:

![Cracking_Result.png](/assets/images/aoc2023/Cracking_Result.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

At the end of the marked lines, you should see the cracked wpa-pwd.

### What suspicious tool is used by the attacker to extract a juicy file from the server?

The information asked in this question is not readable for us currently. This is
also a point where i struggled a lot because of the mentality I showed in this
sidequest. It took very long for me to not see this challenge as a
walkthrough/tutorial but as a scenario and in a real world scenario every new
information would mark a new «Startpoint» to investigate. So with this mindset,
we know what to do next. Adding the wpa-pwd to the wireshark «Decryption keys»
to get some readable logs from the pcapng file!

To do this, go to «Preferences» then «Protocols».

![Wireshark_IEEE.png](/assets/images/aoc2023/Wireshark_IEEE.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

1. Select the «IEEE 802.11» protocol
2. Open the Decryption keys window

![Decrypt_Key_Wireshark.png](/assets/images/aoc2023/Decrypt_Key_Wireshark.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

Now add a new key and select «wpa-pwd» and paste the cracked wpa-pwd to the key
field.

After this step we finally see some decrypted data. Because the question is
about a «juicy file» I started my search by filtering for
[frame length](https://www.wireshark.org/docs/dfref/f/frame.html).

![FrameLen_Wireshark.png](/assets/images/aoc2023/FrameLen_Wireshark.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

And as some novel writers recommend, I start my search at the end.

![start_end.png](/assets/images/aoc2023/start_end.png){: .w-75 w='1212' h='668' }

At the end there were not much entries which have useful information so I
decided to scroll fast upward until I see some «strange» frame length.

![Strange_Length.png](/assets/images/aoc2023/Strange_Length.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

And there it was! Some over 500 length frames got my attention. So I decided to
use more filter, just to make my life easier I added the tcp filter beside the
frame.len filter.

![TCP_Filter.png](/assets/images/aoc2023/TCP_Filter.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

By checking each entry one by one, one showed up to look like a output of
windows cmd or powershell. To read it easier, there are several methods. I just
copy it as base64 and paste it to my terminal:

![B64_Export.png](/assets/images/aoc2023/B64_Export.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

Then just decode it with base64:

![B64_Decoded.png](/assets/images/aoc2023/B64_Decoded.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

By checking more entries, we find the application used:

![Found_App.png](/assets/images/aoc2023/Found_App.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

### What is the case number assigned by the CyberPolice to the issues reported by McSkidy?

To answer this question we need to decrypt some more of the pcapng file. By
finding the application used to extract the so called «juicy file» we find out
that it was a pfx file, which was extracted. To our luck, the attacker has
converted the pfx file to base64:

![B64_PFX.png](/assets/images/aoc2023/B64_PFX.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

The base64 output is split to the three entries (marked violet) to get it, I
copied every entry to base64 and decode it on my terminal. Then joined the three
parts together.

After that the file which contains the base64 can be decoded to the pfx file:

```bash
base64 -d -i MY_BASE64_FILE -o extracted.pfx
```

By joining the base64 parts together some errors could happen. For you to check
if you have harvest the pfx file correctly and it is not in some way damaged,
here the sha256sum of my file:

```
70a20f079ad5bbbc8d479f056f5de093ba7260d6a374842e0803bdb70dec52c1
```

You can check yours with this:

```bash
echo "70a20f079ad5bbbc8d479f056f5de093ba7260d6a374842e0803bdb70dec52c1 YOUR_FILE.PFX" | sha256sum -c
```

_Source:
https://superuser.com/questions/1312740/how-to-take-sha256sum-of-file-and-compare-to-check-in-one-line_

And as we used the wpa-pwd to decrypt further through the pcapng file, we also
want to use this pfx file to decrypt our tls entries. But the file is password
protected!

I waste more than five hours on trying to crack the password without success.
This happens because I ignored the first rule of cracking:

> Check default passwords first!

Yes, we know the tool which was used to extract the pfx file and so, we should
know the default password which is used by the tool.

![TLS_Decrypt.png](/assets/images/aoc2023/TLS_Decrypt.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

With this the whole rdp session is decrypted now. Now to make my life again
easier I changed the filter to rdp and added the attacker ip as the source ip.

```
frame.len > 0 and rdp and ip.src == 10.1.1.1
```

But there were still to many entries to check them all. I thought about it and
there was one filter, which I saw pop up when entering «rdp» as filter:
[rdp_cliprdr](https://www.wireshark.org/docs/dfref/r/rdp_cliprdr.html)

I tried it and the remaining entries were manageable. This way you can find the
issue number but also the flag:

![Issue_Flag.png](/assets/images/aoc2023/Issue_Flag.png){: .w-75 .shadow .rounded-10 w='1212' h='668' }

Don't forget to add the flag to the main room of the side quest!

![darksoulsMeme.png](/assets/images/darksoulsMeme.png){: width="972" height="589" }
