# Sesión 6 — Laboratorio en Kali Linux

**Módulo 10 · Protección de Datos y Criptografía Aplicada**  
**Temas:** Secure Multi-Party Computation (MPC), Zero-Knowledge Proofs (ZKP), blockchain y DLT.

## Objetivo

Mostrar en una única práctica tres ideas:

1. Tres bancos pueden reconstruir **una suma conjunta (55)** utilizando fragmentos matemáticos en lugar de entregar sus cifras individuales a otro banco.
2. Una persona puede **probar conocimiento de un secreto** mediante compromiso, desafío y respuesta, sin transmitir el secreto en el procedimiento demostrado.
3. Un registro con **hashes encadenados** permite detectar que alguien cambió un evento.

> **Importante:** Son demostraciones educativas. MPC se simula en un solo proceso, el ejemplo ZKP usa parámetros deliberadamente inseguros y el registro es una cadena local de hashes, **no una blockchain distribuida**. No utilizar con secretos o datos reales.

## Requisitos

Kali Linux, Python 3 y una terminal. **No necesita Docker ni instalar bibliotecas.**

## Paso 1 — Preparación

```bash
python3 --version
mkdir -p ~/sesion6-privacidad
cd ~/sesion6-privacidad
```

Descarga el archivo `sesion06_demo.py` que acompaña esta guía y guárdalo en la carpeta indicada (también puedes ejecutarlo desde la carpeta de descargas).

## Paso 2 — Ejecución

```bash
python3 sesion06_demo.py
```

### Parte A — MPC

El script parte de tres cifras ficticias: Banco A **12**, Banco B **18** y Banco C **25**. Reparte cada cifra en tres fragmentos aleatorios módulo un primo y reconstruye únicamente la suma. Busca esta salida:

```text
RESULTADO CONJUNTO RECONSTRUIDO: 55
```

**Explicación de clase:** “No hace falta entregar las tres cifras directamente para obtener su suma. En esta simulación, cada banco envía fragmentos preparados matemáticamente.”

**Límite:** el programa contiene todos los valores en memoria porque simula a todos los bancos. Una solución MPC real distribuye las entradas entre distintas máquinas y emplea un protocolo seguro.

### Parte B — ZKP

El script genera un secreto, calcula un dato público relacionado y ejecuta una prueba **tipo Schnorr simplificada**. Muestra un compromiso, un desafío, una respuesta y una verificación.

```text
¿La prueba verifica?: True
```

**Explicación de clase:** “La verificación puede confirmarse sin pedir que el probador envíe directamente su número secreto.”

**Límite:** el grupo matemático es muy pequeño y un atacante puede recuperar el secreto por fuerza bruta; el ejemplo solo enseña el flujo de un protocolo de conocimiento.

### Parte C — Cadena de hashes

El script crea tres eventos encadenados mediante SHA-256 y después cambia uno de los textos. Busca:

```text
¿Cadena íntegra?: True
¿Cadena íntegra ahora?: False
```

**Explicación de clase:** “Cuando el contenido de un registro cambia, su huella deja de coincidir y la alteración puede detectarse.”

**Límite:** un atacante con control total del archivo local podría reconstruir toda la cadena. Una blockchain real incorpora red de nodos, mecanismos de validación y consenso, entre otras protecciones.

## Paso 3 — Repetir

```bash
python3 sesion06_demo.py
```

Compara los fragmentos de MPC y los valores de la prueba ZKP: algunos cambian gracias a la aleatoriedad. Sin embargo, la suma sigue siendo **55**, la prueba legítima verifica y la modificación de la cadena sigue detectándose.

## Preguntas para la audiencia

- ¿Qué dato no necesitan conocer los otros bancos para calcular el total?
- ¿Qué se demuestra en la prueba sin revelar directamente el secreto?
- ¿Por qué la cadena de hashes local no equivale a una blockchain completa?
- ¿Qué información podría seguir revelándose en el resultado de un cálculo MPC?
- ¿Qué tecnología usaríamos para sumar cifras privadas, demostrar una condición o detectar cambios en un historial?

## Referencias

- [NIST — Privacy-Enhancing Cryptography](https://csrc.nist.gov/projects/pec/pec-tools)
- [ZKProof — Community Reference](https://docs.zkproof.org/reference)
- [NIST IR 8202 — Blockchain Technology Overview](https://csrc.nist.gov/pubs/ir/8202/final)
