# Dancing

- Tiramos -> nmap -sCV -Pn 10.129.156.189
- 135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP

- SMB 445
- smbclient -L ip  para listar
- Enter y no tienen password configurada ->nmap nos decia Message signing enabled but not required
- Dentro del scaneo con smbclient ->
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default shar
IPC$            IPC       Remote IPC
WorkShares      Disk     (NO TIENE CONTRASEÑA) 

- Probamos una a una y workshares -> no tiene pw -> entramos al servicio smb con smbclient //Ip/workspaces -n ( no tiene user )
- navegamos con cd por dos users y vemos que james tine la flag. get flag.txt y listo :)
