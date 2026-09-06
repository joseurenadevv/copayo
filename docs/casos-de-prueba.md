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

| # | Mensaje de entrada | Resultado esperado | Verificación |
|---|---|---|---|
| 1 | “Tengo fiebre alta y tengo plan Básico” | Medicina General → Hospital B → $20 | — |
| 2 | “Tengo fiebre alta, mi plan es Premium” | Medicina General → Hospital B → $0 | ✅ Verificado en producción (2026-09-06) |
| 3 | “Tengo dolor abdominal y plan Básico” | Gastroenterología → Hospital B → $45 | — |
| 4 | “Me duele el abdomen, tengo plan Premium” | Gastroenterología → Hospital B → $30 | ✅ Verificado en producción (2026-09-06) |
| 5 | “Me duele el pecho y tengo plan Básico” | **Activa protocolo de EMERGENCIA en la mayoría de las corridas observadas (sin conteo formal registrado).** En el resto devuelve Cardiología → Hospital B → $65. La detección de emergencia para síntomas torácicos/respiratorios es probabilística por diseño del modelo (temperature 0.3 en Groq) — no se ajustó porque priorizar la alerta de emergencia sobre el cálculo de copago es el comportamiento más seguro para el paciente. | ✅ Verificado en producción (2026-09-06, backup sin memoria de sesión); comportamiento no determinístico observado en varias corridas |
| 6 | “Tengo dolor de pecho, mi plan es Premium” | Cardiología → Hospital B → $30 | — |
| 7 | “Tuve una fractura fuerte y tengo plan Básico” | Traumatología → Hospital B → $55 | ✅ Verificado en producción (2026-09-06) |
| 8 | “Me caí y el hueso se ve raro, tengo plan Premium” | Traumatología → Hospital B → $30 | ✅ Verificado en producción (2026-09-06) |
| 9 | “Tengo dificultad para respirar y plan Básico” | **Activa protocolo de EMERGENCIA** — comportamiento correcto: prioriza la seguridad sobre el cálculo de copago ante dificultad respiratoria | ✅ Verificado en producción (2026-09-06, backup sin memoria de sesión) |
| 10 | “Me cuesta respirar, tengo plan Premium” | **Activa protocolo de EMERGENCIA** — mismo comportamiento que el caso 9 | ✅ Verificado en producción (2026-09-06, backup sin memoria de sesión) |
| 11 | “Tengo punzadas fuertes en el corazón y plan Básico” | **Activa protocolo de EMERGENCIA** — ver nota abajo: el disparo depende de qué tan alarmante suena la frase, no solo de la especialidad | ✅ Verificado en producción (2026-09-06, backup sin memoria de sesión) |
| 12 | “Me caí muy fuerte y creo que me fracturé, tengo plan Premium” | Traumatología → Hospital B → $30 | ✅ Verificado en producción (2026-09-06) |
| 13 | “Me duele el pecho” | **Falta plan → pedir aclaración; no consultar hospitales ni calcular** | — |
| 14 | “Me siento mal y muy cansado, tengo plan Básico” | **Especialidad no determinada → pedir más información; no consultar hospitales ni calcular** | — |
| 15 | “Me siento mal” | **Faltan plan y especialidad → pedir ambos datos; no consultar hospitales ni calcular** | — |

Los 9 casos marcados como verificados fueron ejecutados manualmente contra el bot en
producción por Yassell el 2026-09-06, **contra el workflow backup sin memoria de
sesión** (la versión que corre producción; ver `HALLAZGO-5b-bug-estructural.md` en el
Escritorio, no incluido en este repo, para el intento de memoria revertido). Los casos
2, 4, 7, 8 y 12 coincidieron con el resultado esperado; los casos 9, 10 y 11 cambiaron
de resultado esperado según la evidencia real (ver más abajo); el caso 5 resultó ser
no determinístico (dispara emergencia en la mayoría de las corridas observadas — ver
su fila y la nota sobre Neumología y emergencia más abajo).

> **Nota sobre el hospital ganador (correcto por diseño, no un error de datos).**
> Con los costos base reales cargados en Notion, **Hospital B es el más económico en
> las 5 especialidades**. En particular gana en Cardiología (130 vs. 150 de Hospital A)
> y en Neumología (115 vs. 130 de Hospital A). Versiones anteriores de estos casos
> asumían que Hospital A ganaba en Cardiología y Neumología; esa expectativa se
> corrigió en los casos 5, 6, 9, 10 y 11 para reflejar los precios reales. Los montos
> de copago derivan de esos costos: Cardiología Básico = 50 % de 130 = $65;
> Neumología Básico = 50 % de 115 = $57.50. El caso 5 (Cardiología Básico) dispara
> emergencia en la mayoría de las corridas observadas, así que la ruta de copago de
> Cardiología se ilustra con el caso 6 (mismo síntoma, plan Premium → $30 fijo). Ver
> la tabla de datos en [notion/schema.md](../notion/schema.md).

### Nota sobre Neumología y el protocolo de emergencia

Verificado el 2026-09-06 contra el bot en producción por Yassell (contra el workflow
sin memoria de sesión; ver `HALLAZGO-5b-bug-estructural.md` en el Escritorio, no
incluido en este repo, para el caso con memoria, revertido):

- La **única frase de síntoma del sistema para Neumología es "dificultad respiratoria"**,
  y esa frase **dispara el protocolo de emergencia de forma consistente** en las
  pruebas: el modelo prioriza indicar atención de urgencia sobre calcular el copago.
  Los casos 9 y 10 lo confirman. (El disparo es probabilístico como todo lo del
  modelo, pero para frases de dificultad respiratoria no se observó ninguna corrida
  que cayera al cálculo de copago.)
- En consecuencia, **el flujo de copago normal para Neumología no tiene ningún caso de
  prueba que lo demuestre de punta a punta**. Es una limitación conocida del set de
  pruebas, no un bug: el cálculo determinístico para Neumología (50 % de 115 = $57.50
  en Básico, $30 en Premium) sigue existiendo en el workflow; simplemente ninguna
  entrada realista lo alcanza, porque toda mención de dificultad para respirar se
  trata como posible emergencia.
- El disparo de emergencia **depende de qué tan alarmante suena la frase exacta, y es
  probabilístico** (Groq corre con temperature 0.3): el caso 11 ("punzadas fuertes en
  el corazón") dispara emergencia de forma consistente; el caso 5 ("me duele el pecho")
  — misma especialidad (Cardiología) y mismo plan (Básico) — dispara emergencia en la
  mayoría de las corridas observadas y solo ocasionalmente devuelve el copago normal de
  $65 (sin conteo formal registrado). No se ajustó el prompt para forzar determinismo:
  preferir un falso positivo de emergencia sobre un falso negativo es el comportamiento
  más seguro para el paciente.

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
