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
- 
