# Connected
- htb
- ping -c1 10.129.112.183 para ver conecntividad 1 intento.
- nmap -sCV -Pn 10.129.112.183 para ver que puertos tenemos que hay.
- 22 -> ssh / 80 -> http / 443 -> https
- Si entramos mediante http -> http://connected.htb/ redirige a esto, vamoss a añadirlo a nuestro dns local de la máquina para que resuelva el dns.
- sudo nano /etc/hosts -> 10.129.112.183 connected.htb
- 
