# Laboratorio guiado — Criptografía Homomórfica con CKKS en Kali Linux

## Módulo 10 · Protección de Datos y Criptografía Aplicada  
### Sesión 5 · Criptografía Homomórfica y Confidential Computing

---

# 1. Objetivo del laboratorio

El objetivo es comprobar de forma práctica la idea central de la **criptografía homomórfica**:

> **realizar operaciones sobre datos cifrados sin tener que descifrarlos durante el cálculo.**

En el laboratorio vamos a:

1. Preparar Kali Linux con Docker.
2. Ejecutar Ubuntu 24.04 dentro de un contenedor.
3. Instalar OpenFHE-Python.
4. Crear un contexto CKKS.
5. Generar claves.
6. Cifrar dos vectores numéricos.
7. Sumar los ciphertext.
8. Multiplicar los ciphertext.
9. Descifrar únicamente al final.
10. Comparar el resultado cifrado con el resultado esperado en texto claro.

---

# 2. ¿Qué estamos demostrando?

En un modelo tradicional:

```text
Dato cifrado
    ↓
Descifrar
    ↓
Procesar
    ↓
Volver a cifrar
```

El problema es que durante el procesamiento el dato puede quedar visible en memoria.

Con Homomorphic Encryption buscamos:

```text
Dato cifrado
    ↓
Procesar SIN descifrar
    ↓
Resultado cifrado
    ↓
Descifrar solamente al final
```

Para este laboratorio utilizaremos **CKKS**, un esquema homomórfico especialmente útil para números reales aproximados.

---

# 3. Escenario didáctico

Supongamos que una empresa tiene dos conjuntos de valores:

```text
Vector A = [1.5, 2.0, 3.5, 4.0]
Vector B = [2.0, 3.0, 4.0, 5.0]
```

Queremos obtener:

### Suma

```text
[3.5, 5.0, 7.5, 9.0]
```

### Multiplicación elemento a elemento

```text
[3.0, 6.0, 14.0, 20.0]
```

Pero queremos simular que el servidor de cálculo **no conoce los valores originales**.

---

# 4. Arquitectura conceptual

```text
┌─────────────────────────┐
│ Propietario del dato    │
│                         │
│ - genera claves         │
│ - conserva secret key   │
│ - cifra los datos       │
└───────────┬─────────────┘
            │ ciphertext
            ▼
┌─────────────────────────┐
│ Servidor de cálculo     │
│                         │
│ - NO tiene secret key   │
│ - suma ciphertext       │
│ - multiplica ciphertext │
└───────────┬─────────────┘
            │ resultado cifrado
            ▼
┌─────────────────────────┐
│ Propietario             │
│ - descifra el resultado │
└─────────────────────────┘
```

En este laboratorio todo ocurrirá en el mismo contenedor, pero conceptualmente debemos separar esos roles.

---

# 5. Requisitos previos

En Kali Linux necesitamos:

- Docker instalado.
- Acceso a Internet.
- Terminal con privilegios suficientes para usar Docker.

Comprobar Docker:

```bash
docker --version
```

También:

```bash
docker info
```

Si `docker info` funciona, el motor está activo.

---

# 6. Si Docker no está instalado

En Kali:

```bash
sudo apt update
sudo apt install -y docker.io
```

Iniciar el servicio:

```bash
sudo systemctl enable --now docker
```

Comprobar:

```bash
docker --version
docker ps
```

Si tu usuario no tiene permiso para usar Docker sin `sudo`, usa por ejemplo:

```bash
sudo docker ps
```

---

# 7. Descargar Ubuntu 24.04

Desde Kali:

```bash
docker pull ubuntu:24.04
```

Verificar:

```bash
docker images
```

Deberías observar:

```text
ubuntu    24.04
```

---

# 8. Levantar el contenedor del laboratorio

Ejecuta:

```bash
docker run --rm -it \
  --name fhe-lab \
  ubuntu:24.04 bash
```

El prompt cambiará a algo parecido a:

```text
root@xxxxxxxx:/#
```

Eso significa que ya estamos **dentro del contenedor Ubuntu**.

> Desde este punto, `apt`, `python` y `pip` se ejecutan dentro del contenedor.

---

# 9. Instalar Python y herramientas

Dentro del contenedor:

```bash
apt update
apt install -y python3 python3-pip python3-venv nano
```

Verificar:

```bash
python3 --version
```

En Ubuntu 24.04 normalmente veremos Python 3.12.

---

# 10. Crear un entorno virtual

```bash
python3 -m venv /opt/fhe
source /opt/fhe/bin/activate
```

El prompt puede mostrar:

```text
(fhe) root@xxxxxxxx:/#
```

Verificar:

```bash
python --version
pip --version
```

---

# 11. Actualizar pip

```bash
pip install --upgrade pip
```

---

# 12. Instalar OpenFHE-Python

Para Ubuntu 24.04 / Python 3.12 puedes intentar directamente:

```bash
pip install openfhe==1.5.1.0.24.4
```

Si esa versión exacta no estuviera disponible al ejecutar la práctica:

```bash
pip install "openfhe>=1.5,<1.6"
```

Verificar:

```bash
python -c "import openfhe; print('OpenFHE listo')"
```

Resultado esperado:

```text
OpenFHE listo
```

---

# 13. Crear carpeta de trabajo

```bash
mkdir -p /tmp/fhe-lab
cd /tmp/fhe-lab
pwd
```

Debe mostrar:

```text
/tmp/fhe-lab
```

---

# 14. Crear el script Python

```bash
nano ckks_demo.py
```

Pega el siguiente código completo.

---

# 15. Código completo del laboratorio

```python
from openfhe import *

print("=" * 65)
print("LABORATORIO CKKS - CRIPTOGRAFIA HOMOMORFICA")
print("=" * 65)

# 1. Parámetros CKKS
params = CCParamsCKKSRNS()
params.SetMultiplicativeDepth(2)
params.SetScalingModSize(50)
params.SetBatchSize(8)

print("\n[1] Parámetros CKKS configurados")

# 2. Contexto criptográfico
cc = GenCryptoContext(params)
cc.Enable(PKESchemeFeature.PKE)
cc.Enable(PKESchemeFeature.KEYSWITCH)
cc.Enable(PKESchemeFeature.LEVELEDSHE)

print("[2] Contexto criptográfico creado")

# 3. Generación de claves
keys = cc.KeyGen()
cc.EvalMultKeyGen(keys.secretKey)

print("[3] Claves generadas")
print("    - publicKey  -> puede utilizarse para cifrar")
print("    - secretKey  -> debe permanecer con el propietario")
print("    - EvalMultKey -> permite multiplicaciones cifradas")

# 4. Datos originales
x1 = [1.5, 2.0, 3.5, 4.0]
x2 = [2.0, 3.0, 4.0, 5.0]

print("\n[4] Datos originales")
print("Vector A:", x1)
print("Vector B:", x2)

# 5. Codificación CKKS
p1 = cc.MakeCKKSPackedPlaintext(x1)
p2 = cc.MakeCKKSPackedPlaintext(x2)

print("\n[5] Datos codificados como plaintext CKKS")

# 6. Cifrado
c1 = cc.Encrypt(keys.publicKey, p1)
c2 = cc.Encrypt(keys.publicKey, p2)

print("\n[6] DATOS CIFRADOS")
print("El servidor recibe c1 y c2.")
print("El servidor NO necesita la secretKey.")

# 7. Operaciones homomórficas
print("\n[7] OPERANDO SOBRE CIPHERTEXT...")

c_suma = cc.EvalAdd(c1, c2)
c_mult = cc.EvalMult(c1, c2)

print("Suma homomórfica realizada.")
print("Multiplicación homomórfica realizada.")
print("Hasta este punto NO hemos descifrado los datos.")

# 8. Descifrar solamente al final
print("\n[8] DESCIFRANDO RESULTADOS...")

# En los builds actuales del wrapper Python esta forma suele estar soportada.
r_suma = cc.Decrypt(c_suma, keys.secretKey)
r_mult = cc.Decrypt(c_mult, keys.secretKey)

r_suma.SetLength(4)
r_mult.SetLength(4)

# 9. Resultados esperados
esperada_suma = [a + b for a, b in zip(x1, x2)]
esperada_mult = [a * b for a, b in zip(x1, x2)]

print("\n" + "=" * 65)
print("RESULTADOS")
print("=" * 65)

print("\nEntrada A:")
print(x1)

print("\nEntrada B:")
print(x2)

print("\nSuma esperada:")
print(esperada_suma)

print("\nSuma obtenida con CKKS:")
print(r_suma.GetCKKSPackedValue())

print("\nMultiplicación esperada:")
print(esperada_mult)

print("\nMultiplicación obtenida con CKKS:")
print(r_mult.GetCKKSPackedValue())

print("\n" + "=" * 65)
print("CONCLUSIÓN")
print("=" * 65)

print("""
La suma y la multiplicación fueron realizadas sobre ciphertext.
La secretKey solamente fue utilizada al final para recuperar
los resultados.

Esto demuestra el principio básico de Homomorphic Encryption:
CALCULAR SIN DESCIFRAR LOS DATOS DURANTE EL PROCESAMIENTO.
""")
```

---

# 16. Guardar el archivo en nano

Usa:

```text
CTRL + O
```

Presiona `Enter` y luego:

```text
CTRL + X
```

---

# 17. Verificar el archivo

```bash
ls -l
head -20 ckks_demo.py
```

Debe aparecer `ckks_demo.py`.

---

# 18. Ejecutar el laboratorio

```bash
python ckks_demo.py
```

---

# 19. Salida esperada

La salida tendrá una estructura parecida a:

```text
=================================================================
LABORATORIO CKKS - CRIPTOGRAFIA HOMOMORFICA
=================================================================

[1] Parámetros CKKS configurados
[2] Contexto criptográfico creado
[3] Claves generadas

[4] Datos originales
Vector A: [1.5, 2.0, 3.5, 4.0]
Vector B: [2.0, 3.0, 4.0, 5.0]

[5] Datos codificados como plaintext CKKS

[6] DATOS CIFRADOS
El servidor recibe c1 y c2.
El servidor NO necesita la secretKey.

[7] OPERANDO SOBRE CIPHERTEXT...
Suma homomórfica realizada.
Multiplicación homomórfica realizada.
Hasta este punto NO hemos descifrado los datos.

[8] DESCIFRANDO RESULTADOS...
```

---

# 20. Resultado esperado de la suma

```text
[1.5, 2.0, 3.5, 4.0]
+
[2.0, 3.0, 4.0, 5.0]
=
[3.5, 5.0, 7.5, 9.0]
```

La salida CKKS puede verse aproximadamente así:

```text
(3.500000..., 5.000000..., 7.500000..., 9.000000...)
```

---

# 21. Resultado esperado de la multiplicación

```text
1.5 × 2.0 = 3.0
2.0 × 3.0 = 6.0
3.5 × 4.0 = 14.0
4.0 × 5.0 = 20.0
```

Resultado:

```text
[3.0, 6.0, 14.0, 20.0]
```

CKKS puede mostrar valores como:

```text
(3.000001..., 5.999999..., 14.00000..., 19.99999...)
```

Esto es normal.

---

# 22. ¿Por qué los valores no siempre son exactos?

CKKS utiliza **aritmética aproximada**.

Por ejemplo, podríamos esperar:

```text
20.0
```

y obtener:

```text
19.9999998
```

Eso no significa que el laboratorio falló.

CKKS intercambia una pequeña cantidad de precisión por la capacidad de trabajar eficientemente con números reales cifrados.

Esto lo hace interesante para:

- estadística;
- machine learning;
- análisis financiero;
- procesamiento de señales;
- inferencia sobre datos cifrados.

---

# 23. ¿Dónde ocurrió realmente la parte homomórfica?

Estas líneas son el corazón del laboratorio:

```python
c_suma = cc.EvalAdd(c1, c2)
c_mult = cc.EvalMult(c1, c2)
```

Usamos `c1` y `c2`, que son **ciphertexts**.

No usamos `x1` y `x2` para realizar esas operaciones.

---

# 24. ¿Cuándo utilizamos la secret key?

Solamente durante el descifrado final:

```python
r_suma = cc.Decrypt(c_suma, keys.secretKey)
r_mult = cc.Decrypt(c_mult, keys.secretKey)
```

Conceptualmente:

```text
PROPIETARIO
   │
   ├── conserva secretKey
   ▼
SERVIDOR
   │
   ├── recibe ciphertext
   ├── calcula
   └── NO necesita secretKey
```

---

# 25. Qué demuestra este laboratorio

## Modelo tradicional

```text
Datos
  ↓
Cifrar
  ↓
Enviar
  ↓
Descifrar en cloud
  ↓
Calcular
```

El proveedor puede ver los datos durante el cálculo.

## Modelo homomórfico

```text
Datos
  ↓
Cifrar
  ↓
Enviar ciphertext
  ↓
Calcular sobre ciphertext
  ↓
Resultado cifrado
  ↓
Propietario descifra
```

El servidor de cálculo no necesita conocer los valores originales.

---

# 26. Experimento adicional — cambiar los valores

Edita:

```bash
nano ckks_demo.py
```

Cambia:

```python
x1 = [1.5, 2.0, 3.5, 4.0]
x2 = [2.0, 3.0, 4.0, 5.0]
```

por:

```python
x1 = [10.0, 20.0, 30.0, 40.0]
x2 = [2.0, 3.0, 4.0, 5.0]
```

Ejecuta nuevamente:

```bash
python ckks_demo.py
```

Resultados esperados:

```text
Suma:           [12, 23, 34, 45]
Multiplicación: [20, 60, 120, 200]
```

---

# 27. Experimento adicional — multiplicar por una constante

Después de:

```python
c_mult = cc.EvalMult(c1, c2)
```

agrega:

```python
c_doble = cc.EvalMult(c1, 2.0)
```

Después descifra:

```python
r_doble = cc.Decrypt(c_doble, keys.secretKey)
r_doble.SetLength(4)
```

Imprime:

```python
print("\nVector A multiplicado por 2:")
print(r_doble.GetCKKSPackedValue())
```

Resultado esperado aproximado:

```text
[3.0, 4.0, 7.0, 8.0]
```

---

# 28. Preguntas para los estudiantes

1. ¿En qué momento se cifran los datos?
2. ¿Quién debería conservar la `secretKey`?
3. ¿El servidor necesita la `secretKey` para sumar?
4. ¿El servidor necesita la `secretKey` para multiplicar?
5. ¿En qué momento ocurre el descifrado?
6. ¿Por qué CKKS puede producir pequeñas diferencias decimales?
7. ¿Qué ventaja tiene HE frente al modelo tradicional?
8. ¿Cuál es la principal desventaja de HE?
9. ¿En qué escenarios tendría sentido utilizar CKKS?
10. ¿Qué diferencia conceptual existe entre HE y Confidential Computing?

---

# 29. Respuestas esperadas

1. **Cifrado:** antes de enviar los datos al entorno de cálculo.
2. **Secret key:** el propietario autorizado de los datos.
3. **Suma:** no necesita la secret key.
4. **Multiplicación:** tampoco; puede necesitar claves de evaluación.
5. **Descifrado:** después de recibir el resultado cifrado.
6. **Aproximación:** CKKS trabaja con aritmética aproximada.
7. **Ventaja:** reduce la necesidad de revelar los datos al servidor de cálculo.
8. **Desventaja:** mayor costo computacional, tamaño y complejidad.
9. **Casos CKKS:** estadística, ML, análisis vectorial, señales y cálculos numéricos aproximados.
10. **HE vs CC:** HE mantiene el dato cifrado durante el cálculo; Confidential Computing permite procesarlo en texto claro dentro de un TEE protegido.

---

# 30. Troubleshooting — `docker: command not found`

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

---

# 31. Troubleshooting — `permission denied` con Docker

Prueba:

```bash
sudo docker ps
```

Opcionalmente:

```bash
sudo usermod -aG docker $USER
```

Luego cierra sesión y vuelve a entrar.

---

# 32. Troubleshooting — nombre de contenedor existente

Si aparece:

```text
Conflict. The container name "/fhe-lab" is already in use
```

Ejecuta:

```bash
docker ps -a
docker rm -f fhe-lab
```

Luego vuelve a iniciar el contenedor.

---

# 33. Troubleshooting — `pip` no permite instalar

Asegúrate de estar dentro del entorno virtual:

```bash
source /opt/fhe/bin/activate
```

Verifica:

```bash
which python
which pip
```

Deberían apuntar a `/opt/fhe/bin/...`.

---

# 34. Troubleshooting — no encuentra la versión de OpenFHE

```bash
pip install --upgrade pip
pip index versions openfhe
```

Luego prueba:

```bash
pip install "openfhe>=1.5,<1.6"
```

---

# 35. Troubleshooting — `ModuleNotFoundError: No module named 'openfhe'`

```bash
pip list | grep -i openfhe
python -c "import openfhe; print(openfhe)"
```

Si no aparece:

```bash
pip install --force-reinstall "openfhe>=1.5,<1.6"
```

---

# 36. Troubleshooting — error en `Decrypt`

El wrapper de OpenFHE ha mostrado más de una forma de invocación según ejemplos/versiones.

La guía usa:

```python
cc.Decrypt(ciphertext, keys.secretKey)
```

Si tu build devuelve un error de firma, prueba:

```python
cc.Decrypt(keys.secretKey, ciphertext)
```

Ejemplo:

```python
r_suma = cc.Decrypt(keys.secretKey, c_suma)
r_mult = cc.Decrypt(keys.secretKey, c_mult)
```

Después:

```python
r_suma.SetLength(4)
r_mult.SetLength(4)
```

---

# 37. Troubleshooting — aparecen muchos valores

CKKS utiliza packing y puede manejar más slots que nuestros cuatro datos.

Por eso usamos:

```python
r_suma.SetLength(4)
r_mult.SetLength(4)
```

Así mostramos solo los cuatro valores que interesan.

---

# 38. Troubleshooting — pequeñas diferencias decimales

Una salida como:

```text
3.499999998
```

en lugar de:

```text
3.5
```

es normal con CKKS.

En una aplicación real se utiliza tolerancia, por ejemplo:

```python
abs(resultado - 3.5) < 0.001
```

---

# 39. Limpiar el laboratorio

Salir del contenedor:

```bash
exit
```

Como utilizamos `--rm`, Docker eliminará automáticamente el contenedor.

Comprobar:

```bash
docker ps -a
```

La imagen Ubuntu queda instalada para reutilizarla.

---

# 40. Comandos consolidados

## En Kali

```bash
docker --version
docker pull ubuntu:24.04

docker run --rm -it \
  --name fhe-lab \
  ubuntu:24.04 bash
```

## Dentro de Ubuntu

```bash
apt update
apt install -y python3 python3-pip python3-venv nano

python3 -m venv /opt/fhe
source /opt/fhe/bin/activate

pip install --upgrade pip
pip install openfhe==1.5.1.0.24.4

python -c "import openfhe; print('OpenFHE listo')"

mkdir -p /tmp/fhe-lab
cd /tmp/fhe-lab
nano ckks_demo.py
```

Ejecutar:

```bash
python ckks_demo.py
```

---

# 41. Flujo completo

```text
KALI LINUX
     │
     ▼
DOCKER
     │
     ▼
UBUNTU 24.04
     │
     ▼
PYTHON 3.12
     │
     ▼
OPENFHE
     │
     ▼
CKKS
     │
     ├── Generar claves
     ├── Codificar
     ├── Cifrar
     ├── Sumar ciphertext
     ├── Multiplicar ciphertext
     ▼
RESULTADO CIFRADO
     │
     ▼
DESCIFRAR
     │
     ▼
RESULTADO
```

---

# 42. Mensaje final para la clase

> **En este laboratorio el servidor pudo sumar y multiplicar información sin necesitar la clave privada ni descifrar los valores durante el procesamiento.**

Eso demuestra el principio central de Homomorphic Encryption:

```text
CALCULAR
   +
DATOS CIFRADOS
   =
RESULTADOS ÚTILES
SIN REVELAR LOS DATOS DURANTE EL CÁLCULO
```

---

# 43. Relación con Confidential Computing

```text
HOMOMORPHIC ENCRYPTION

Dato permanece cifrado
durante el cálculo
```

frente a:

```text
CONFIDENTIAL COMPUTING

Dato puede estar en plaintext
dentro de un TEE protegido
```

Ambas tecnologías buscan proteger **DATA IN USE**, pero usan modelos de confianza diferentes.

---

# 44. Resultado de aprendizaje

Al finalizar el estudiante debería poder explicar:

- qué significa `data in use`;
- qué es Homomorphic Encryption;
- qué es CKKS;
- qué función tiene la public key;
- qué función tiene la secret key;
- qué son las evaluation keys;
- por qué el servidor puede calcular sin descifrar;
- por qué CKKS produce resultados aproximados;
- qué ventaja ofrece FHE;
- qué costos introduce;
- cuándo podría utilizarse;
- cómo se diferencia de Confidential Computing.

---

## Referencias técnicas

- OpenFHE — biblioteca open source de Fully Homomorphic Encryption.
- OpenFHE Python — wrapper oficial para Python.
- OpenFHE CKKS examples — ejemplos oficiales de operaciones con números reales.
- HomomorphicEncryption.org — Homomorphic Encryption Standard.
- Confidential Computing Consortium — conceptos de protección de datos en uso.

> **Nota:** este laboratorio utiliza parámetros educativos y está diseñado para aprendizaje. Una implementación de producción requiere selección formal de parámetros de seguridad, pruebas de rendimiento, gestión de claves y revisión criptográfica.
