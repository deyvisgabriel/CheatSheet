# Laboratorio práctico de criptografía post-cuántica con ML-KEM y ML-DSA

## Finalidad del laboratorio

El objetivo de este laboratorio es **demostrar de forma práctica cómo funcionan dos de los principales mecanismos de criptografía post-cuántica estandarizados actualmente**:

- **ML-KEM-768**, utilizado para establecer un secreto compartido de forma segura.
- **ML-DSA-65**, utilizado para firmar digitalmente información y detectar cualquier modificación posterior.

Durante la práctica se comprobará que:

1. Dos partes pueden obtener exactamente el mismo secreto utilizando ML-KEM sin necesidad de transmitir directamente dicho secreto.
2. Un mensaje puede ser firmado mediante ML-DSA.
3. La firma será válida mientras el mensaje permanezca intacto.
4. Si el contenido se modifica, incluso mínimamente, la verificación de la firma fallará.
5. La criptografía post-cuántica introduce nuevos tamaños de claves, firmas y artefactos que deben considerarse en una futura migración.

> **Idea principal:** el laboratorio no busca profundizar en las matemáticas internas de los algoritmos, sino comprender su funcionamiento práctico y sus diferencias respecto de los mecanismos criptográficos tradicionales.

---

# 1. Preparar Kali Linux

Abrir una terminal en Kali Linux.

Verificar si Docker se encuentra instalado:

```bash
docker --version
```

Debería mostrarse una salida similar a:

```text
Docker version 27.x.x
```

Si Docker no se encuentra instalado:

```bash
sudo apt update
sudo apt install docker.io -y
```

Iniciar el servicio:

```bash
sudo systemctl enable --now docker
```

Comprobar su funcionamiento:

```bash
sudo docker run hello-world
```

---

# 2. Descargar el entorno de criptografía post-cuántica

Descargar la imagen preparada por Open Quantum Safe:

```bash
sudo docker pull openquantumsafe/oqs-ossl3
```

Este contenedor proporciona un entorno con OpenSSL y soporte para algoritmos de criptografía post-cuántica.

---

# 3. Ingresar al contenedor

Ejecutar:

```bash
sudo docker run --rm -it \
  --entrypoint /bin/sh \
  openquantumsafe/oqs-ossl3
```

Una vez dentro aparecerá un prompt similar a:

```text
/ #
```

A partir de este momento, los comandos OpenSSL se ejecutarán **dentro del contenedor**.

---

# 4. Verificar la versión de OpenSSL

Ejecutar:

```bash
openssl version
```

También puede verificarse qué proveedores criptográficos están disponibles:

```bash
openssl list -providers
```

---

# 5. Verificar soporte para ML-KEM

Ejecutar:

```bash
openssl list -kem-algorithms | grep -i ML-KEM
```

Deberían aparecer algoritmos similares a:

```text
ML-KEM-512
ML-KEM-768
ML-KEM-1024
```

Para este laboratorio utilizaremos:

```text
ML-KEM-768
```

---

# 6. Verificar soporte para ML-DSA

Ejecutar:

```bash
openssl list -signature-algorithms | grep -i ML-DSA
```

Deberían aparecer:

```text
ML-DSA-44
ML-DSA-65
ML-DSA-87
```

Para el laboratorio utilizaremos:

```text
ML-DSA-65
```

---

# 7. Crear el directorio de trabajo

Dentro del contenedor:

```bash
mkdir -p /tmp/pqc-lab
cd /tmp/pqc-lab
```

Verificar:

```bash
pwd
```

Resultado esperado:

```text
/tmp/pqc-lab
```

---

# Parte I. ML-KEM-768

## 8. ¿Qué vamos a demostrar?

ML-KEM permite que dos participantes obtengan **el mismo secreto compartido**.

Para facilitar la explicación utilizaremos dos participantes:

- **Alicia:** posee una clave privada y publica su clave pública.
- **Bob:** utiliza la clave pública de Alicia para generar un secreto y encapsularlo.

Bob enviará el encapsulado a Alicia, pero **no enviará directamente el secreto**.

---

# 9. Generar la clave privada ML-KEM de Alicia

Ejecutar:

```bash
openssl genpkey \
  -algorithm ML-KEM-768 \
  -out mlkem768.pem
```

Comprobar:

```bash
ls -lh
```

Debería existir:

```text
mlkem768.pem
```

Este archivo contiene la clave privada.

---

# 10. Extraer la clave pública

Ejecutar:

```bash
openssl pkey \
  -in mlkem768.pem \
  -pubout \
  -out mlkem768_pub.pem
```

Comprobar:

```bash
ls -lh
```

Ahora existirán:

```text
mlkem768.pem
mlkem768_pub.pem
```

Conceptualmente:

```text
Alicia
 ├── Clave privada → mlkem768.pem
 └── Clave pública → mlkem768_pub.pem
```

