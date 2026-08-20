# Cap
- nmap -sCV -Pn -n  10.129.103.101
- Ftp -> puerto 21 22 -> ssh y servidore web -> 80 Gunicorn
- Accedemos a la web y vemos un panel que esta logeado Nathan.
- Vemos que hay varias rutas -> networkstatus ifconfig y un dashbord con datos.
- Procedemos a fuzzear por si nos dejamos rutas -> ffuf -u http://10.129.103.101/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -mc 200
- /ip,/netstat y /data lo visto.
- http://10.129.103.101/data/X podemos ir modificando y ver diferentes datos.
- Probamos bajando y llegamos al http://10.129.103.101/data/0 vemos más datos como Number of Packets
	72
Number of IP Packets
	69
Number of TCP Packets
	69
Number of UDP Packets
- Me descargo el archivo.
- Veo que es un archivo de trafico de wireshark procedo a abrirlo y analizarlo. -> wireshark &> /dev/null & disown abro en segundo plano y importo el archivo.
- 36	4.126500	192.168.196.1	192.168.196.16	FTP	69	Request: USER nathan -> pillo esto
- 40	5.424998	192.168.196.1	192.168.196.16	FTP	78	Request: PASS Buck3tH4TF0RM3! -> tenemos user:pw de ftp :)
- probamos ftp 10.129.103.101 -> logeamos con credenciales éxito -> ls -> pillamos la flag de user con get.
- Ahora voy a probar con ssh para ver si puedo logearme a la máquina -> Accedemos con la misma contraseña que ftp XD!!!!.
- id -> no veo grupo raro y pruebo sudo -l para ver si podemos ejecutar algo con sudo ( no hay nada que se pueda ejecutar)
- miramos SUIDs -> find / -perm -4000 -type f 2>/dev/null -> Identifico pkexec con suid miro su version -> pkexec version 0.105
- busco vulnerabilidades -> 
PolicyKit-1 0.105-31 - Privilege Escalation 2021-4034

export TERM=xterm-256color para que me pille la kitty
Seguimos los pasos de la vulnerabilidad creando los 3 archivos ejecutamos make all y nos crea dos archivos -> evil.so y exploit. Procedemos a ejecutar el binario de exploit ./Exploit
Comprobamos que somos root :)))

Pillo flag y listo (:
