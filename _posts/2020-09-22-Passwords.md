---
title: Password Cracking 
date: 2020-09-22 18:08:00 +/- 0000
categories: [Notes, Passwords]
tags: [Passwords, Hashcat, JTR, Crunch, Cewl, ssh2john, Hydra, zip2john]
headline: Comprehensive notes for password cracking and brute forcing.
image: /assets/img/Pass/cat_logo.png
---

Password cracking, brute forcing and wordlist creation are an important part of infosec, from doing CTF's as a hobby or as a professional pentester, having a solid methodology to giving yourself the best chance crack a password or hash is a vital skill. These notes are not set in stone and not all encompassing. If you see a mistake or have a suggestion for the page, please contact me on Discord at ***Buter#2867*** 

## Basic Everyday Cracking

Just so we have a hash to play with, I'll generate an md5 hash of the word 'passw0rd'.
```shell
echo -n "passw0rd" | md5sum > hash.txt
```

![make_hash](/assets/img/Pass/1_make_hash.png)

We know this is an md5 hash, but you'll always have to check the type of hash you are trying to crack. My tool for this is [hashid.py](https://github.com/psypanda/hashID). I like this tool because it does a good job of identifying the hash and will also give you the Hashcat and JTR code if you ask for it with -m and -j respectively.
```shell
./hashid.py -m -j 'bed128365216c019988915ed3add75fb'
```

![Hash_id](/assets/img/Pass/2_hashid.py.png)

Another method to quickly identify the hash I stole from Ippsec:
```shell
hashcat --example-hashes | grep '$2a$05' -B5
```

![Ippy](/assets/img/Pass/ippy.png)

We get a list of possible hashes so a bit of common sense comes into play, here. The first Hashcat code is for a raw md5, which is what we know it is, and it has a code of 0. 

![Cat](/assets/img/Pass/cat_logo.png)

We will now try to crack the hash with ***Hashcat*** by specifying the hash file and the wordlist we want to use, rockyou.txt in this case. I am using my Kali machine, so we need to use ***--force*** to force it to use the CPU and we'll use -a 0 for a straight attack.
```shell
hashcat --force -m 0 -a 0 hash.txt rockyou.txt
```

![Hashcat](/assets/img/Pass/3_hashcat.png)

After it shows us a couple of warnings about using --force, it takes a few seconds to sort it's life out and then starts the cracking, which takes it less than a second.

![John_logo](/assets/img/Pass/jtr_logo.png)
Now let's try the same attack with ***John
 The Ripper***.
```shell
john --wordlist=rockyou.txt --format=raw-md5 hash.txt
```

![JTR](/assets/img/Pass/jtr.png)

John cracked it in less than a second as well. It's worth noting that if you try to crack it again, you won't get the output as JTR will have stored the cracked hash in the pot file at **/root/.john/john.pot**. You will need to erase the file to crack it again, or cat the file to see the password.

## Hydra Attacks

![Hydra](/assets/img/Pass/hydra_logo.png)

I won't lie, Hydra is a finicky little fella, so getting the syntax right is vital and I certainly seem to get it wrong more often than I get it right on certain attacks. Note the use of upper/lower case; -p password vs. -P password.txt.

The -t flag specifies the number of threads you are running in the attack. For ssh I commonly use 4. For other services I will use trial and error to find a good number.

If the service you are attacking is running on a *non-standard* port, you must specify that with the -s flag. ex: ftp is running on port 1021, the end of the attack would look be **ftp -s 1021**

**Hydra Options:**

* -l Single username
* -L Username list
* -p Password
* -P Password list
* -t Threads
* -V Verbose output
* -f Stop when valid login found
* -s Specific port

**HTTP(s) Post Form**
```shell
hydra -l admin -P rockyou.txt $ip http-post-form '<Login Page>:<Request Body>:<Error Message>' -V
```

**FTP**
```shell
hydra -t 1 -l user -P rockyou.txt -vV $ip ftp
```
**SSH**
```shell
hydra -v -V -u -l user -P rockyou.txt -t 4 -u $ip ssh
```
**POP3**
```shell
hydra -l user -P rockyou.txt -f $ip pop3 -V -t 1
```
**Wordpress**
```shell
hydra -l user -P rockyou.txt $ip -V http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Location'
```
**WPscan for Wordpress**
```shell
wpscan --url $ip -U user -P rockyou.txt
```
**Windows RDP**
```shell
hydra -t 1 -V -f -l user -P rockyou.txt rdp://$ip
```
**SMB**
```shell
hydra -t 1 -V -f -l user -P rockyou.txt $ip smb
```
**401 Auth**
```shell
hydra -l user -P rockyou.txt $ip http-get /path
```
**SNMP**
```shell
hydra -P rockyou.txt -v $ip snmp
```
**MYSQL**
```shell
hydra -l user -P rockyou.txt $ip mysql -V -f
```
**VNC**
```shell
hydra -P rockyou.txt $ip vnc -V
```

## Zip Passwords

Zip files can have passwords set on them, but we have a way to crack those, too! We will use JTR for this as it seems to be a bit more forgiving, but first, we need to create a hash that John can understand and zip2john can do this for us.
```shell
zip2john tom.zip | cut -d ':' -f 2 > hash.txt
```
Then use john to crack the zip password.
```shell
john hash.txt --format=PKZIP --wordlist=/root/RockYou/rockyou.txt
```
![Zip](/assets/img/Pass/zippy.png)

Hashcat may also be used to crack the zip password, but the hash may need to be modified as per the example on the hashcat example page.
<https://hashcat.net/wiki/doku.php?id=example_hashes>

```shell
hashcat -a 0 -m 17200 hashes.txt rockyou.txt
```

A third, possibly more elegant, solution is fcrackzip. Simply install it with:
```shell
sudo apt install fcrackzip
```
Then run it. Here I will use the rockyou.txt wordlist again.
```shell
fcrackzip -D -p /root/RockYou/rockyou.txt file.zip
```

![Fcrack](/assets/img/Pass/fcrack.png)

## Linux Passwords

Linux passwords are kept in the /etc/shadow file, not the /etc/passwd file. To crack them easily, we need to "unshadow" them. For that we will need the passwd and the shadow file. I've created a user *unicorn* with password *password* to illustrate.
```shell
unshadow test_passwd.txt test_shadow.txt > test.txt
hashcat --force -m 1800 -a 0 test.txt rockyou.txt
```

![Shadow1](/assets/img/Pass/shadow1.png)
![Shadow2](/assets/img/Pass/shadow2.png)

## SSH Key Encryption

Getting an SSH key to a machine is always pretty cool. Well, until you try to use it and find out it's encrypted and you don't have the password. John The Ripper comes to our rescue again. Like a zip file, we need to get the file into a format that JTR can read, so we will use ssh2john.py.

Here we have recovered a private key, but it clearly says it's encrypted. If we try to use it, we will be asked for a password.

![SSH1](/assets/img/Pass/ssh1.png)

I can't remember where ssh2john lives, so I locate it and use it's full path from my RockYou directory. Once ssh2john has run, we can use JTR to crack the password as usual.
```shell
python /usr/share/john/ssh2john.py id_rsa > id_john
john id_john --wordlist=rockyou.txt
```

![SSH2](/assets/img/Pass/ssh2.png)

John cracks the password to ***superpassword*** so this key may now be used to login.

Top tip from Mr. Ippsec in his [Tenten video](https://www.youtube.com/watch?v=A4U3xiRWfsU) is to use [sshng2john](https://github.com/stricture/hashstack-server-plugin-jtr/blob/master/scrapers/sshng2john.py) if you are using JumboJohn to crack. 






