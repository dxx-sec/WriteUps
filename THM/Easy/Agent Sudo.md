# Agent Sudo
- THM -> 10.128.166.133
- nmap -sCV -Pn 10.128.166.133
- 21 ftp, 22 ssh, 80 http y miramos código fuente
- ffuf -u http://10.128.166.133/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt y posteriormnete medium ( solo index.php )
