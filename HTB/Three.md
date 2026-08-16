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

