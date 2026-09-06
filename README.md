## Problema

Los usuarios de seguros de salud pueden tener dificultades para estimar cuánto deberán pagar por una atención médica antes de acudir a un hospital. El costo puede depender de la especialidad médica, del hospital disponible y de las condiciones del plan de seguro.

El MVP busca reducir esta incertidumbre mediante un sistema que reciba un síntoma y un plan de seguro, determine la especialidad correspondiente, encuentre el hospital más económico disponible y estime el copago aplicable.

## Solución

Copayo es un agente conversacional en Telegram. El paciente escribe su síntoma y su
plan de seguro; el sistema clasifica el síntoma en una especialidad, busca en Notion
el costo base de cada hospital para esa especialidad, elige el más económico y aplica
la regla de copago del plan. El cálculo del monto se hace de forma determinística en
un nodo Code de n8n (nunca en el LLM); Groq solo redacta la respuesta final en
lenguaje natural.

Detalle en [docs/arquitectura.md](docs/arquitectura.md) y [docs/alcance.md](docs/alcance.md).

## Alcance del MVP

El MVP contempla:

- 5 síntomas mapeados a 5 especialidades médicas:
  - dolor de pecho → Cardiología
  - fiebre alta → Medicina General
  - dolor abdominal → Gastroenterología
  - fractura/trauma → Traumatología
  - dificultad respiratoria → Neumología
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

Flujo de 5 pasos: Telegram → n8n (clasifica y valida) → Notion (lookups por `select`) → cálculo determinístico en JS (elige hospital más económico y aplica la regla del plan) → Groq redacta la respuesta. El dinero **nunca** se calcula en el LLM. Detalle en [docs/arquitectura.md](docs/arquitectura.md).

Con los costos base reales, Hospital B es el más económico en las 5 especialidades (correcto por diseño; ver [notion/schema.md](notion/schema.md)).

## Casos de prueba

Detalle completo de las reglas de copago por plan/especialidad y los 15 casos de prueba (incluyendo los casos de síntoma o plan ambiguos) en [docs/casos-de-prueba.md](docs/casos-de-prueba.md).

## Cómo correrlo

1. Crear las 3 bases de Notion (Síntomas, Hospitales, Planes) con el esquema y los
   datos de [notion/schema.md](notion/schema.md). Las propiedades `Especialidad` deben
   ser tipo `select` en las 3 bases.
2. Crear un bot de Telegram con @BotFather y obtener el token.
3. Obtener una API key de Groq (modelo `openai/gpt-oss-120b`) y un token de
   integración de Notion con acceso a las 3 bases.
4. En n8n, importar `n8n/workflow-copago.json` y configurar las credenciales de
   Telegram, Notion y Groq.
5. Activar el workflow y escribirle al bot en Telegram.

Ver `.env.example` para las variables requeridas.
