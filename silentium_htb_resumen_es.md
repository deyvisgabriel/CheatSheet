# CTF Writeup Silentium – HTB

> **Fuente original:** artículo de Lewis Sawe en DEV Community: “CTF Writeup Silentium - HTB”.  
> **Tipo de documento:** versión en español adaptada y resumida en formato Markdown.  
> **Nota:** este material no es una traducción literal completa del artículo original. Resume y reorganiza la cadena de ataque con fines educativos y de laboratorio autorizado.

---

## 1. Descripción general

La máquina **Silentium** de Hack The Box presenta una cadena de explotación que inicia con una instancia vulnerable de **Flowise AI** y termina con una escalada de privilegios aprovechando una vulnerabilidad reciente en **Gogs**.

La ruta general es:

1. Enumeración de servicios.
2. Descubrimiento de virtual hosts.
3. Acceso inicial mediante vulnerabilidad de restablecimiento de contraseña en Flowise AI.
4. Ejecución remota de código en Flowise mediante configuración insegura de MCP.
5. Escape o pivote desde contenedor Docker hacia el host.
6. Escalada de privilegios mediante escritura arbitraria de archivos en Gogs.
7. Obtención de privilegios de `root`.

---

## 2. Enumeración inicial

La enumeración del objetivo permite identificar un servidor web **Nginx** y el servicio **SSH** abierto.

Comando típico:

```bash
nmap -sV -sC <IP_OBJETIVO>
```

El servidor web redirige hacia el dominio:

```text
silentium.htb
```

Por ello, se agrega el dominio al archivo `/etc/hosts`:

```bash
echo "<IP_OBJETIVO> silentium.htb" | sudo tee -a /etc/hosts
```

---

## 3. Fuzzing de virtual hosts

Al tratarse de una aplicación web, se realiza fuzzing de subdominios o virtual hosts. El objetivo es descubrir entornos adicionales expuestos en el mismo servidor.

Ejemplo con `gobuster`:

```bash
gobuster vhost -u http://silentium.htb \
  -w /usr/share/wordlists/dirb/common.txt \
  --append-domain
```

El resultado relevante es el subdominio:

```text
staging.silentium.htb
```

Luego se agrega también al archivo `/etc/hosts`:

```bash
echo "<IP_OBJETIVO> staging.silentium.htb" | sudo tee -a /etc/hosts
```

Al navegar al subdominio se identifica una instancia de **Flowise AI**, versión **3.0.5**.

---

## 4. Acceso inicial: vulnerabilidad de restablecimiento de contraseña en Flowise AI

La instancia de Flowise AI presenta una falla lógica en la API de recuperación de contraseña. Según el writeup original, la vulnerabilidad permite obtener un token temporal de restablecimiento directamente desde la respuesta de la API.

El punto vulnerable corresponde al flujo de “forgot password”. Al enviar una solicitud con un usuario válido, la respuesta incluye información sensible como el token temporal.

Ejemplo conceptual:

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"<USUARIO_VALIDO>"}}'
```

La respuesta puede incluir un campo similar a:

```text
tempToken
```

Este token tiene una vida útil corta, por lo que debe utilizarse rápidamente.

---

## 5. Restablecimiento exitoso de contraseña

El writeup indica que, al intentar restablecer la contraseña, inicialmente se obtenía un error `500 Internal Server Error`. El problema se debía principalmente a dos aspectos:

1. El token expiraba rápidamente.
2. La estructura esperada del JSON no era anidada, sino plana.

La estructura corregida del payload es similar a:

```json
{
  "tempToken": "<TOKEN_RECIENTE>",
  "password": "<NUEVA_PASSWORD>",
  "confirmPassword": "<NUEVA_PASSWORD>"
}
```

Solicitud conceptual:

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d @reset.json
```

Después de este paso, se logra iniciar sesión en el panel de Flowise con las nuevas credenciales.

---

## 6. Ejecución remota de código en Flowise mediante MCP

Una vez autenticado en Flowise, se analiza la funcionalidad relacionada con **Model Context Protocol (MCP)**.

El writeup describe que el endpoint:

```text
/api/v1/node-load-method/customMCP
```

procesa el parámetro `mcpServerConfig` de forma insegura. La aplicación evalúa el contenido dentro del entorno Node.js sin un aislamiento adecuado, lo que permite inyectar código JavaScript.

La idea del ataque es:

