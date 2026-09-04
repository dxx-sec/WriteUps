# Simple CTF 
- THM
- 10.128.169.184
- nmap -sCV -Pn 10.128.169.184
- 21,80,2222 ssh -> 7.2 enumeartionuser!!
- ftp-anon: Anonymous FTP login allowed (FTP code 230)
- ffuf -u http://10.128.169.184/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
- /simple/
- probamos de mientras a ver que hay en el servicio ftp
- 
