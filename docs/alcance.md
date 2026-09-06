# Alcance

## Objetivo

Reducir la incertidumbre del paciente sobre cuánto pagará por una atención médica:
recibe un síntoma y un plan de seguro, determina la especialidad, encuentra el
hospital más económico disponible y estima el copago aplicable.

## MVP cerrado

El MVP contempla exactamente:

- **5 síntomas** en la base de Notion, cada uno mapeado a una de 5 especialidades:
  Medicina General, Cardiología, Gastroenterología, Traumatología, Neumología.
- **3 hospitales** ficticios (Hospital A, B, C) con costo base por especialidad
  (15 filas). Datos completos en [notion/schema.md](../notion/schema.md).
- **2 planes** de seguro: Básico y Premium, con reglas de copago por
  (plan, especialidad) (10 filas).
- Datos estructurados en Notion; orquestación en n8n; clasificación de síntomas con
  Groq; **cálculo del copago determinístico en un nodo Code de n8n, nunca en el LLM**.
- Canal: Telegram.

## Fuera de alcance (decisiones de scope, no bugs pendientes)

- **Sin memoria de conversación entre mensajes.** Cada mensaje se procesa desde cero.
  Si el bot pide una aclaración (falta plan o especialidad) y el paciente responde en
  un mensaje nuevo, esa respuesta se procesa como una consulta independiente, sin
  recordar la pregunta anterior. Es una decisión de scope para el MVP, no un bug
  pendiente.
- Sin diagnóstico médico: el sistema solo clasifica dentro de los síntomas
  contemplados.
- Sin síntomas, hospitales ni planes fuera de los listados arriba. Ante entradas
  fuera de ese conjunto, el sistema pide más información o informa que no puede
  procesar el caso; no inventa datos.
- Sin autenticación de pacientes, sin persistencia de historial, sin integración con
  sistemas reales de aseguradoras u hospitales (los datos son ficticios).

## Validación

15 casos de prueba (flujo normal + casos de falta de información) en
[docs/casos-de-prueba.md](casos-de-prueba.md). Con los precios reales, Hospital B es
el más económico en las 5 especialidades (correcto por diseño).