1. Obtener un token de sesión o una API key desde el panel.
2. Construir un payload dirigido al método `customMCP`.
3. Inyectar una expresión JavaScript que ejecute comandos en el servidor.
4. Enviar el payload autenticado contra el endpoint vulnerable.

Ejemplo conceptual del envío:

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d @payload.json
```

El resultado es una shell dentro del contenedor Docker donde se ejecuta Flowise, con el usuario del proceso Node.js.

---

## 7. Pivote desde el contenedor hacia el host

Una vez dentro del contenedor, se realiza enumeración local. Un punto importante es revisar variables de entorno, ya que pueden contener credenciales, secretos o datos de conexión.

Comando útil:

```bash
env
```

En el caso descrito, se encuentran credenciales asociadas al usuario local `ben`. Como el servicio SSH está expuesto en el host, esas credenciales permiten iniciar sesión fuera del contenedor:

```bash
ssh ben@silentium.htb
```

Con ello se obtiene acceso como usuario del sistema y se puede leer la flag de usuario:

```bash
cat /home/ben/user.txt
```

---

## 8. Enumeración para escalada de privilegios

Como usuario `ben`, se realiza una nueva enumeración del sistema. Se identifica una instancia local de **Gogs** escuchando en puertos internos, como `3000` o `3001`.

También se confirma que el proceso de Gogs se ejecuta con privilegios elevados.

Comandos de referencia:

```bash
ss -tulpn
ps aux | grep gogs
```

El hallazgo clave es que Gogs se ejecuta como `root`, por lo que una vulnerabilidad de escritura arbitraria de archivos podría permitir modificar archivos sensibles del sistema.

---

## 9. Port forwarding hacia Gogs

Como Gogs solo está expuesto localmente, se utiliza reenvío de puertos por SSH para acceder desde la máquina atacante.

Ejemplo:

```bash
ssh -L 8080:127.0.0.1:3000 ben@silentium.htb
```

Luego se puede abrir Gogs desde el navegador local:

```text
http://127.0.0.1:8080
```

---

## 10. Escalada de privilegios mediante Gogs

El writeup utiliza una vulnerabilidad de Gogs asociada a escritura arbitraria de archivos mediante manejo inseguro de enlaces simbólicos a través de la API.

La lógica general es:

1. Crear una cuenta en Gogs.
2. Generar un token de API.
3. Crear un repositorio.
4. Subir un enlace simbólico que apunte a un archivo fuera del repositorio.
5. Usar la API de Gogs para actualizar el contenido del enlace simbólico.
6. Forzar que Gogs escriba el contenido en una ruta sensible del sistema.

El objetivo práctico descrito es escribir una regla en `/etc/sudoers.d/` para permitir que el usuario `ben` ejecute comandos con `sudo` sin contraseña.

Payload conceptual:

```text
ben ALL=(ALL) NOPASSWD: ALL
```

Es importante que el archivo termine con salto de línea, ya que los archivos de configuración de `sudoers` pueden fallar si el formato no es correcto.

---

## 11. Explotación automatizada

El artículo original muestra un script en Python que automatiza el proceso de autenticación, generación de token, creación de repositorio, creación del enlace simbólico y escritura final del payload.

En esta versión resumida no se reproduce el exploit completo. A nivel lógico, el script realiza estas acciones:

1. Inicia sesión en Gogs.
2. Obtiene un token CSRF.
3. Genera un token de API.
4. Crea un repositorio inicializado.
5. Clona el repositorio.
6. Crea un enlace simbólico hacia una ruta sensible.
7. Hace commit y push del enlace.
8. Usa la API para escribir contenido sobre el enlace simbólico.
9. Modifica la configuración de `sudoers` para permitir escalada.

Ejecución conceptual:

```bash
python3 exploit_gogs_lab.py -u http://127.0.0.1:8080
```

---

## 12. Escalada final

Después de ejecutar la explotación de Gogs, se vuelve a la sesión SSH como `ben` y se verifica si el usuario ya puede ejecutar comandos como `root`.

```bash
sudo id
```

Resultado esperado:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Finalmente, se puede leer la flag de root:

```bash
sudo cat /root/root.txt
```

---

## 13. Cadena de ataque resumida

```text
Nmap
  ↓
Descubrimiento de silentium.htb
  ↓
Fuzzing de virtual hosts
  ↓
staging.silentium.htb
  ↓
Flowise AI 3.0.5
  ↓
