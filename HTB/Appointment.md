# Appointment
- Escaneo -> nmap -sCV -p- -Pn 10.129.157.169
- 
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Login

- Servidor apache web en p80
- Muestra un panel de login
- Fuzzeamos la web -> ffuf -u http://10.129.157.169/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- Encontramos index.php (es el login) nada más.
- Con SQLi accedemos mediante user -> admin'# -> esto significa cierra la consulta y el resto leelo como un comentario ( la pw )
- 
