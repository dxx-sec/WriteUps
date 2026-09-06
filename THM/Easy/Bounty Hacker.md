# Bounty Hacker
- THM
- 10.128.182.117
- nmap -sCV -Pn 10.128.182.117
- 21 ftp anonymous , 22ssh, 80http
- pillamos dos archivos con ftp anonymous
- Spike,Jet,Edward,Ed,Ein,Faye parece faye el que quiere acceder y dicen que puede
- ffuf -u http://10.128.182.117/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
