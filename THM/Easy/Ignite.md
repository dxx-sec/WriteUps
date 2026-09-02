# Ignite
- THM
- 10.130.175.229
- nmap -sCV -Pn 10.130.175.229 -> Solo puerto http 80 abierto :)
- Fuel CMS -> Version 1.4 identificamos CMS.
- Fuzzeamos -> ffuf -u http://10.130.175.229/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- /0 , /home, /index , /index.php , /offline y /robots.txt -> encuentro Disallow: /fuel/ -> http://10.130.175.229/fuel/login/5a6e566c6243396b59584e6f596d3968636d513d -> redirige a un panel de login.
- 
