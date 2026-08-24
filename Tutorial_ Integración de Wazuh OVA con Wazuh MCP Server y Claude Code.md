# Tutorial: Integración de Wazuh OVA con Wazuh MCP Server y Claude Code

## 1. Objetivo

El objetivo de este laboratorio es integrar una instalación de **Wazuh desplegada mediante OVA** con **Wazuh MCP Server** y **Claude Code**, de forma que un analista pueda consultar la información de Wazuh utilizando lenguaje natural.

Al finalizar será posible realizar preguntas como:

```text
Muéstrame todos los agentes registrados en Wazuh e indica cuáles están activos.
```

```text
Analiza las alertas de las últimas 24 horas y dime cuáles son las más relevantes.
```

```text
Busca eventos Windows Event ID 4769 y analiza si existe comportamiento compatible con Kerberoasting.
```

La arquitectura utilizada es:

```text
                 USUARIO
                    │
                    │ Lenguaje natural
                    ▼
              Claude Code
                en Kali
                    │
                    │ MCP / HTTP
                    ▼
          Wazuh MCP Server
          127.0.0.1:3000
              /          \
             /            \
      HTTPS 55000       HTTPS 9200
          │                 │
          ▼                 ▼
     Wazuh Manager     Wazuh Indexer
             Wazuh OVA
          172.16.42.200
                │
                │ TCP 1514
                ▼
          Windows 11 Agent
```

Wazuh MCP Server utiliza la API del Manager para consultar información administrativa de Wazuh y el Indexer para consultas de alertas, agregaciones y vulnerabilidades. El proyecto actualmente documenta Docker 20.10+ con Compose v2 y Wazuh 4.8.0–4.14.7 como prerrequisitos.

---

# 2. Componentes necesarios

## 2.1. Infraestructura

Para reproducir este laboratorio se necesitan como mínimo:

| Componente | Función |
|---|---|
| Wazuh OVA | SIEM central |
| Kali Linux | Servidor MCP + Claude Code |
| Windows 11 | Endpoint monitoreado |
| VMware/VirtualBox | Plataforma de virtualización |
| Internet en Kali | Instalación de Docker, MCP y Claude |

En este ejemplo:

```text
Wazuh OVA:   172.16.42.200
Kali Linux:  misma red de laboratorio
Windows 11:  misma red de laboratorio
```

> **Importante:** sustituir `172.16.42.200` por la dirección IP real del Wazuh utilizado en cada laboratorio.

---

# 3. Puertos utilizados

| Puerto | Componente | Uso |
|---:|---|---|
| 443/TCP | Wazuh Dashboard | Interfaz web |
| 55000/TCP | Wazuh Manager API | Consultas administrativas |
| 9200/TCP | Wazuh Indexer | Alertas, búsquedas, vulnerabilidades |
| 1514/TCP | Wazuh Manager | Comunicación de agentes |
| 1515/TCP | Wazuh Manager | Enrollment de agentes |
| 3000/TCP | Wazuh MCP Server | Servicio MCP |

El MCP quedará publicado únicamente en:

```text
127.0.0.1:3000
```

por lo que solo Claude Code ejecutándose en el mismo Kali podrá acceder directamente.

---

# 4. Preparación de la red

Las máquinas virtuales deben encontrarse en una red que permita comunicación entre ellas.

Por ejemplo:

```text
Wazuh     172.16.42.200/24
Kali      172.16.42.X/24
Windows   172.16.42.Y/24
```

En Wazuh verificar:

```bash
ip a
```

En Kali:

```bash
ip a
```

Luego probar conectividad:

```bash
ping -c 4 172.16.42.200
```

También:

```bash
ip route get 172.16.42.200
```

Si aparece:

```text
Network is unreachable
```

el problema es de red y no de Wazuh.

Revisar que las máquinas estén conectadas al mismo:

```text
NAT
Host-only
Bridged
Custom VMnet
```

según el diseño del laboratorio.

---

# 5. Verificación inicial de Wazuh OVA

Desde Kali comprobar:

```bash
nc -zv 172.16.42.200 443
```

```bash
nc -zv 172.16.42.200 55000
```

```bash
nc -zv 172.16.42.200 9200
```

