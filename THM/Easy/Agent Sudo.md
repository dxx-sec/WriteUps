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
