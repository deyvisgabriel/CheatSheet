# Laboratorio guiado: del secreto escrito al secreto administrado con HashiCorp Vault

## 1. Objetivo del laboratorio

El objetivo de este laboratorio es comprender, de forma práctica, qué cambia cuando una organización deja de guardar contraseñas, tokens o credenciales directamente en archivos o código y comienza a administrarlos mediante un **gestor centralizado de secretos**.

Durante el laboratorio utilizaremos **HashiCorp Vault** ejecutándose en **Docker** para demostrar:

- Centralización de secretos.
- Control de acceso.
- Principio de mínimo privilegio.
- Uso de tokens.
- Revocación de acceso.
- Diferencia entre un secreto estático y un secreto administrado.

> **Idea principal:** el objetivo no es solamente ocultar mejor una contraseña, sino controlar **quién puede acceder a ella, cuándo, para qué y durante cuánto tiempo**.

---

## 2. Escenario

Imaginemos que una aplicación necesita conectarse a una base de datos y actualmente tiene esta información escrita directamente en un archivo de configuración:

```text
usuario=demo
password=ClaveTemporal123
```

Este modelo presenta varios riesgos: el archivo puede ser copiado, incluido en backups o repositorios, el secreto puede durar años y es difícil saber quién lo utilizó.

El objetivo será cambiar de:

```text
Aplicación
   |
   └── password=ClaveTemporal123
```

a:

```text
Aplicación
     |
     | solicita el secreto
     v
   Vault
     |
     | verifica identidad y permisos
     v
  Secreto
```

---

## 3. Requisitos previos

Necesitamos:

- Docker instalado.
- Acceso a una terminal.
- Conexión a Internet para descargar la imagen de Vault.

Comprobar Docker:

```bash
docker --version
docker info
```

---

## 4. Descargar la imagen de Vault

```bash
docker pull hashicorp/vault
```

Comprobar:

```bash
docker images
```

---

## 5. Levantar Vault en Docker

Ejecutaremos Vault en **modo desarrollo**.

> **Importante:** el modo `dev` es únicamente para laboratorios y demostraciones. No debe utilizarse en producción.

```bash
docker run --name vault-lab   --cap-add=IPC_LOCK   -e 'VAULT_DEV_ROOT_TOKEN_ID=root'   -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200'   -p 8200:8200   hashicorp/vault
```

Este comando:

- crea un contenedor llamado `vault-lab`;
- inicia Vault;
- define el token root como `root`;
- expone Vault en el puerto `8200`.

Mantén esta terminal abierta.

---

## 6. Abrir una segunda terminal

Verificar que el contenedor esté funcionando:

```bash
docker ps
```

---

## 7. Entrar al contenedor

Para no instalar el cliente Vault localmente, ejecutaremos los comandos dentro del contenedor:

```bash
docker exec -it vault-lab sh
```

Comprobar la versión:

```bash
vault version
```

---

## 8. Configurar variables de entorno

Dentro del contenedor:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'
```

Verificar:

```bash
vault status
```

Deberíamos observar:

```text
Initialized    true
Sealed         false
```

---

## 9. Ver los motores de secretos

```bash
vault secrets list
```

En modo desarrollo normalmente veremos una ruta:

```text
secret/
```

---

## 10. Verificar el motor KV v2

```bash
vault secrets list -detailed
```

Si `secret/` no existe:

```bash
vault secrets enable -path=secret -version=2 kv
```

Si Vault indica que ya existe, no debes recrearlo.

---

## 11. Guardar el primer secreto

```bash
vault kv put -mount=secret app   usuario=demo   password=ClaveTemporal123
```

Conceptualmente pasamos de:

```text
Aplicación
 └── password
```

a:

```text
Vault
 └── password
```

---

## 12. Leer el secreto

```bash
vault kv get -mount=secret app
```

Salida esperada aproximada:

```text
Key         Value
---         -----
usuario     demo
password    ClaveTemporal123
```

En este momento usamos `root`, que tiene privilegios totales.

---

## 13. Crear una política de mínimo privilegio

Crearemos una política que permita **solo lectura**:

```bash
vault policy write lector-app - <<EOF
path "secret/data/app" {
  capabilities = ["read"]
}
EOF
```

La ruta usa `secret/data/app` porque trabajamos con **KV v2**.

Verificar:

```bash
vault policy read lector-app
```

---

## 14. Crear un token limitado

```bash
vault token create -policy=lector-app
```

La salida mostrará un token parecido a:

```text
hvs.xxxxxxxxxxxxxxxxx
```

Guárdalo.

---

## 15. Guardar el token root y el token limitado

```bash
export ROOT_TOKEN='root'
export LIMITED_TOKEN='hvs.xxxxxxxxxxxxxxxxx'
```

Reemplaza el segundo valor por el token real.

Cambiar a la identidad limitada:

```bash
export VAULT_TOKEN=$LIMITED_TOKEN
```

---

## 16. Probar lectura permitida

```bash
vault kv get -mount=secret app
```

Debe funcionar.

---

## 17. Probar modificación denegada

```bash
vault kv put -mount=secret app   usuario=demo   password=NuevaClave456
```

La operación debería devolver:

```text
permission denied
```

Esto demuestra el principio de **mínimo privilegio**.

---

## 18. Revocar el token

Volver temporalmente a root:

```bash
export VAULT_TOKEN=$ROOT_TOKEN
```

Revocar el token limitado:

```bash
vault token revoke $LIMITED_TOKEN
```

---

## 19. Comprobar la revocación

Volver al token revocado:

```bash
export VAULT_TOKEN=$LIMITED_TOKEN
```

Intentar leer:

```bash
vault kv get -mount=secret app
```

La operación debe fallar.

---

## 20. ¿Qué acabamos de demostrar?

```text
1. Crear Vault
        |
        v
