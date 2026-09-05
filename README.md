## Problema

Los usuarios de seguros de salud pueden tener dificultades para estimar cuánto deberán pagar por una atención médica antes de acudir a un hospital. El costo puede depender de la especialidad médica, del hospital disponible y de las condiciones del plan de seguro.

El MVP busca reducir esta incertidumbre mediante un sistema que reciba un síntoma y un plan de seguro, determine la especialidad correspondiente, encuentre el hospital más económico disponible y estime el copago aplicable.

## Solución

**Pendiente** — descripción de la solución (agente conversacional en Telegram, arquitectura n8n + Notion + Groq).

## Alcance del MVP

El MVP contempla:

- 5 especialidades médicas:
  - Medicina General
  - Cardiología
  - Gastroenterología
  - Traumatología
  - Neumología
- 3 hospitales ficticios con diferentes costos por especialidad.
- 2 planes de seguro: Básico y Premium.
- Reglas de copago específicas para cada combinación de plan y especialidad.
- Datos estructurados almacenados en Notion.
- Automatización y procesamiento mediante n8n.
- Clasificación inicial de síntomas mediante un modelo de lenguaje.
- Cálculo determinista del copago mediante las reglas almacenadas en Notion.

### Limitaciones conocidas del MVP
- El workflow no maneja sesión ni memoria de conversación: cada mensaje se procesa desde cero. Si el bot pide una aclaración (falta especialidad o plan) y el paciente responde en un mensaje nuevo, esa respuesta se procesa como una consulta independiente, sin recordar la pregunta anterior.

## Arquitectura

El MVP no pretende realizar diagnósticos médicos. Su función es clasificar entradas dentro del conjunto de casos contemplados y realizar una estimación de copago basada en los datos configurados.

Cuando un síntoma sea ambiguo o esté fuera del alcance del MVP, el sistema debe solicitar información adicional o informar que no puede procesar el caso, en lugar de inventar una especialidad, hospital o costo.

## Casos de prueba

Detalle completo de las reglas de copago por plan/especialidad y los 15 casos de prueba (incluyendo los casos de síntoma o plan ambiguos) en [docs/casos-de-prueba.md](docs/casos-de-prueba.md).

## Cómo correrlo

**Pendiente** — pasos para levantar el bot (importar `n8n/workflow-copago.json`, configurar credenciales de Telegram/Notion/Groq, poblar las bases de Notion).
