# Dreaming
- THM
- 10.130.178.56  y nmap -sCV -Pn 10.130.178.56
- 22 y 80 http
- ❯ ffuf -u http://10.130.178.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt para ver que rutas hay en la web.
- encuentro con ❯ ffuf -u http://10.130.178.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -> /app 301 redirigido permanentemente y index.html
- EN app encuentro -> pluck-4.7.13/	2020-01-29 08:55 	-> http://10.130.178.56/app/pluck-4.7.13/?file=dreaming -> pruebo pero lo bloquea -> http://10.130.178.56/app/pluck-4.7.13/?file=../../../../etc/passwd
- ❯ ffuf -u http://10.130.178.56/app/pluck-4.7.13/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -> admin.php, data , docs, files, images,index.php, robots.txt
- 
