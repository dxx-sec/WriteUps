# Sequel
- escaneamos puertos de la maquina -> nmap -sCV -p- -Pn 10.129.157.177
- nmap -Pn -T4 --max-retries 2 --host-timeout 5m 10.129.157.181 -> lanzamos este por que daba problemas nmap
- Identificamos el servicio mysql en el puerto 3306
- mysql -u root -p -h 10.129.157.181 --skip-password -> no conecta.
- mariadb -u root -h 10.129.157.181 -P 3306 --skip-password -> probamos con el gestor de bd mariadb
- Navegamos por la base de datos vemos que hay 4 entramos con use a htb vemos que tablas hay y mostramos todo de las tablas con select. vemos que hay una columna flag en config;
- 
