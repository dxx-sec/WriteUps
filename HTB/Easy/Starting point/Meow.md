# 🖥️ Meow

**Plataforma:** HTB
**Dificultad:** Easy

---

## 1. 🔎 Reconocimiento

**Info:**

* IP: `10.10.X.X`
* SO: Linux
* Servicio principal: Telnet

---

## 2. 🔍 Enumeración

### Puertos

```bash
nmap -p- --open -sS -Pn -sCV [IP]
```

| Puerto | Servicio | Versión |
| ------ | -------- | ------- |
| 23     | Telnet   |         |

### Otros hallazgos

* Puerto 23/TCP abierto.
* Servicio Telnet disponible.

---

## 3. 💥 Explotación

**Vulnerabilidad:** Credenciales de acceso débiles / ausencia de contraseña.

```bash
telnet [IP] 23
```

Se prueba el usuario:

```text
root
```

El acceso es exitoso sin contraseña.

**Acceso obtenido:** `root`

---

## 4. 🐚 Escalada de privilegios

No es necesaria, ya que se obtiene acceso directamente como `root`.

---

## 5. 📄 Reporte

### Vulnerabilidades

* **Telnet expuesto** → Permite conexiones remotas inseguras.
* **Root sin contraseña** → Permite obtener acceso privilegiado directamente.

### Remediación

* Deshabilitar Telnet y utilizar SSH.
* Establecer contraseñas robustas y evitar accesos directos como `root`.

---

## 🧠 Lessons Learned

* **Aprendí:** VM, terminal, VPN, ping, Nmap, Telnet y root.
* **Herramientas:** Nmap, Telnet.
* **Comandos nuevos:** `nmap`, `telnet`.