2. Guardar secreto
        |
        v
3. Leer secreto
        |
        v
4. Crear política
        |
        v
5. Crear token limitado
        |
        v
6. Leer → FUNCIONA
        |
        v
7. Modificar → DENEGADO
        |
        v
8. Revocar token
        |
        v
9. Leer → DENEGADO
```

---

## 21. Comparación: secreto estático vs secreto administrado

### Modelo tradicional

```text
config.txt

usuario=demo
password=ClaveTemporal123
```

Problemas:

- se puede copiar;
- puede quedar en backups;
- puede terminar en Git;
- es difícil saber quién lo utilizó;
- puede permanecer válido durante años.

### Modelo administrado

```text
Aplicación
     |
     | identidad
     v
   Vault
     |
     | política
     v
  secreto
```

Vault permite:

- centralizar;
- autorizar;
- auditar;
- revocar;
- rotar.

---

## 22. Relación con la Sesión 2

| Tecnología | Función principal |
|---|---|
| Vault | Administración de secretos |
| KMS | Administración de claves criptográficas |
| HSM | Protección fuerte de claves y operaciones criptográficas |

Ejemplo:

```text
Vault
 ├── password
 ├── API key
 └── token

KMS
 ├── AES key
 └── encryption keys

HSM
 └── claves criptográficas críticas
```

En arquitecturas reales estos componentes pueden integrarse.

---

## 23. Preguntas para discusión

1. ¿Qué riesgo existe al guardar una contraseña directamente dentro de una aplicación?
2. ¿Qué ventaja aporta centralizar el secreto en Vault?
3. ¿Por qué no deberíamos entregar un token root a una aplicación?
4. ¿Qué demuestra la política `lector-app`?
5. ¿Qué sucede después de revocar el token?
6. ¿Qué diferencia existe entre guardar un secreto y administrar el acceso al secreto?
7. ¿Qué ventaja tendría utilizar tokens con una duración limitada?

---

## 24. Actividad adicional

Modificar la política para permitir también actualización:

```hcl
path "secret/data/app" {
  capabilities = ["read", "update"]
}
```

Crear un nuevo token y comprobar:

```text
READ     permitido
UPDATE   permitido
DELETE   denegado
```

---

## 25. Limpieza del laboratorio

Salir del contenedor:

```bash
exit
```

Detener Vault:

```bash
docker stop vault-lab
```

Eliminar el contenedor:

```bash
docker rm vault-lab
```

Opcionalmente eliminar la imagen:

```bash
docker rmi hashicorp/vault
```

> Como utilizamos Vault en modo desarrollo, los secretos almacenados se perderán al eliminar el contenedor.

---

## 26. Comandos consolidados

### Terminal 1

```bash
docker run --name vault-lab   --cap-add=IPC_LOCK   -e 'VAULT_DEV_ROOT_TOKEN_ID=root'   -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200'   -p 8200:8200   hashicorp/vault
```

### Terminal 2

```bash
docker exec -it vault-lab sh
```

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'
```

```bash
vault status
```

```bash
vault kv put -mount=secret app   usuario=demo   password=ClaveTemporal123
```

```bash
vault kv get -mount=secret app
```

```bash
vault policy write lector-app - <<EOF
path "secret/data/app" {
  capabilities = ["read"]
}
EOF
```

```bash
vault token create -policy=lector-app
```

```bash
export ROOT_TOKEN='root'
export LIMITED_TOKEN='hvs.xxxxxxxxxxxxxxxxx'
export VAULT_TOKEN=$LIMITED_TOKEN
```

```bash
vault kv get -mount=secret app
```

```bash
vault kv put -mount=secret app   usuario=demo   password=NuevaClave456
```

```bash
export VAULT_TOKEN=$ROOT_TOKEN
vault token revoke $LIMITED_TOKEN
```

```bash
export VAULT_TOKEN=$LIMITED_TOKEN
vault kv get -mount=secret app
```

---

## 27. Resultado esperado

Al finalizar, el estudiante debería poder explicar:

- por qué los secretos no deberían quedar escritos directamente en el código;
- qué función cumple un gestor de secretos;
- qué significa mínimo privilegio;
- cómo una política limita acciones;
- qué función cumple un token;
- qué significa revocar una credencial;
- por qué la gestión de secretos es más que almacenar contraseñas.

---

## 28. Mensaje final

> **No basta con proteger un secreto. Debemos controlar quién puede utilizarlo, para qué operación, durante cuánto tiempo y tener la capacidad de retirar ese acceso cuando sea necesario.**
