# 🖥️ Dancing

**Plataforma:** HTB
**Dificultad:** Easy

---

## 1. 🔎 Reconocimiento

Se realiza un escaneo con Nmap:

```bash
nmap -sCV -Pn [IP]
```

---

## 2. 🔍 Enumeración

### Puertos

| Puerto | Servicio | Versión                   |
| ------ | -------- | ------------------------- |
| 135    | MSRPC    | Microsoft Windows RPC     |
| 139    | NetBIOS  | Microsoft Windows NetBIOS |
| 445    | SMB      | Microsoft-DS              |
| 5985   | HTTP     | Microsoft HTTPAPI 2.0     |

### SMB — 445

Se utiliza `smbclient` para listar los recursos compartidos:

```bash
smbclient -L [IP]
```

**Recursos encontrados:**

```text
ADMIN$
C$
IPC$
WorkShares
```

* `WorkShares` permite acceso sin contraseña.
* Nmap indica que **SMB Message Signing** está habilitado pero no es obligatorio.

---

## 3. 💥 Explotación

**Vulnerabilidad:** SMB Share accesible sin autenticación.

Nos conectamos al recurso:

```bash
smbclient //[IP]/WorkShares -N
```

Una vez dentro, navegamos por los directorios:

```bash
cd [directorio]
ls
```

Encontramos la flag en el directorio de `James`:

```bash
get flag.txt
```

**Acceso obtenido:** Acceso al recurso SMB `WorkShares`.

---

## 4. 🐚 Escalada de privilegios

No es necesaria. La flag es accesible directamente mediante el recurso compartido.

---

## 5. 📄 Reporte

### Vulnerabilidades

* **SMB Share sin autenticación** → Permite acceder a archivos compartidos sin credenciales.

### Remediación

* Requerir autenticación para los recursos SMB.
* Aplicar permisos adecuados a los recursos compartidos.

---

## 🧠 Lessons Learned

* **Aprendí:** SMB, recursos compartidos y `smbclient`.
* **Herramientas:** Nmap, smbclient.
* **Comandos nuevos:** `smbclient -L`, `smbclient -N`, `get`.