Los puertos 443 y 55000 deberían estar accesibles.

En muchas instalaciones OVA, el Indexer inicialmente escucha únicamente en `127.0.0.1:9200`, por lo que el tercer comando podría fallar.

Wazuh documenta `127.0.0.1` como valor predeterminado de `network.host` para el Indexer.

---

# 6. Verificación del Dashboard

Desde Kali:

```bash
curl -vk https://172.16.42.200/
```

Una respuesta como:

```text
HTTP/1.1 302 Found
location: /app/login
```

significa que el Dashboard está funcionando.

Acceder mediante:

```text
https://172.16.42.200
```

Las instalaciones Wazuh utilizan certificados propios/autofirmados de manera predeterminada, por lo que el navegador puede mostrar una advertencia.

## Problema frecuente: proxy de Firefox

Si Firefox muestra:

```text
The proxy server is refusing connections
```

revisar:

```text
Firefox
→ Settings
→ Network Settings
→ No proxy
```

También revisar FoxyProxy, Burp Suite o ZAP.

Si se desea conservar el proxy, agregar:

```text
172.16.42.200
```

a las exclusiones.

---

# 7. Verificación de la API de Wazuh

La API del Manager utiliza normalmente:

```text
TCP 55000
HTTPS
```

Desde Kali:

```bash
curl -k https://172.16.42.200:55000
```

Si aparece:

```json
{
  "title": "Unauthorized",
  "detail": "No authorization token provided"
}
```

la conectividad es correcta.

En instalaciones realizadas mediante **OVA**, Wazuh crea inicialmente:

```text
Usuario API: wazuh
Password:    wazuh
```

salvo que posteriormente se cambien estas credenciales.

Probar autenticación:

```bash
curl -k -u wazuh:wazuh \
-X POST \
"https://172.16.42.200:55000/security/user/authenticate?raw=true"
```

La respuesta debe ser un JWT similar a:

```text
eyJhbGciOiJFUzUxMiIsInR5cCI6...
```

Wazuh documenta este procedimiento para obtener el token de la API.

---

# 8. Habilitar acceso remoto al Wazuh Indexer

Wazuh MCP Server necesita acceso al Indexer para realizar búsquedas de alertas, agregaciones y consultas de vulnerabilidades.

Primero ingresar a la consola de la OVA.

Comprobar:

```bash
sudo ss -lntp | grep -E ':9200|:55000|:443'
```

Puede aparecer:

```text
0.0.0.0:55000
0.0.0.0:443
127.0.0.1:9200
```

Esto explica por qué Kali puede acceder a 55000 pero no a 9200.

## 8.1. Respaldar configuración

```bash
sudo cp /etc/wazuh-indexer/opensearch.yml \
/etc/wazuh-indexer/opensearch.yml.bak
```

## 8.2. Editar configuración

