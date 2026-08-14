# Crocodile
- nmap -sCV -Pn 10.129.157.193
- 21 ftp -> anonymous login allowed - vsftpd 3.0.3
- 80 -> web hhtp apache -> fuzeamos
- ffuf -u http://10.129.157.193/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php
- filtrando por archivos -> config.php,index.hmtl,login.php.
- en config.php no vemos nada pero en login si vemos un panel de logeo.
- Podemos probar con los listados el logeo.
- COn admin y probando contraseñas accedemos al panel y pillamos la flag en el panel
