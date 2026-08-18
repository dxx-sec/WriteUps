# Sequel

## 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -sCV -p- -Pn 10.129.157.177
```

El escaneo presenta problemas, por lo que realizamos un segundo escaneo:

```bash
nmap -Pn -T4 --max-retries 2 --host-timeout 5m 10.129.157.181
```

* Identificamos **MySQL/MariaDB** en el puerto `3306/tcp`.

---

## 2. Enumeración

### Conexión a MySQL

Probamos conectarnos como `root` sin contraseña:

```bash
mysql -u root -p -h 10.129.157.181 --skip-password
```

No conseguimos establecer la conexión.

Probamos utilizando el cliente de **MariaDB**:

```bash
mariadb -u root -h 10.129.157.181 -P 3306 --skip-password
```

Esta vez conseguimos acceder al gestor de bases de datos.

---

## 3. Explotación

Una vez dentro, enumeramos las bases de datos disponibles.

* Encontramos **4 bases de datos**.
* Seleccionamos la base de datos `htb`:

```sql
USE htb;
```

* Enumeramos sus tablas:

```sql
SHOW TABLES;
```

* Consultamos el contenido de las tablas mediante `SELECT`.

Encontramos una tabla `config` que contiene una columna llamada `flag`.

---

## 4. Post-Explotación

* Acceso conseguido al servicio **MariaDB/MySQL** mediante el usuario `root` sin contraseña.
* Enumeramos la base de datos `htb`.
* Localizamos la tabla `config`.
* Encontramos la columna `flag`.

---

## 5. Flags / Evidencias

* **Flag:** encontrada en la columna `flag` de la tabla `config`.

---

## Lessons Learned

* Si un cliente de MySQL presenta problemas, probar también con `mariadb`.
* El puerto `3306` es el puerto habitual de **MySQL/MariaDB**.
* Una vez dentro de una base de datos, el flujo básico de enumeración es:

  * `SHOW DATABASES;`
  * `USE <database>;`
  * `SHOW TABLES;`
  * `SELECT * FROM <table>;`
* Siempre comprobar si existen credenciales por defecto o accesos sin contraseña en servicios expuestos.