```bash
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Buscar:

```yaml
network.host: "127.0.0.1"
```

Para un **laboratorio aislado**, cambiar a:

```yaml
network.host: "0.0.0.0"
```

Otra alternativa más restrictiva es utilizar la IP específica:

```yaml
network.host: "172.16.42.200"
```

`network.host` determina la interfaz donde el Indexer acepta conexiones. Wazuh advierte que en despliegues formales este valor debe planificarse de acuerdo con la dirección del nodo y los certificados utilizados.

## 8.3. Reiniciar Indexer

```bash
sudo systemctl restart wazuh-indexer
```

Verificar:

```bash
sudo systemctl status wazuh-indexer --no-pager
```

Debe aparecer:

```text
Active: active (running)
```

Comprobar escucha:

```bash
sudo ss -lntp | grep 9200
```

Esperado:

```text
0.0.0.0:9200
```

---

# 9. Verificar Indexer desde Kali

Desde Kali:

```bash
curl -k https://172.16.42.200:9200/
```

Un mensaje `Unauthorized` confirma que existe conectividad.

La OVA utiliza inicialmente:

```text
Usuario: admin
Password: admin
```

salvo que las credenciales hayan sido cambiadas.

Probar:

```bash
curl -k -u admin:admin \
https://172.16.42.200:9200/
```

Debe devolver información similar a:

```json
{
  "name": "node-1",
  "cluster_name": "wazuh-cluster",
  "version": {
    ...
  }
}
```

La autenticación básica del Indexer está habilitada de forma predeterminada.

---

# 10. Consideración de seguridad

La apertura:

```yaml
network.host: "0.0.0.0"
```

se utilizó exclusivamente para facilitar este laboratorio.

En producción se recomienda:

- limitar el acceso mediante firewall;
- utilizar una interfaz/IP específica;
- utilizar túneles o redes de administración;
- validar correctamente certificados;
- no exponer TCP/9200 a redes no confiables.

---

# 11. Preparación de Kali Linux

Actualizar paquetes:

```bash
sudo apt update
```

Instalar dependencias:

```bash
sudo apt install -y \
git \
curl \
jq \
openssl \
python3 \
docker.io \
docker-compose
```

Activar Docker:

```bash
sudo systemctl enable --now docker
```

Verificar:

```bash
docker --version
```

```bash
docker compose version
```

## Lección aprendida

En Kali, durante este laboratorio:

```bash
sudo apt install docker-compose-plugin
```

devolvió:

```text
Package docker-compose-plugin is not available
```

La solución fue:

```bash
sudo apt install -y docker-compose
```

y posteriormente:

```bash
docker compose version
```

funcionó correctamente.

---

# 12. Descargar Wazuh MCP Server

En Kali:

```bash
cd ~
```

Clonar:

```bash
git clone https://github.com/gensecaihq/Wazuh-MCP-Server.git
```

Ingresar:

```bash
cd ~/Wazuh-MCP-Server
```

El proyecto oficialmente recomienda:

```bash
git clone https://github.com/gensecaihq/Wazuh-MCP-Server.git
cd Wazuh-MCP-Server
cp .env.example .env
```



Crear configuración:

```bash
cp .env.example .env
```

---

# 13. Generar credenciales internas del MCP

El `compose.yml` actual ejecuta el contenedor con:

```text
ENVIRONMENT=production
```

y `AUTH_MODE=bearer` de manera predeterminada. En producción `AUTH_SECRET_KEY` es obligatorio. Además, Compose fuerza internamente `MCP_HOST=0.0.0.0`, pero publica el servicio en `MCP_BIND`, cuyo valor predeterminado es `127.0.0.1`.

Generar `AUTH_SECRET_KEY`:

```bash
openssl rand -hex 32
```

Guardar el resultado.

Generar `MCP_API_KEY`:

```bash
python3 -c "import secrets; print('wazuh_' + secrets.token_urlsafe(32))"
```

Guardar también este resultado.

Ejemplo:

```text
AUTH_SECRET_KEY=79a0...
MCP_API_KEY=wazuh_FJ2k....
```

No utilizar estos ejemplos literalmente.

---

# 14. Configurar `.env`

Editar:

```bash
nano ~/Wazuh-MCP-Server/.env
```

Configurar:

```ini
# =========================================
# WAZUH MANAGER
# =========================================

WAZUH_HOST=https://172.16.42.200
WAZUH_PORT=55000

WAZUH_USER=wazuh
WAZUH_PASS=wazuh

WAZUH_VERIFY_SSL=false
WAZUH_ALLOW_SELF_SIGNED=true


# =========================================
# WAZUH INDEXER
# =========================================

WAZUH_INDEXER_HOST=172.16.42.200
WAZUH_INDEXER_PORT=9200

WAZUH_INDEXER_USER=admin
WAZUH_INDEXER_PASS=admin

WAZUH_INDEXER_SSL=true
WAZUH_INDEXER_VERIFY_SSL=false


# =========================================
# MCP SERVER
# =========================================

MCP_BIND=127.0.0.1
MCP_PORT=3000


# =========================================
# AUTENTICACIÓN
# =========================================

AUTH_MODE=bearer

AUTH_SECRET_KEY=PEGAR_AQUI_LA_PRIMERA_CLAVE

MCP_API_KEY=PEGAR_AQUI_LA_SEGUNDA_CLAVE

MCP_API_KEY_SCOPES=wazuh:read


# =========================================
# LOGGING
# =========================================