Password reset token leak
  ↓
Acceso al panel de Flowise
  ↓
RCE mediante customMCP
  ↓
Shell dentro de contenedor Docker
  ↓
Credenciales en variables de entorno
  ↓
SSH como ben en el host
  ↓
Gogs local ejecutándose como root
  ↓
Port forwarding SSH
  ↓
Abuso de symlink + API en Gogs
  ↓
Escritura en /etc/sudoers.d/
  ↓
sudo sin contraseña
  ↓
Root
```

---

## 14. Aprendizajes principales

### Enumeración web

El fuzzing de virtual hosts puede revelar entornos de staging, pruebas o desarrollo que no aparecen directamente en la página principal.

### Seguridad en APIs

Una API nunca debería devolver tokens sensibles en la respuesta de un flujo de recuperación de contraseña. El token debe enviarse por un canal controlado y seguro, normalmente correo electrónico, y nunca exponerse directamente al cliente.

### Validación de estructuras JSON

Los errores al explotar una vulnerabilidad pueden deberse a detalles simples como la estructura esperada del payload, campos anidados o expiración de tokens.

### Riesgo de secretos en contenedores

Las variables de entorno dentro de contenedores pueden contener credenciales reutilizables. Si esas credenciales también sirven para acceder al host, el contenedor se convierte en un punto de pivote crítico.

### Aplicaciones internas con privilegios altos

Una aplicación interna como Gogs no debería ejecutarse como `root` salvo que sea estrictamente necesario. Si una aplicación vulnerable se ejecuta con privilegios elevados, el impacto de cualquier vulnerabilidad aumenta significativamente.

### Control de enlaces simbólicos

Las aplicaciones que gestionan repositorios, archivos o APIs de escritura deben validar adecuadamente los symlinks para evitar escrituras fuera del directorio permitido.

---

## 15. Recomendaciones defensivas

1. No exponer entornos de staging sin autenticación fuerte.
2. Revisar endpoints de recuperación de contraseña.
3. Evitar que tokens temporales aparezcan en respuestas de API.
4. Aplicar expiración corta y uso único de tokens.
5. No evaluar código o configuraciones dinámicas sin sandboxing.
6. Restringir secretos en variables de entorno.
7. Separar credenciales de contenedores y usuarios del host.
8. Ejecutar servicios como Gogs con usuarios sin privilegios.
9. Validar symlinks y rutas antes de permitir escrituras por API.
10. Mantener Flowise, Gogs y dependencias actualizadas.
11. Monitorear tráfico interno y accesos SSH.
12. Revisar permisos en `/etc/sudoers.d/`.

---

## 16. Comandos de referencia rápida

```bash
# Enumeración inicial
nmap -sV -sC <IP_OBJETIVO>

# Agregar dominio principal
echo "<IP_OBJETIVO> silentium.htb" | sudo tee -a /etc/hosts

# Fuzzing de virtual hosts
gobuster vhost -u http://silentium.htb \
  -w /usr/share/wordlists/dirb/common.txt \
  --append-domain

# Agregar subdominio descubierto
echo "<IP_OBJETIVO> staging.silentium.htb" | sudo tee -a /etc/hosts

# Revisar variables de entorno dentro del contenedor
env

# Acceso SSH al host
ssh ben@silentium.htb

# Reenvío de puerto para Gogs
ssh -L 8080:127.0.0.1:3000 ben@silentium.htb

# Verificación de privilegios
sudo id
```

---

## 17. Glosario breve

**Flowise AI:** plataforma visual para crear flujos de trabajo con modelos de lenguaje e integraciones.  
**MCP:** Model Context Protocol, mecanismo para conectar modelos con herramientas externas.  
**RCE:** Remote Code Execution, ejecución remota de código.  
**Docker container:** entorno aislado donde se ejecuta una aplicación.  
**Gogs:** plataforma ligera de alojamiento de repositorios Git.  
**Symlink:** enlace simbólico que apunta a otro archivo o ruta.  
**Arbitrary File Write:** vulnerabilidad que permite escribir archivos en rutas no autorizadas.  
**sudoers:** configuración que define permisos de ejecución con privilegios elevados en Linux.

---

## 18. Nota ética

Este contenido debe utilizarse únicamente en entornos autorizados, como laboratorios de Hack The Box, máquinas propias o ambientes de entrenamiento. No debe aplicarse contra sistemas reales sin autorización expresa.

