# tomghost
Identify recent vulnerabilities to try exploit the system or read files that you should not have access to.

Author: [Stuxnet](https://tryhackme.com/p/stuxnet) \
Room Link: https://tryhackme.com/r/room/tomghost

## Enumeration
To start off, I ran an nmap scan using \
`namp -sV -F <ip addr>` 

![scan](https://github.com/Shibss1/TryHackMe/assets/88189483/22980762-eb8c-40ee-86b5-89d117760af1) 
There were see 3 notable services: 
1) OpenSSH 7.2.2
2) Apache Jserv (Protocol 1.3) (aka AJP)
3) Apache Tomcat 9.0.30

## Vulnerability
Doing a quick google search of exploits relating to AJP lead me to this [page](https://book.hacktricks.xyz/network-services-pentesting/8009-pentesting-apache-jserv-protocol-ajp).

A part of the page shows this:
> If the AJP port is exposed, Tomcat might be susceptible to the Ghostcat vulnerability

From this I was able to deduce that this machine was susceptible to [Ghostcat (CVE-2020-1938)](https://www.chaitin.cn/en/ghostcat).

## Exploitation
To exploit this vulnerability, I utilised [this exploit](https://github.com/xindongzhuaizhuai/CVE-2020-1938) to read the web.xml file, where it showed credentials for a user "skyfuck".

![ghostcat](https://github.com/Shibss1/TryHackMe/assets/88189483/8c5cf90e-27dd-4a33-920b-3b4f106c85fa)
Using the obtained username password, I SSH'ed into "skyfuck" and was greeted with 2 files upon enumerating the current directory.

![skyfuck](https://github.com/Shibss1/TryHackMe/assets/88189483/7fbf7ed3-6061-4464-b262-351d356e8b3f) \
The 2 files were
- credential.pgp - Encrypted PGP file
- tryhackme.asc - PGP Private Key

I then copied these files over to my local machine using scp so that I can decrypt them.

![scp](https://github.com/Shibss1/TryHackMe/assets/88189483/626f7d52-07f7-40df-8512-718beba001cd)

## Decrypting PGP file
Initially, I just tried to import the private key (.asc file). However, it prompted me for a passphrase and guessing it was getting me nowhere ☹️ 

I decided to search up if there was any way to crack the passphrase. And yes, there [was](https://blog.atucom.net/2015/08/cracking-gpg-key-passwords-using-john.html); using John The Ripper (JTR).

### gpg2john
I used this tool to help convert the private key into a hash readable by JTR \
`gpg2john tryhackme.asc > john.hash`

### JTR
I ran the hash file through JTR and finally acquired the passphrase for the PGP key \
`john --wordlist=/path/to/wordlist john.hash` 

![crackedPass](https://github.com/Shibss1/TryHackMe/assets/88189483/439e894f-c025-40e3-97d4-f88cc5bd656c)

### Decryption
Now with the passphrase, I was able to import the PGP key and use it to decrypt credential.pgp \
`gpg --import-key tryhackme.asc` - To import key \
`gpg -d credential.pgp` - To decrypt the file

![decrypted](https://github.com/Shibss1/TryHackMe/assets/88189483/c7d1e1b5-36ba-4cb9-b854-60e3d47ac7da)
This file displayed the credentials for the user "merlin".

The user skyfuck couldn't use `sudo` nor were there any PE-able SUID bits.

## Accessing merlin
With nothing else to do with skyfuck, I naturally proceeded to SSH into merlin using the freshly decrypted password.

Upon `ls`ing, user.txt was present and reading the file grants us the 1st flag.

![flag1](https://github.com/Shibss1/TryHackMe/assets/88189483/71879997-7655-4a20-bcbb-2e14b9181c44) \
THM{First_Flag}

## Privilege Escalation (PE)
The first thing I did was to run `sudo -l` to find out what commands merlin could run using sudo.

![sudoPE](https://github.com/Shibss1/TryHackMe/assets/88189483/104962d4-82ae-49e0-8bb4-b73cbb7a17fe) \
We find out that this user could use `sudo` to run `zip` as root. And it just so happens that we are able to use [that](https://gtfobins.github.io/gtfobins/zip/) to escalate our privileges.

![actualPE](https://github.com/Shibss1/TryHackMe/assets/88189483/b9315b44-7408-42ce-b4db-5e89ba0a4587) \
Now that we are root, let's list the /root folder. 

![flag2](https://github.com/Shibss1/TryHackMe/assets/88189483/0fa9410c-a3d2-4c26-b8f2-d07338317c92) \
We will be able to see a root.txt file and upon reading it, the second flag shows itself. \
THM{Second_Flag}

### Finding alternatives
I wanted to try and quickly find another method of PE. Thus I tried to find an SUID bit \
`find / -perm /4000 -type f 2>/dev/null` 

`sudo` showed up as a possible PE vector. \
However, upon running it, it prompted a notice saying that merlin had no permission to run that command using sudo 🤷🏻‍♂️

# END
Date first written: 24 March 2024