LOG_LEVEL=INFO
```

Wazuh MCP Server define `WAZUH_HOST`, `WAZUH_USER` y `WAZUH_PASS` como variables requeridas. Para alertas y vulnerabilidades utiliza las variables `WAZUH_INDEXER_*`.

---

# 15. ¿Por qué `wazuh:read`?

Se utiliza:

```ini
MCP_API_KEY_SCOPES=wazuh:read
```

para permitir a Claude consultar:

- agentes;
- alertas;
- vulnerabilidades;
- compliance;
- información del sistema;
- threat hunting;
- análisis.

El MCP actual separa herramientas `wazuh:read` y `wazuh:write`; el permiso de escritura habilita funciones de Active Response y otras acciones que modifican el entorno.

Para una primera PoC no utilizar:

```text
wazuh:write
```

---

# 16. Verificar que no queden placeholders

Este fue uno de los problemas más importantes encontrados durante el laboratorio.

Ejecutar:

```bash
grep -E \
'^(WAZUH_HOST|WAZUH_PORT|WAZUH_USER|WAZUH_INDEXER_HOST|WAZUH_INDEXER_PORT|WAZUH_INDEXER_USER)=' \
.env
```

Debe mostrar:

```text
WAZUH_HOST=https://172.16.42.200
WAZUH_PORT=55000
WAZUH_USER=wazuh
WAZUH_INDEXER_HOST=172.16.42.200
WAZUH_INDEXER_PORT=9200
WAZUH_INDEXER_USER=admin
```

Buscar también:

```bash
grep -R "your-wazuh-server.com" -n . --exclude-dir=.git
```

Puede aparecer en:

```text
.env.example
```

pero **no debe aparecer dentro de `.env`**.

---

# 17. Validar Docker Compose

Desde:

```bash
cd ~/Wazuh-MCP-Server
```

ejecutar:

```bash
docker compose config
```

Si no existen errores, continuar.

---

# 18. Levantar Wazuh MCP Server

Ejecutar:

```bash
docker compose up -d
```

La documentación oficial utiliza `docker compose up -d` como mecanismo estándar de despliegue.

Comprobar:

```bash
docker compose ps
```

Debe aparecer aproximadamente:

```text
NAME                 STATUS
wazuh-main-server    Up (healthy)
```

También:

```text
127.0.0.1:3000->3000/tcp
```

El `compose.yml` actual utiliza el nombre:

```text
wazuh-main-server
```

y publica el puerto por defecto en loopback.

---

# 19. Warning BUILD_DATE

Puede aparecer:

```text
WARN The "BUILD_DATE" variable is not set.
Defaulting to a blank string.
```

Para este laboratorio ese warning no impide el funcionamiento del contenedor.

Si:

```bash
docker compose ps
```

muestra:

```text
Up (...) (healthy)
```

el servidor está funcionando.

---

# 20. Comprobar salud del MCP

Ejecutar:

```bash
curl -s http://127.0.0.1:3000/ready | jq .
```

El endpoint `/ready` valida la aplicación y sus conexiones con Wazuh/Indexer.

También:

```bash
curl -s http://127.0.0.1:3000/health | jq .
```

Si `jq` no está instalado:

```bash
sudo apt install -y jq
```

---

# 21. Verificar configuración realmente cargada por Docker

Este paso es muy importante.

Ejecutar:

```bash
docker exec wazuh-main-server sh -c \
'echo "Manager: $WAZUH_HOST:$WAZUH_PORT"; \
 echo "Indexer: $WAZUH_INDEXER_HOST:$WAZUH_INDEXER_PORT"'
```

Debe aparecer:

```text
Manager: https://172.16.42.200:55000
Indexer: 172.16.42.200:9200
```

Si aparece:

```text
your-wazuh-server.com
```

el contenedor sigue utilizando la configuración de ejemplo.

---

# 22. Si se modifica `.env`

Una lección importante:

**editar `.env` no garantiza que un contenedor ya creado adopte inmediatamente los nuevos valores.**

Después de modificar `.env`, ejecutar:

```bash
docker compose down
```

y:

```bash
docker compose up -d --force-recreate
```

Luego:

```bash
docker compose ps
```

y nuevamente:

```bash
docker exec wazuh-main-server sh -c \
'echo "Manager: $WAZUH_HOST:$WAZUH_PORT"; \
 echo "Indexer: $WAZUH_INDEXER_HOST:$WAZUH_INDEXER_PORT"'
