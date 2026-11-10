# Sesión 7 — Laboratorio guiado: Aprendizaje Federado, FedAvg y privacidad

**Módulo 10:** Protección de Datos y Criptografía Aplicada  
**Tema:** Inteligencia Artificial, Aprendizaje Federado, Secure Aggregation y Differential Privacy  
**Entorno:** Kali Linux o cualquier equipo con Python 3  
**Duración sugerida:** 45–60 minutos  
**Dificultad:** Introductoria, sin experiencia previa en machine learning.

> **Importante:** Los registros son inventados y no corresponden a pacientes reales. El programa simula varios participantes dentro de un único proceso Python. **No** implementa una red federada, una agregación segura criptográfica ni una garantía formal de privacidad diferencial. Sirve para visualizar el principio y conversar sobre sus riesgos.

## 1. ¿Qué aprenderemos?

Imagina que tres hospitales desean construir una regla de predicción mejor que la que obtendría cada uno por separado. Sin embargo, ninguno desea compartir sus registros originales con un servidor externo.

En el ejemplo, **Hospital Norte** tiene 4 pares numéricos ficticios, **Hospital Centro** tiene 5 y **Hospital Sur** tiene 3. Cada hospital calcula un pequeño modelo local a partir de sus observaciones. El agregador recibe parámetros del modelo, no las filas originales, y calcula un promedio ponderado. Esto es una representación sencilla de **Federated Averaging (FedAvg)**.

```text
 Hospital Norte       Hospital Centro        Hospital Sur
 Registros propios    Registros propios      Registros propios
        │                    │                    │
        ▼                    ▼                    ▼
 Entrena modelo A     Entrena modelo B       Entrena modelo C
        └────────────────────┼────────────────────┘
                             │
                   Solo modelos locales
                             ▼
                   SERVIDOR AGREGADOR
                      FedAvg ponderado
                             │
                             ▼
                       Modelo global
                             │
               Se distribuye otra vez a todos
```

**Pregunta para la clase:** ¿Que los datos originales permanezcan en cada hospital significa que nadie puede inferir información sensible? **No necesariamente**. Las actualizaciones también pueden filtrar información.

## 2. Preparación en Kali Linux

Abre una terminal y comprueba Python:

```bash
python3 --version
```

Crea una carpeta:

```bash
mkdir -p ~/sesion07
cd ~/sesion07
```

Descarga el archivo `sesion07_federated_lab.py` adjunto junto a esta guía y cópialo dentro de `~/sesion07`. Si lo guardaste en `~/Descargas`, por ejemplo:

```bash
cp ~/Descargas/sesion07_federated_lab.py ~/sesion07/
```

Si tu navegador lo guarda en otra ruta, cambia `~/Descargas` por la carpeta correspondiente. Verifica:

```bash
ls -lh ~/sesion07/sesion07_federated_lab.py
```

No necesitas Docker, `pip`, bibliotecas de inteligencia artificial ni conexión a Internet para ejecutarlo.

## 3. Ejecutar el entrenamiento federado

Desde la carpeta de trabajo:

```bash
python3 sesion07_federated_lab.py
```

Al comienzo verás que el programa identifica tres hospitales ficticios y comunica únicamente la **cantidad de registros y los parámetros de modelo** en la salida del agregador.

Por ejemplo, durante la primera ronda:

```text
Hospital Norte: 4 registros locales, envía modelo (1.701, 0.591)
Hospital Centro: 5 registros locales, envía modelo (2.113, 0.484)
Hospital Sur: 3 registros locales, envía modelo (1.971, 0.531)
```

**Cómo explicarlo:** Los modelos son pequeñas fórmulas de la forma `predicción = w × x + b`. El símbolo `w` es la pendiente y `b` el intercepto. Se entrenan modificando esos dos números para reducir el error. Aunque el script utiliza observaciones de ejemplo dentro del mismo proceso, estamos simulando que cada hospital calcula sus parámetros localmente.

