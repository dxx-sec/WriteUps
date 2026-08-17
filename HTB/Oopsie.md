# Oopsie.md
- nmap -sCV -p- -Pn 10.129.163.38 -> para ver que tenemos
- Puerto 22 -> ssh 80 -> http apache
- Vemos una página -> fuzeamos -> ffuf -u http://10.129.163.38/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- - solo encontramos index.php -> contacto dominio -> @megacorp.com
  - Vamos a utilizar otra wordlist -> No encontramos nada, nos ponemo s aivnestigar el código fuente y scripts y vemos que hay un login /cdn-cgi/login/script.js
  - probamos a acceder a http://10.129.163.38/cdn-cgi/login/ y vemos un panel de login -> podemos acceder como invitado.

- guest	guest@megacorp.com -> cuenta de invitado.
- http://10.129.163.38/cdn-cgi/login/admin.php?content=accounts&id=2 -> podemos modificar el id -> y ver que existe la cuenta admin@megacorp.com accesID 34322
- Cambiamos la cookie de user a la de admin con el accesID