```

---

# 23. Ver logs del MCP

Para troubleshooting:

```bash
docker compose logs --tail=100
```

En tiempo real:

```bash
docker compose logs -f --timestamps wazuh-main-server
```

La guía operativa oficial documenta estos comandos.

---

# 24. Instalar Claude Code en Kali

Ejecutar:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Comprobar:

```bash
claude --version
```

---

# 25. Instalación de Claude con salida visible

Si se desea observar la descarga:

```bash
curl -L --progress-bar \
https://claude.ai/install.sh \
-o /tmp/claude-install.sh
```

Luego ejecutar con trazado:

```bash
bash -x /tmp/claude-install.sh
```

Esto permite ver cada comando que ejecuta el instalador.

Si la descarga interna parece detenida, en otra terminal puede observarse el archivo:

```bash
watch -n 1 'ls -lh /root/.claude/downloads/'
```

---

# 26. Iniciar Claude Code

Ejecutar:

```bash
claude
```

Completar el proceso de autenticación/login de Claude.

Luego salir temporalmente para configurar MCP.

---

# 27. Obtener un token del Wazuh MCP Server

Wazuh MCP utiliza `AUTH_MODE=bearer`.

El flujo es:

```text
MCP_API_KEY
     │
     ▼
POST /auth/token
     │
     ▼
JWT
     │
     ▼
