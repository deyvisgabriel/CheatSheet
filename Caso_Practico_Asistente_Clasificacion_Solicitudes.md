# Caso práctico — Asistente de Clasificación de Solicitudes

## Sesión 2 — APIs y Automatización Inteligente

### Objetivo

Comprender de forma práctica cómo una solución de Inteligencia Artificial puede:

- Recibir una solicitud.
- Interpretar su contenido.
- Clasificarla.
- Asignar una prioridad.
- Generar una respuesta.
- Preparar información que posteriormente podría enviarse a una API.

> **Importante:** Para desarrollar este caso no es necesario instalar ninguna herramienta adicional. Puede resolverse utilizando ChatGPT, un navegador y un documento de Word, Google Docs o similar.

---

# 1. Escenario

Una empresa recibe diariamente solicitudes de clientes y usuarios mediante correo electrónico y WhatsApp.

Actualmente, una persona debe leer cada mensaje, identificar de qué se trata y derivarlo manualmente al área correspondiente.

La empresa desea analizar cómo podría utilizar Inteligencia Artificial para apoyar este proceso.

Su equipo deberá diseñar un pequeño **Asistente de Clasificación de Solicitudes**.

---

# 2. Solicitudes recibidas

Analice las siguientes cinco solicitudes:

### Solicitud 1

> “No puedo ingresar al sistema desde esta mañana.”

### Solicitud 2

> “Mi factura llegó con un monto incorrecto.”

### Solicitud 3

> “Necesito información sobre los planes comerciales.”

### Solicitud 4

> “Quiero cancelar mi servicio.”

### Solicitud 5

> “El sistema está muy lento y no puedo trabajar.”

---

# 3. Paso 1 — Clasificar las solicitudes

Cada solicitud debe ser asignada a una de las siguientes categorías:

- Accesos
- Facturación
- Comercial
- Cancelación
- Soporte técnico

Complete una tabla similar a la siguiente:

| Nº | Solicitud | Categoría |
|---|---|---|
| 1 | No puedo ingresar al sistema desde esta mañana. | |
| 2 | Mi factura llegó con un monto incorrecto. | |
| 3 | Necesito información sobre los planes comerciales. | |
| 4 | Quiero cancelar mi servicio. | |
| 5 | El sistema está muy lento y no puedo trabajar. | |

---

# 4. Paso 2 — Asignar una prioridad

Para cada solicitud, asigne una prioridad:

- Baja
- Media
- Alta

Utilice su criterio considerando el impacto que podría tener el problema para el usuario.

Complete la tabla:

| Nº | Categoría | Prioridad |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |
| 4 | | |
| 5 | | |

---

# 5. Paso 3 — Crear un prompt

Ahora deberá construir un prompt para que una Inteligencia Artificial pueda realizar automáticamente la clasificación.

El prompt debe indicar claramente:

- El rol que tendrá la IA.
- Qué debe analizar.
- Qué categorías puede utilizar.
- Qué prioridades puede asignar.
- Cómo debe presentar la respuesta.

### Estructura recomendada

```text
Rol + Tarea + Contexto + Reglas + Formato de salida
```

### Ejemplo de inicio

```text
Eres un asistente de mesa de ayuda.
Analiza la siguiente solicitud...
```

Complete su propio prompt.

---

# 6. Paso 4 — Probar el prompt en ChatGPT

Copie su prompt en ChatGPT junto con las cinco solicitudes.

Compare la respuesta obtenida con la clasificación que realizó manualmente.

Responda:

### Pregunta 1

¿La Inteligencia Artificial clasificó las solicitudes de la misma manera que su equipo?

```text
Respuesta:
```

### Pregunta 2

¿Qué diferencias encontraron?

```text
Respuesta:
```

### Pregunta 3

¿Qué cambiarían en el prompt para obtener un mejor resultado?

```text
Respuesta:
```

---

# 7. Paso 5 — Convertir una respuesta a JSON

Las APIs suelen intercambiar información utilizando formatos estructurados como JSON.

Seleccione una de las cinco solicitudes y represente el resultado utilizando esta estructura:

```json
{
  "categoria": "",
  "prioridad": "",
  "descripcion": ""
}
```

Complete los valores correspondientes.

### Ejemplo de estructura

```json
{
  "categoria": "Accesos",
  "prioridad": "Alta",
  "descripcion": "Usuario no puede ingresar al sistema"
}
```

---

# 8. Paso 6 — Diseñar el flujo del asistente

Dibuje un flujo que represente cómo podría funcionar el asistente.

Puede utilizar este modelo como referencia:

```text
Solicitud del usuario
        ↓
IA analiza
        ↓
Clasifica
        ↓
Asigna prioridad
        ↓
Genera respuesta
        ↓
Deriva al responsable
```

Puede dibujarlo:

- En Word.
- En PowerPoint.
- En Google Docs.
- En papel y tomar una fotografía.
- Utilizando cualquier herramienta de diagramación disponible.

---

# 9. Paso 7 — ¿Dónde interviene una API?

Imagine ahora que este asistente está conectado a los sistemas reales de una empresa.

Responda:

### Pregunta 1

¿Qué podría hacer una API dentro de este proceso?

```text
Respuesta:
```

### Pregunta 2

¿En qué momento del flujo utilizaría una API?

```text
Respuesta:
```

### Pregunta 3

Mencione al menos tres acciones que una API podría realizar.

```text
1.
2.
3.
```

---

# 10. Paso 8 — Generar una respuesta para el usuario

Seleccione una de las solicitudes.

Prepare una respuesta corta que el asistente podría enviar al usuario.

### Solicitud seleccionada

```text
```

### Respuesta propuesta

```text
```

La respuesta debe ser:

- Clara.
- Breve.
- Profesional.
- Coherente con la categoría y prioridad asignada.

---

# 11. Pregunta final de análisis

Explique con sus propias palabras la diferencia entre:

```text
Inteligencia Artificial
```

y

```text
API
```

### Respuesta

```text
```

---

# 12. Entregable

Cada equipo deberá entregar **un único documento** que contenga:

1. Clasificación de las cinco solicitudes.
2. Prioridad asignada a cada solicitud.
3. Prompt utilizado.
4. Comparación entre su respuesta y la respuesta obtenida con IA.
5. Ejemplo en formato JSON.
6. Diagrama del flujo del asistente.
7. Respuestas sobre el uso de una API.
8. Respuesta generada para un usuario.
9. Diferencia entre Inteligencia Artificial y API.
10. Conclusión breve del equipo.

---

# 13. Conclusión del equipo

En máximo cinco líneas, responda:

> ¿Cómo podría este tipo de solución ayudar a una empresa?

```text
Conclusión:
```

---

# 14. Criterios de evaluación

| Criterio | Peso |
|---|---:|
| Clasificación de solicitudes | 25% |
| Asignación de prioridades | 20% |
| Calidad del prompt | 20% |
| Comprensión de JSON | 15% |
| Comprensión del rol de una API | 20% |
| **Total** | **100%** |

---

# 15. Tiempo sugerido

Tiempo estimado para resolver el caso:

```text
30 a 40 minutos
```

---

## Idea principal

Al finalizar el ejercicio debe quedar claro que:

```text
IA = comprende, interpreta, clasifica y genera respuestas.

API = permite consultar información o ejecutar acciones en otros sistemas.
```

Una solución empresarial puede combinar ambas capacidades para construir procesos inteligentes y automatizados.