### ¿Por qué los modelos tienen valores diferentes?

Porque cada institución tiene un conjunto de ejemplos diferente. Un hospital puede tener cuatro registros y otro cinco; cada grupo influye de manera distinta en sus parámetros locales.

## 4. Entender FedAvg

El agregador combina los parámetros según el número de registros de cada institución:

```text
Modelo global =
  (4 × Modelo Norte + 5 × Modelo Centro + 3 × Modelo Sur) / 12
```

Esta operación se realiza **por cada parámetro**, como `w` y `b`.

El programa repite ocho rondas. La salida será semejante a:

```text
Ronda 1: w=1.9403, b=0.5318
Ronda 2: w=2.1015, b=0.5891
Ronda 3: w=2.1120, b=0.6059
...
Ronda 8: w=2.0971, b=0.6669
```

Al finalizar, la predicción de ejemplo cuando `x = 4` es aproximadamente **9.055**.

**Explicación para los alumnos:** La regla comenzó con valores iniciales iguales a cero, y después de varias rondas aprendió una relación entre dos variables numéricas. No entrenamos un modelo médico real: solo demostramos cómo los modelos locales contribuyen a construir uno global.

## 5. Detectar una limitación: el agregador ve actualizaciones

Observa que en la primera ronda aparece el modelo enviado por cada hospital. Eso demuestra un riesgo importante: **no transmitir las filas originales no significa que cada actualización sea confidencial**.

Un adversario con acceso a actualizaciones de determinados modelos podría intentar inferir propiedades de los datos o incluso reconstruir información bajo ciertas condiciones. El riesgo depende del tipo de modelo, número de participantes, cantidad de ejemplos y acceso del atacante.

**Pregunta de reflexión:** ¿Qué controles podríamos añadir si no queremos que el agregador vea los parámetros de cada hospital de manera individual?

## 6. Demostración de máscaras que se cancelan

En la segunda parte, el script utiliza tres números ficticios que representan una sola coordenada de actualización:

```text
Hospital A: 12
Hospital B: 18
Hospital C: 25
TOTAL:      55
```

Cada participante suma una máscara. Las máscaras se eligen para que su suma total sea cero. En la ejecución reproducible verás:

```text
Valores enmascarados enviados: [-1, 18, 38]
Suma del agregador:             55
Suma real:                      55
```

¿Por qué ocurre? Porque:

```text
(12 + máscara_A) + (18 + máscara_B) + (25 + máscara_C)
  = 12 + 18 + 25 + (máscara_A + máscara_B + máscara_C)
  = 55 + 0
  = 55
```

El resultado conjunto no cambia aunque los valores transmitidos sean distintos.

**Advertencia docente:** Este ejemplo NO es un protocolo criptográficamente seguro de Secure Aggregation. En el script, las máscaras se calculan en un único proceso y no se protegen contra un agregador malicioso, colusión o abandono de participantes. Para el despliegue real se requieren protocolos especializados, gestión de claves y una evaluación formal del modelo de amenaza.

## 7. Mostrar qué significa agregar ruido

La última parte imprime:

```text
Total real agregado:                120
Total publicado con ruido:          124
```

**Pregunta para la clase:** ¿Por qué querríamos publicar un número ligeramente diferente al real?

Respuesta: porque, en ciertos sistemas estadísticos, aceptar una pequeña pérdida de precisión puede reducir lo que se infiere sobre una persona. Pero hay una diferencia enorme entre agregar ruido de cualquier forma e implementar **Differential Privacy** correctamente. Una solución DP real debe definir sensibilidad, mecanismo de ruido, límites de contribución y presupuesto (`ε`, y cuando aplique `δ`).

**No presentes este ejemplo como una garantía de privacidad diferencial**: solo ilustra la tensión entre privacidad y precisión.

## 8. Actividad para estudiantes (10–15 minutos)

En parejas, redacten una respuesta breve a cada pregunta:

