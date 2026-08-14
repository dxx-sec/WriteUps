# Crocodile

## 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sCV -Pn 10.129.157.193
```

**Servicios identificados:**

* `21/tcp` → FTP

  * **vsftpd 3.0.3**
  * Permite **login anónimo**.
* `80/tcp` → HTTP

  * Servidor web **Apache**.

---

## 2. Enumeración

### FTP

El servicio FTP permite acceso anónimo:

```text
anonymous
```

Se puede acceder al servicio sin proporcionar credenciales válidas.

### Web

Realizamos fuzzing de directorios y archivos:

```bash
ffuf -u http://10.129.157.193/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php
```

**Archivos encontrados:**

* `config.php`
* `index.html`
* `login.php`

`config.php` no muestra información relevante directamente.

`login.php` contiene un **panel de autenticación**.

---

## 3. Explotación

### Acceso al panel

Utilizamos los listados disponibles para probar credenciales.

Con el usuario:

```text
admin
```

y una de las contraseñas encontradas en los listados, conseguimos autenticarnos correctamente.

---

## 4. Post-Explotación

* Acceso conseguido al panel web.
* Dentro del panel encontramos la **flag**.

---

## 5. Flags / Evidencias

* **Flag:** obtenida desde el panel de administración.

---

## Lessons Learned

* Comprobar siempre si FTP permite **acceso anónimo**.
* El fuzzing con extensiones permite descubrir archivos PHP que no aparecen con un listado básico.
* Los archivos como `config.php` pueden ser interesantes aunque no muestren información directamente.
* Los paneles de login deben probarse con las credenciales obtenidas durante la enumeración.
* Una contraseña encontrada en un recurso accesible puede reutilizarse para autenticarse en otros servicios.
