# Billing - THM
- 10.130.160.160
- nmap -sCV -Pn 10.130.160.160 -> 22,80,3306 mariadb -> mariaDB 10.3.23 or earlier,5038/tcp asterisk  
- accedemos web y te redirige a /mbilling
- fuzzeamos -> vemos robots.txt luuego / LICENSE y index.html y index.php -> ffuf -u http://10.130.160.160/mbilling/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt ffuf -u http://10.130.160.160/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- http://10.130.160.160/mbilling/index.php si entramos aquí vemos parametros de configuración
