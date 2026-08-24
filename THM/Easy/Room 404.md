# Room 404
- nmap -sCV -Pn 10.129.172.90 y ping para comprobar conectividad
- puerto 22 ssh y 8080 servidor web Werkzeug -> python3
- 8080/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.12.3)
| http-git: 
|   10.129.172.90:8080/.git/
|     Git repository found!

Nos indican los scripts de namp que ha encontrado un repositorio git.
http://10.129.172.90:8080/ accedemos y vemos una web del hotel. Si le das a reserve te lleva a http://10.129.172.90:8080/booking y te da error 404.
ffuf -u http://10.129.172.90:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt ->.git/head/logs/index/config.
http://10.129.172.90:8080/.git entramos aqui

http://10.129.172.90:8080/.git/config y me lo descargo el archivo leemos y no encuentro gran cosa.
git-dumper http://10.129.172.90:8080/.git/ repo con -> git-dumper.
entramoss al repo y vemos en el código la apiruta api :)
y leemoss el readme y vemosss la flag.