1. ¿Dónde se conservan los registros originales en el diseño federado?
2. ¿Qué recibe el agregador durante FedAvg en el ejemplo básico?
3. ¿Por qué las actualizaciones pueden representar un riesgo de privacidad?
4. ¿Qué intenta conseguir Secure Aggregation?
5. ¿Por qué agregar ruido arbitrario no equivale a Differential Privacy?
6. ¿Qué sucedería si uno de los hospitales enviara actualizaciones manipuladas?
7. Propongan dos medidas de seguridad adicionales que deberían existir en una solución de producción.

### Respuestas esperadas

- Los registros permanecen en cada institución participante.
- El agregador recibe modelos o actualizaciones y cantidades de ejemplos usadas para ponderar.
- Las actualizaciones pueden contener señales sobre ejemplos locales; hay ataques de inferencia y reconstrucción bajo ciertos supuestos.
- Secure Aggregation intenta permitir el cálculo agregado sin revelar aportes individuales al agregador.
- DP requiere una garantía matemática y parámetros explícitos, no ruido improvisado.
- Un participante malicioso podría degradar el modelo o introducir un comportamiento no deseado (*poisoning* o *backdoor*).
- Controles: autenticación, canales cifrados, protección de clientes, agregación segura, DP calibrada, validación de contribuciones, auditoría y evaluación del modelo.

## 9. Repetir o modificar el laboratorio

Abre el script:

```bash
nano ~/sesion07/sesion07_federated_lab.py
```

Puedes cambiar el número de rondas en:

```python
for rnd in range(1,9):
```

Por ejemplo, para ejecutar 12 rondas:

```python
for rnd in range(1,13):
```

Guarda con `Ctrl + O`, `Enter` y `Ctrl + X`. Vuelve a ejecutar:

```bash
python3 ~/sesion07/sesion07_federated_lab.py
```

Observa cómo cambian los parámetros. El aumento de rondas **no garantiza** que siempre mejore el rendimiento; dependerá de los datos y del entrenamiento.

## 10. Troubleshooting

### `python3: command not found`

En Kali, si fuese necesario:

```bash
sudo apt update
sudo apt install -y python3
```

### `can't open file`

Comprueba la carpeta y el nombre del archivo:

```bash
pwd
ls -lh
```

O usa la ruta completa:

```bash
python3 ~/sesion07/sesion07_federated_lab.py
```

### El script funciona, pero no veo archivos o servidores separados

Es el comportamiento esperado: se trata de una **simulación en un solo proceso**, no de varias máquinas conectadas. Para una implementación más realista con clientes y servidor, se pueden estudiar frameworks como Flower y APPFL, con infraestructura y controles adicionales.

## 11. Conclusión para la clase

> **El aprendizaje federado ayuda a construir un modelo compartido sin reunir necesariamente todos los registros originales, pero las actualizaciones y el modelo final también necesitan protección.**

En términos sencillos:

```text
Aprendizaje federado  → No centralizar necesariamente datos originales
FedAvg                 → Combinar modelos locales
Secure Aggregation     → Proteger aportes individuales del agregador
Differential Privacy   → Limitar inferencias de contribuciones individuales
Gobierno y controles   → Hacer viable y auditable la solución
```

## Referencias

- NIST, *Protecting Model Updates in Privacy-Preserving Federated Learning*: https://www.nist.gov/blogs/cybersecurity-insights/protecting-model-updates-privacy-preserving-federated-learning
- Flower, *Federated Learning with Flower*: https://flower.ai/docs/framework/tutorial-series-get-started-with-flower-pytorch.html
- Google Research, *Practical Secure Aggregation for Federated Learning on User-Held Data*: https://research.google/pubs/practical-secure-aggregation-for-federated-learning-on-user-held-data/
- Google Research, *Toward provably private learning from federated data* (2026): https://research.google/blog/toward-provably-private-learning-from-federated-data/
