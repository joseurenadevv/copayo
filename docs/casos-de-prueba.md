# Casos de prueba
Estos casos verifican la clasificación del síntoma, la selección del hospital más económico y el cálculo determinista del copago.

## Reglas utilizadas

### Plan Básico

- Medicina General: monto fijo de $20.
- Cardiología: 50% del costo base.
- Gastroenterología: 50% del costo base.
- Traumatología: 50% del costo base.
- Neumología: 50% del costo base.

### Plan Premium

- Medicina General: monto fijo de $0.
- Cardiología: monto fijo de $30.
- Gastroenterología: monto fijo de $30.
- Traumatología: monto fijo de $30.
- Neumología: monto fijo de $30.

## Casos de prueba

| # | Mensaje de entrada | Resultado esperado |
|---|---|---|
| 1 | “Tengo fiebre alta y tengo plan Básico” | Medicina General → Hospital B → $20 |
| 2 | “Tengo fiebre alta, mi plan es Premium” | Medicina General → Hospital B → $0 |
| 3 | “Tengo dolor abdominal y plan Básico” | Gastroenterología → Hospital B → $45 |
| 4 | “Me duele el abdomen, tengo plan Premium” | Gastroenterología → Hospital B → $30 |
| 5 | “Me duele el pecho y tengo plan Básico” | Cardiología → Hospital A → $75 |
| 6 | “Tengo dolor de pecho, mi plan es Premium” | Cardiología → Hospital A → $30 |
| 7 | “Tuve una fractura fuerte y tengo plan Básico” | Traumatología → Hospital B → $55 |
| 8 | “Me caí y el hueso se ve raro, tengo plan Premium” | Traumatología → Hospital B → $30 |
| 9 | “Tengo dificultad para respirar y plan Básico” | Neumología → Hospital A → $65 |
| 10 | “Me cuesta respirar, tengo plan Premium” | Neumología → Hospital A → $30 |
| 11 | “Tengo punzadas fuertes en el corazón y plan Básico” | Cardiología → Hospital A → $75 |
| 12 | “Me caí muy fuerte y creo que me fracturé, tengo plan Premium” | Traumatología → Hospital B → $30 |
| 13 | “Me duele el pecho” | **Falta plan → pedir aclaración; no consultar hospitales ni calcular** |
| 14 | “Me siento mal y muy cansado, tengo plan Básico” | **Especialidad no determinada → pedir más información; no consultar hospitales ni calcular** |
| 15 | “Me siento mal” | **Faltan plan y especialidad → pedir ambos datos; no consultar hospitales ni calcular** |

## Casos ambiguos y falta de información

### Caso 13 — Falta el plan

**Entrada:** “Me duele el pecho.”

**Resultado esperado:**

El sistema debe solicitar el plan de seguro antes de continuar.

Mientras no exista un plan válido:

- No se calcula el copago.
- No se debe completar el resultado como si el plan fuera conocido.

### Caso 14 — Especialidad no determinada

**Entrada:** “Me siento mal y muy cansado, tengo plan Básico.”

**Resultado esperado:**

El sistema no debe asignar automáticamente una especialidad. Groq debe solicitar información adicional para poder clasificar el caso.

Mientras no exista una especialidad válida:

- No se consulta la base de hospitales.
- No se selecciona un hospital.
- No se calcula el copago.

### Caso 15 — Faltan plan y especialidad

**Entrada:** “Me siento mal.”

**Resultado esperado:**

El sistema debe solicitar información adicional sobre el problema o síntoma y el plan de seguro.

Mientras no existan ambos datos:

- No se consulta la base de hospitales.
- No se selecciona un hospital.
- No se calcula el copago.

## Objetivo de las pruebas

Los primeros casos verifican el flujo normal:

`Mensaje del usuario → Síntoma → Especialidad → Hospital más económico → Regla del plan → Copago`

El mensaje del usuario contiene el síntoma y, cuando corresponde, el plan de seguro.

Los casos de falta de información verifican que el sistema solicite aclaraciones cuando no pueda determinar la especialidad o el plan.

El cálculo del copago debe realizarse de forma determinista en n8n utilizando los valores almacenados en Notion.

El MVP no debe inventar una especialidad, hospital, plan o costo cuando la información proporcionada por el usuario no sea suficiente.