La clave pública puede entregarse a Bob.

---

# 11. Bob encapsula un secreto

Ejecutar:

```bash
openssl pkeyutl -encap \
  -inkey mlkem768_pub.pem \
  -pubin \
  -out ciphertext.bin \
  -secret secreto_bob.bin
```

Este comando genera dos elementos:

```text
ciphertext.bin
secreto_bob.bin
```

Comprobar:

```bash
ls -lh
```

Interpretación:

```text
Clave pública de Alicia
          │
          ▼
       ML-KEM
       /    \
      /      \
     ▼        ▼
Secreto    Ciphertext
de Bob
```

El archivo:

```text
secreto_bob.bin
```

queda en poder de Bob.

El archivo:

```text
ciphertext.bin
```

se envía a Alicia.

---

# 12. Visualizar el secreto de Bob

Ejecutar:

```bash
xxd secreto_bob.bin
```

Se observará información hexadecimal similar a:

```text
00000000: a4f7 26b3 8c21 ...
```

Este valor no debe interpretarse como una contraseña legible.

Es material criptográfico generado aleatoriamente.

---

# 13. Alicia decapsula el secreto

Alicia recibe únicamente:

```text
ciphertext.bin
```

y utiliza su clave privada:

```text
mlkem768.pem
```

Ejecutar:

```bash
openssl pkeyutl -decap \
  -inkey mlkem768.pem \
  -in ciphertext.bin \
  -secret secreto_alicia.bin
```

Comprobar:

```bash
ls -lh
```

Ahora existirá:

```text
secreto_alicia.bin
```

---

# 14. Visualizar el secreto de Alicia

Ejecutar:

```bash
xxd secreto_alicia.bin
```

El contenido debería ser exactamente igual al observado anteriormente en:

```text
secreto_bob.bin
```

---

# 15. Comparar ambos secretos

Ejecutar:

```bash
cmp secreto_bob.bin secreto_alicia.bin
```

Si ambos archivos son idénticos, `cmp` no mostrará ninguna salida.

Para obtener un resultado más visual:

```bash
cmp secreto_bob.bin secreto_alicia.bin \
  && echo "SECRETOS IGUALES"
```

Resultado esperado:

```text
SECRETOS IGUALES
```

---

# 16. Comprobar mediante SHA-256

Ejecutar:

```bash
sha256sum secreto_bob.bin secreto_alicia.bin
```

Ejemplo:

```text
8f7a...  secreto_bob.bin
8f7a...  secreto_alicia.bin
```

Los hashes deberían ser idénticos.

Esto demuestra que:

```text
Secreto de Bob = Secreto de Alicia
```

---

# 17. ¿Qué acaba de ocurrir?

El proceso puede representarse así:

```text
BOB                                         ALICIA

                       Clave pública
                  <---------------------

                  Encapsulación ML-KEM

Secreto Bob
     +
Ciphertext
                       -------------------->
                                              Ciphertext
                                                  +
                                            Clave privada
                                                  │
                                                  ▼
                                            Decapsulación
                                                  │
                                                  ▼
                                            Secreto Alicia

                  Secreto Bob = Secreto Alicia
```

Lo importante es observar que:

> Bob nunca tuvo que enviar directamente el secreto compartido a Alicia.

---

# Parte II. ML-DSA-65

# 18. ¿Qué vamos a demostrar?

Ahora utilizaremos ML-DSA para:

- Firmar un mensaje.
- Comprobar que el mensaje es auténtico.
- Detectar modificaciones.

Utilizaremos como ejemplo una transferencia bancaria.

---

# 19. Crear el mensaje

Ejecutar:

```bash
echo "Transferencia autorizada: Bs 1000" > mensaje.txt
```

Visualizar:

```bash
cat mensaje.txt
```

Resultado:

```text
Transferencia autorizada: Bs 1000
```

---

# 20. Generar la clave privada ML-DSA

Ejecutar:

```bash
openssl genpkey \
  -algorithm ML-DSA-65 \
  -out mldsa65.pem
```

---

# 21. Extraer la clave pública

Ejecutar:

```bash
openssl pkey \
  -in mldsa65.pem \
  -pubout \
  -out mldsa65_pub.pem
```

Comprobar:

```bash
ls -lh
```

Deberían existir:

```text
mldsa65.pem
mldsa65_pub.pem
```

---

# 22. Firmar el mensaje

Ejecutar:

```bash
openssl pkeyutl -sign \
  -in mensaje.txt \
  -inkey mldsa65.pem \
  -out firma.bin
```

Ahora existen:

```text
mensaje.txt
firma.bin
```

Conceptualmente:

```text
Mensaje
   +
Clave privada
   │
   ▼
 ML-DSA
   │
   ▼
 Firma
```

---

# 23. Verificar la firma

Ejecutar:

```bash
openssl pkeyutl -verify \
  -in mensaje.txt \
  -pubin \
  -inkey mldsa65_pub.pem \
  -sigfile firma.bin
```

