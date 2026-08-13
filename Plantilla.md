# 🖥️ [Nombre de la máquina]

**Plataforma:** HTB / THM / ...
**Dificultad:** Easy / Medium / Hard

---

## 1. 🔎 Reconocimiento

```bash
ping -c 1 [IP]
```

**Info:**

* IP:
* SO:
* Hostname:

---

## 2. 🔍 Enumeración

### Puertos

```bash
nmap -p- --open -sS -Pn [IP]
nmap -sCV -p[PUERTOS] [IP]
```

| Puerto | Servicio | Versión |
| ------ | -------- | ------- |
| 22     | SSH      |         |
| 80     | HTTP     |         |

### Otros hallazgos

* [Directorios]
* [Usuarios]
* [Archivos]
* [Vulnerabilidades]

---

## 3. 💥 Explotación

**Vulnerabilidad:** [Nombre / CVE]

```bash
[comandos]
```

**Acceso obtenido:** `[usuario]`

---

## 4. 🐚 Escalada de privilegios

### Enumeración

```bash
id
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

**Hallazgo:** [Vector de escalada]

### Escalada

```bash
[comandos]
```

**Privilegios:** `[root/Administrator]`

---

## 5. 📄 Reporte

### Vulnerabilidades

* **[Vulnerabilidad]** → [Impacto]
* **[Vulnerabilidad]** → [Impacto]

### Remediación

* [Cómo solucionarlo]
* [Cómo solucionarlo]

---

## 🧠 Lessons Learned

* **Aprendí:** [Conceptos]
* **Herramientas:** [Nmap, FFUF, etc.]
* **Comandos nuevos:** `[comando]`
* **Tengo que estudiar:** [Concepto]
