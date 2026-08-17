# Vaccine

10.129.163.110
nmap -sCV -Pn 10.129.163.110 -> 21 -> FTP // 22 -> Ssh // 80 -> HTTP
Tiene anonymous FTP LOGIN ALLOWED!!! -> nos traemos un .zip que está cifrado.
zip2jhon backup.zip -> pillamos el hash.txt y ahora lo desciframos
usamos john (johntheripper) para descifrar el hash -> y obtenemos la pw del .zip

trae un archivo .php y un .css -> nos interesa el index.php
sacamos una conexión y vemos user y contraseña -> admin  + 2cb42f8734ea607eefed3b70af13bbd3 (cifrado md5) -> La contraseña tiene que ser esto
echo '2cb42f8734ea607eefed3b70af13bbd3' > hash.txt -> guardamos el hash md5 raw y lo deshasehamos -> john hash.txt --format=Raw-MD5 -w=/usr/share/wordlists/rockyou.txt
Vamos a la web y probamos credenciales -> nos da el login  a http://10.129.163.110/dashboard.php