Claude Code
```

El proyecto documenta el intercambio de API Key por JWT mediante `/auth/token`.

Ingresar:

```bash
cd ~/Wazuh-MCP-Server
```

Cargar la API Key:

```bash
export MCP_API_KEY=$(
  grep '^MCP_API_KEY=' .env |
  cut -d= -f2-
)
```

Generar JWT:

```bash
export WAZUH_MCP_TOKEN=$(
  curl -s \
    -X POST \
    http://127.0.0.1:3000/auth/token \
    -H "Content-Type: application/json" \
    -d "{\"api_key\":\"$MCP_API_KEY\"}" |
  jq -r '.access_token'
)
```

Comprobar que existe:

```bash
echo ${#WAZUH_MCP_TOKEN}
```

Debe devolver un número mayor que cero.

No mostrar ni compartir el token.

---

# 28. Conectar Claude Code con Wazuh MCP

Ejecutar:

```bash
claude mcp add \
  --transport http \
  wazuh \
  http://127.0.0.1:3000/mcp \
  --header "Authorization: Bearer $WAZUH_MCP_TOKEN"
```

La sintaxis actual de Claude Code para servidores HTTP es:

```text
claude mcp add --transport http <nombre> <url>
```

y permite agregar el Bearer token mediante `--header`.

---

# 29. Error de sintaxis encontrado

Esta forma:

```bash
claude mcp add \
  --transport http \
  --header "Authorization: Bearer ..." \
  wazuh \
  http://127.0.0.1:3000/mcp
```

puede producir:

```text
error: missing required argument 'name'
```

La forma utilizada correctamente fue:

```bash
claude mcp add \
  --transport http \
  wazuh \
  http://127.0.0.1:3000/mcp \
  --header "Authorization: Bearer $WAZUH_MCP_TOKEN"
```

---

# 30. Verificar integración desde terminal

Ejecutar:

```bash
claude mcp list
```

Debe mostrar algo equivalente a:

```text
wazuh
http://127.0.0.1:3000/mcp
Connected
```

También:

```bash
claude mcp get wazuh
```

Claude Code documenta `mcp list`, `mcp get` y `/mcp` como mecanismos de administración y diagnóstico.

---

# 31. Scope de configuración de Claude

Cuando se ejecuta:

```bash
claude mcp add ...
```

sin especificar `--scope`, Claude utiliza por defecto configuración **local**, asociada al proyecto/directorio.

En nuestro caso quedó asociado aproximadamente a:

```text
/root/Wazuh-MCP-Server
```

Por esta razón es recomendable ingresar a:

```bash
cd ~/Wazuh-MCP-Server
```

antes de ejecutar:

```bash
claude
```

Claude Code documenta los scopes local, project y user para MCP.

Si se desea utilizar el MCP desde cualquier directorio puede evaluarse posteriormente:

```text
--scope user
```

---

# 32. Verificar MCP desde Claude

Ejecutar:

```bash
cd ~/Wazuh-MCP-Server
claude
```

Dentro de Claude:

```text
/mcp
```

Deben aparecer herramientas similares a:

```text
/wazuh:threat_hunt
/wazuh:compliance_audit
/wazuh:iso27001_assessment
/wazuh:security_investigation
/wazuh:vulnerability_assessment
```

Esto confirma que Claude reconoce las herramientas expuestas por Wazuh MCP Server.

---

# 33. Primera consulta

Salir del menú `/mcp` mediante:

```text
Esc
```

y escribir:

```text
Consulta Wazuh mediante las herramientas MCP.
Muéstrame todos los agentes registrados, incluyendo ID,
nombre, IP, sistema operativo y estado de conexión.
```

La respuesta debe provenir del entorno Wazuh real.

---

# 34. Consultas recomendadas

## Estado de agentes

```text
Muéstrame todos los agentes registrados en Wazuh
e indica cuáles están activos y cuáles desconectados.
```

## Alertas recientes

```text
Muéstrame las 10 alertas más recientes de Wazuh.
Para cada una indica agente, severidad, regla y descripción.
```

## Análisis de endpoint

```text
Analiza las alertas generadas por el agente Windows 11
durante las últimas 24 horas y determina cuáles son las
más relevantes desde el punto de vista de seguridad.
```

## Threat hunting

```text
Realiza una búsqueda de comportamiento sospechoso
en los agentes Wazuh durante las últimas 24 horas.
```

## Kerberoasting

```text
Busca eventos Windows Event ID 4769 en Wazuh
durante las últimas 24 horas y analiza si existen
solicitudes de tickets Kerberos que puedan ser
compatibles con Kerberoasting.
```

---

# 35. Windows 11 como agente Wazuh

El agente Windows debe reportar al Wazuh Manager.

La configuración se encuentra normalmente en:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Debe contener:

```xml
<client>
  <server>
    <address>172.16.42.200</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Wazuh documenta este archivo y la configuración `<client><server><address>` para el Manager.

Comprobar desde Windows:

```powershell
Test-NetConnection 172.16.42.200 -Port 1514
```

---

# 36. Si cambia la IP del servidor Wazuh

Modificar:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

y cambiar:

```xml
<address>IP_ANTERIOR</address>
```

por:

```xml
<address>NUEVA_IP</address>
```

Luego identificar el servicio:

```powershell
Get-Service *wazuh*
```

y reiniciar utilizando el nombre encontrado.

Las versiones/documentación de Wazuh muestran comandos tanto con `wazuh` como con `WazuhSvc/wazuhsvc`, por lo que descubrir primero el nombre del servicio evita problemas entre versiones.

---

# 37. Troubleshooting

## 37.1. `docker: unknown command: docker compose`

Problema:

```text
docker: unknown command: docker compose
```

Solución utilizada en Kali:

```bash
sudo apt update
sudo apt install -y docker-compose
```

Verificar:

```bash
docker compose version
```

---

## 37.2. `docker-compose-plugin is not available`

Si aparece:

```text
Package docker-compose-plugin is not available
```

en Kali instalar:

```bash
sudo apt install -y docker-compose
```

---

## 37.3. `jq: command not found`

Instalar:

```bash
sudo apt install -y jq
```

---

## 37.4. Indexer 9200 no responde

Desde Kali:

```text
Failed to connect to 172.16.42.200 port 9200
```

Comprobar en Wazuh:

```bash
sudo ss -lntp | grep 9200
```

Si aparece:

```text
127.0.0.1:9200
```

revisar:

```text
/etc/wazuh-indexer/opensearch.yml
```

y `network.host`.

---

## 37.5. `Unauthorized`

Si:

```bash
curl -k https://172.16.42.200:55000
```

devuelve `Unauthorized`, esto no necesariamente es un error.

Significa:

```text
Conectividad OK
Autenticación pendiente
```

---

## 37.6. Dashboard funciona en curl pero no en navegador

Si:

```bash
curl -vk https://172.16.42.200/
```

devuelve:

```text
302 Found
location: /app/login
```

pero Firefox no conecta, revisar:

- Proxy manual.
- FoxyProxy.
- Burp.
- ZAP.
- Excepciones de proxy.
- Certificado autofirmado.

---

## 37.7. `Network is unreachable`

No es problema de Wazuh.

Comprobar:

```bash
ip -br a
```

```bash
ip route
```

```bash
ip route get 172.16.42.200
```

Revisar adaptadores de VMware/VirtualBox.

---

## 37.8. MCP muestra `healthy` pero Claude no obtiene agentes

Comprobar configuración real:

```bash
docker exec wazuh-main-server sh -c \
'echo "$WAZUH_HOST"; \
 echo "$WAZUH_INDEXER_HOST"'
```

---

## 37.9. Claude informa `your-wazuh-server.com`

Este fue uno de los errores encontrados.

Ejemplo:

```text
el servidor Wazuh MCP apunta a
your-wazuh-server.com:55000
```

Significa que el contenedor utiliza los valores de `.env.example`.

Corregir `.env`:

```ini
WAZUH_HOST=https://172.16.42.200
```

Luego:

```bash
docker compose down
```

```bash
docker compose up -d --force-recreate
```

Comprobar:

```bash
docker exec wazuh-main-server sh -c \
'echo "$WAZUH_HOST"'
```

Debe mostrar:

```text
https://172.16.42.200
```

---

## 37.10. Claude muestra herramientas MCP pero no datos reales

Que aparezca:

```text
/wazuh:threat_hunt
/wazuh:security_investigation
```

solo confirma:

```text
Claude → MCP
```

No confirma necesariamente:

```text
MCP → Wazuh
```

Por eso se debe comprobar:

```bash
curl -s http://127.0.0.1:3000/ready | jq .
```

y:

```bash
docker exec wazuh-main-server sh -c \
'echo "$WAZUH_HOST"; echo "$WAZUH_INDEXER_HOST"'
```

---

# 38. Token MCP expira

La configuración actual utiliza por defecto una vida de token de aproximadamente **24 horas**.

Si posteriormente Claude deja de autenticarse, generar nuevamente:

```bash
export MCP_API_KEY=$(
  grep '^MCP_API_KEY=' ~/Wazuh-MCP-Server/.env |
  cut -d= -f2-
)
```

```bash
export WAZUH_MCP_TOKEN=$(
  curl -s \
    -X POST \
    http://127.0.0.1:3000/auth/token \
    -H "Content-Type: application/json" \
    -d "{\"api_key\":\"$MCP_API_KEY\"}" |
  jq -r '.access_token'
)
```

Luego actualizar la configuración MCP en Claude.

---

# 39. Session limit de Claude

Si Claude muestra:

```text
You've hit your session limit
```

no es un error de:

```text
Wazuh
MCP
Docker
red
```

Es únicamente un límite temporal de la cuenta/sesión de Claude.

La configuración MCP permanece guardada.

---

# 40. Checklist final

Antes de ejecutar consultas comprobar:

- [ ] Wazuh OVA encendida.
- [ ] Kali puede hacer ping a Wazuh.
- [ ] Windows 11 reporta a Wazuh.
- [ ] TCP/55000 accesible desde Kali.
- [ ] TCP/9200 accesible desde Kali.
- [ ] API Manager autentica correctamente.
- [ ] Indexer autentica correctamente.
- [ ] Docker está activo.
- [ ] `docker compose version` funciona.
- [ ] `.env` contiene la IP real.
- [ ] No existe `your-wazuh-server.com` en `.env`.
- [ ] `docker compose ps` muestra `healthy`.
- [ ] `/ready` funciona.
- [ ] El contenedor tiene cargada la IP real.
- [ ] Claude Code está instalado.
- [ ] Existe `WAZUH_MCP_TOKEN`.
- [ ] `claude mcp list` muestra Wazuh conectado.
- [ ] `/mcp` muestra las herramientas Wazuh.
- [ ] Una consulta devuelve agentes reales.

---

# 41. Flujo de validación rápido

Para futuras instalaciones se recomienda validar por capas.

## Capa 1 — Red

```bash
ping -c 4 172.16.42.200
```

## Capa 2 — Manager

```bash
curl -k -u wazuh:wazuh \
-X POST \
"https://172.16.42.200:55000/security/user/authenticate?raw=true"
```

## Capa 3 — Indexer

```bash
curl -k -u admin:admin \
https://172.16.42.200:9200/
```

## Capa 4 — MCP

```bash
docker compose ps
```

```bash
curl -s http://127.0.0.1:3000/ready | jq .
```

## Capa 5 — Configuración efectiva

```bash
docker exec wazuh-main-server sh -c \
'echo "$WAZUH_HOST"; \
 echo "$WAZUH_INDEXER_HOST"'
```

## Capa 6 — Claude MCP

```bash
claude mcp list
```

## Capa 7 — Consulta real

```text
Consulta Wazuh mediante MCP y muéstrame todos
los agentes activos.
```

Este enfoque permite localizar rápidamente dónde existe un problema.

---

# 42. Recomendaciones para producción

Este laboratorio prioriza facilidad de implementación.

Para un entorno empresarial se recomienda modificar varios aspectos.

## No utilizar credenciales predeterminadas

Cambiar:

```text
wazuh:wazuh
admin:admin
```

por credenciales robustas.

## Crear cuenta exclusiva para MCP

Utilizar una cuenta de servicio específica y aplicar principio de mínimo privilegio.

## No exponer 9200 indiscriminadamente

Evitar:

```text
0.0.0.0:9200
```

en redes productivas.

Restringir el acceso al host MCP mediante firewall.

## Mantener MCP en loopback

Mantener:

```ini
MCP_BIND=127.0.0.1
```

cuando Claude Code y MCP ejecutan en el mismo servidor.

## Validar certificados

En producción evitar:

```ini
WAZUH_VERIFY_SSL=false
WAZUH_INDEXER_VERIFY_SSL=false
```

e instalar certificados confiables.

## Mantener solo lectura inicialmente

Utilizar:

```ini
MCP_API_KEY_SCOPES=wazuh:read
```

y habilitar:

```text
wazuh:write
```

únicamente después de una evaluación formal de riesgos.

## Considerar confidencialidad de datos

Las respuestas obtenidas de Wazuh pueden contener:

- direcciones IP;
- nombres de equipos;
- usuarios;
- vulnerabilidades;
- eventos de autenticación;
- información de incidentes.

Antes de utilizar modelos de IA externos con información corporativa se debe verificar la política de tratamiento de información aplicable.

---

# 43. Arquitectura recomendada para una futura implementación empresarial

```text
                   Claude / LLM
                        │
                        │ HTTPS
                        ▼
                Reverse Proxy
                 TLS + Firewall
                        │
                        ▼
              Wazuh MCP Server
             VM independiente
                 Solo lectura
                  /        \
                 /          \
            55000            9200
              │                │
              ▼                ▼
        Wazuh Manager     Wazuh Indexer
                \           /
                 \         /
                    Wazuh
                     │
           ┌─────────┼─────────┐
           ▼         ▼         ▼
        Windows    Linux     Servers
```

Para laboratorio, sin embargo, la arquitectura implementada:

```text
Claude Code + MCP
      Kali
       │
       ▼
   Wazuh OVA
       │
       ▼
   Windows 11
```

es suficiente para demostrar integración y consultas SIEM mediante lenguaje natural.

---

# 44. Resumen

La integración se compone de cuatro capas:

```text
1. Endpoint
   Windows 11 → Wazuh

2. SIEM
   Wazuh Manager + Indexer

3. Integración IA
   Wazuh MCP Server

4. Interfaz conversacional
   Claude Code
```

El flujo final es:

```text
Pregunta en lenguaje natural
            │
            ▼
       Claude Code
            │
            ▼
      Protocolo MCP
            │
            ▼
    Wazuh MCP Server
            │
      ┌─────┴─────┐
      ▼           ▼
 Manager API    Indexer
      │           │
      └─────┬─────┘
            ▼
       Datos Wazuh
            │
            ▼
       Claude analiza
            │
            ▼
     Respuesta al analista
```

Con esta configuración es posible convertir Wazuh en una plataforma SIEM consultable mediante lenguaje natural y utilizar Claude como interfaz para análisis, threat hunting, investigación, vulnerabilidades y revisión de eventos de seguridad.