Resultado esperado:

```text
Signature Verified Successfully
```

Esto demuestra que:

- El mensaje corresponde con la firma.
- La firma fue generada utilizando la clave privada correspondiente.

---

# 24. Simular una manipulación

Ahora modificaremos el mensaje.

Original:

```text
Transferencia autorizada: Bs 1000
```

Nuevo valor:

```text
Transferencia autorizada: Bs 9000
```

Ejecutar:

```bash
echo "Transferencia autorizada: Bs 9000" > mensaje.txt
```

Comprobar:

```bash
cat mensaje.txt
```

Resultado:

```text
Transferencia autorizada: Bs 9000
```

---

# 25. Verificar nuevamente la firma

Ejecutar exactamente el mismo comando:

```bash
openssl pkeyutl -verify \
  -in mensaje.txt \
  -pubin \
  -inkey mldsa65_pub.pem \
  -sigfile firma.bin
```

Ahora la verificación debería fallar.

Resultado esperado similar a:

```text
Signature Verification Failure
```

La firma era válida para:

```text
Transferencia autorizada: Bs 1000
```

pero no para:

```text
Transferencia autorizada: Bs 9000
```

---

# 26. ¿Qué demuestra esta prueba?

ML-DSA permite detectar modificaciones en la información.

```text
Mensaje original
      +
Firma
      │
      ▼
Verificación
      │
      ▼
     OK
```

Pero:

```text
Mensaje modificado
      +
Firma original
      │
      ▼
Verificación
      │
      ▼
    FALLA
```

Incluso un cambio pequeño modifica completamente el resultado de la verificación.

---

# Parte III. Observar artefactos y tamaños

# 27. Listar los archivos generados

Ejecutar:

```bash
ls -lh
```

También podemos visualizar solamente los principales artefactos:

```bash
ls -lh \
  mlkem768.pem \
  mlkem768_pub.pem \
  ciphertext.bin \
  secreto_bob.bin \
  secreto_alicia.bin \
  mldsa65.pem \
  mldsa65_pub.pem \
  firma.bin
```

---

# 28. Observar los tamaños exactos

Ejecutar:

```bash
wc -c *
```

También puede utilizarse:

```bash
du -b *
```

Esto permite observar el tamaño de:

- Claves privadas.
- Claves públicas.
- Ciphertext.
- Secretos.
- Firmas.

---

# 29. Comparar con RSA 3072

Como comparación adicional, generar una clave RSA tradicional:

```bash
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:3072 \
  -out rsa3072.pem
```

Comparar tamaños:

```bash
ls -lh rsa3072.pem mldsa65.pem firma.bin
```

También puede ejecutarse:

```bash
wc -c rsa3072.pem mldsa65.pem firma.bin
```

---

# 30. Interpretación de los tamaños

No debemos concluir simplemente:

> “La criptografía post-cuántica pesa más.”

Cada archivo cumple una función distinta.

Por ejemplo:

```text
mlkem768.pem       → clave privada
mlkem768_pub.pem   → clave pública
ciphertext.bin     → encapsulado ML-KEM
secreto_bob.bin    → secreto compartido
mldsa65.pem        → clave privada de firma
mldsa65_pub.pem    → clave pública
firma.bin          → firma digital
rsa3072.pem        → clave privada RSA
```

La migración hacia criptografía post-cuántica puede afectar:

- Tamaños de claves.
- Tamaños de firmas.
- Certificados digitales.
- Protocolos de red.
- Almacenamiento.
- Ancho de banda.
- Hardware criptográfico.
- Aplicaciones existentes.

---

# Resumen del laboratorio

## ML-KEM-768

Su objetivo es:

```text
ESTABLECER UN SECRETO COMPARTIDO
```

El resultado principal fue:

```text
Secreto Bob = Secreto Alicia
```

sin que Bob tuviera que transmitir directamente el secreto.

---

## ML-DSA-65

Su objetivo es:

```text
FIRMAR INFORMACIÓN
```

y permitir comprobar:

```text
AUTENTICIDAD + INTEGRIDAD
```

Resultado:

```text
Mensaje original
      ↓
Firma válida
```

pero:

```text
Mensaje modificado
      ↓
Firma inválida
```

---

# Conclusión

Este laboratorio permite observar dos funciones criptográficas diferentes:

| Algoritmo | Función principal |
|---|---|
| **ML-KEM-768** | Establecer un secreto compartido |
| **ML-DSA-65** | Crear y verificar firmas digitales |

La idea fundamental es:

```text
ML-KEM
   ↓
¿Cómo acordamos una clave de forma segura?

ML-DSA
   ↓
¿Cómo demostramos quién firmó algo y que no fue modificado?
```

Ambos algoritmos forman parte de la transición hacia sistemas preparados frente a futuras capacidades de computación cuántica.
