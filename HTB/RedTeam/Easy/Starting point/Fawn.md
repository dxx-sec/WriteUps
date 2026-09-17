# 🖥️ Fawn

**Plataforma:** HTB
**Dificultad:** Easy

---

## 1. 🔎 Reconocimiento

* Se realiza un escaneo con Nmap.

---

## 2. 🔍 Enumeración

### Puertos

```bash
nmap -sCV -Pn [IP]
```

| Puerto | Servicio | Versión      |
| ------ | -------- | ------------ |
| 21     | FTP      | vsftpd 3.0.3 |

### Otros hallazgos

* Nmap indica que **Anonymous Login** está permitido.

---

## 3. 💥 Explotación

**Vulnerabilidad:** FTP Anonymous Login

```bash
ftp [IP]
```

Se accede utilizando el usuario:

```text
anonymous
```

Una vez dentro, se utiliza `get` para descargar el archivo que contiene la flag.

```bash
get [archivo]
```

**Acceso obtenido:** Acceso al servidor FTP.

---

## 4. 🐚 Escalada de privilegios

No es necesaria. La flag se encuentra directamente en el servidor FTP.

---

## 5. 📄 Reporte

### Vulnerabilidades

* **FTP Anonymous Login** → Permite acceder al servidor sin credenciales.

### Remediación

* Deshabilitar el acceso anónimo.
* Utilizar autenticación mediante credenciales.

---

## 🧠 Lessons Learned

* **Aprendí:** FTP, Anonymous Login.
* **Herramientas:** Nmap, FTP.
* **Comandos nuevos:** `ftp`, `get`.
