# Room 404 — THM

**Plataforma:** TryHackMe
**Dificultad:** Fácil  
**Categoría:** Web / Git / Information Disclosure

## 1. Reconocimiento

### Nmap

- Primero comprobamos la conectividad con el objetivo:

```bash
ping -c 4 10.129.172.90
```

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.129.172.90
```

- Encontramos:

- `22/tcp` → SSH
- `8080/tcp` → HTTP — Werkzeug 3.0.1 (Python 3.12.3)

- Nmap también nos muestra algo especialmente interesante:

```text
http-git:
10.129.172.90:8080/.git/
  Git repository found!
```

- Esto indica que el directorio `.git` del repositorio está expuesto públicamente mediante el servidor web.

## 2. Enumeración

### Enumeración web

- Accedemos a:

```text
http://10.129.172.90:8080/
```

- Encontramos una página web relacionada con un hotel.

- La opción `Reserve` nos lleva a:

```text
http://10.129.172.90:8080/booking
```

- Esta ruta devuelve un error `404 Not Found`.

### Fuzzing

- Utilizamos `ffuf` para buscar rutas y archivos ocultos:

```bash
ffuf -u http://10.129.172.90:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

- Encontramos varios recursos relacionados con el repositorio Git:

```text
.git
.git/head
.git/logs
.git/index
.git/config
```

- Accedemos directamente al directorio:

```text
http://10.129.172.90:8080/.git/
```

- También podemos consultar el archivo de configuración:

```text
http://10.129.172.90:8080/.git/config
```

- Descargamos el archivo y lo revisamos, pero no encontramos información especialmente relevante.

## 3. Explotación

### Git Repository Disclosure

- El problema principal es que el directorio `.git` está expuesto públicamente.

- Esto significa que podemos intentar recuperar el repositorio completo y acceder al historial y al código fuente de la aplicación.

- Utilizamos `git-dumper` para descargar el repositorio:

```bash
git-dumper http://10.129.172.90:8080/.git/ repo
```

- Esto recupera el contenido del repositorio en el directorio:

```text
repo
```

- Entramos en el repositorio:

```bash
cd repo
```

- Revisamos los archivos del proyecto:

```bash
ls -la
```

- Al analizar el código encontramos información relacionada con una ruta de la API.

- También revisamos el `README` del repositorio:

```bash
cat README.md
```

- En el `README` encontramos la información necesaria para obtener la flag de la máquina.

## 4. Escalada de privilegios

- En esta máquina no necesitamos realizar una escalada de privilegios.

- El objetivo se puede completar mediante la exposición del repositorio Git y la información almacenada en el código/README.

## 5. Reporte

- La máquina **Room 404** expone un servidor web Werkzeug en el puerto `8080`.
- Durante el reconocimiento, Nmap detecta que existe un repositorio Git accesible públicamente mediante `/.git/`.
- Mediante `ffuf` confirmamos la existencia de diferentes archivos y directorios pertenecientes al repositorio.
- Intentamos revisar `.git/config`, pero no obtenemos información relevante.
- Utilizamos `git-dumper` para recuperar el repositorio completo.
- Una vez descargado, analizamos el código fuente y encontramos información relacionada con una ruta de la API.
- También revisamos el `README.md`, donde encontramos la información necesaria para obtener la flag.
- No fue necesario realizar escalada de privilegios.

### Cadena de ataque

```text
Nmap
 ↓
HTTP :8080
 ↓
/.git/
 ↓
Repositorio Git expuesto
 ↓
ffuf
 ↓
git-dumper
 ↓
Código fuente
 ↓
API / README.md
 ↓
Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Repositorio `.git` expuesto | Alta | Permite recuperar el código fuente y el historial del proyecto |
| Información sensible en el repositorio | Alta | Puede revelar rutas, endpoints y otros datos internos |

### Mitigaciones

- No exponer el directorio `.git` desde el servidor web.
- Configurar correctamente el servidor para bloquear el acceso a `/.git/`.
- No almacenar secretos o información sensible dentro del repositorio.
- Revisar el contenido del historial Git antes de desplegar una aplicación.
- Separar correctamente el código fuente del contenido público del servidor web.

## 🧠 Lessons Learned

- Nmap puede detectar automáticamente repositorios Git expuestos mediante `http-git`.
- Un directorio `/.git/` accesible desde Internet puede permitir recuperar el repositorio completo.
- `ffuf` ayuda a confirmar la existencia de archivos y directorios ocultos.
- `git-dumper` permite reconstruir un repositorio Git expuesto.
- El código fuente y los archivos `README` pueden contener información importante para entender la aplicación.
- Cuando encontramos un `.git` expuesto, siempre debemos investigar tanto el código actual como el historial del repositorio.
