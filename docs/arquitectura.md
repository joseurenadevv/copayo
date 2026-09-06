# Arquitectura

Copayo es un agente conversacional (vía Telegram) que estima el copago de un plan de
salud y recomienda el hospital más económico de la red para un síntoma dado.

Stack: **n8n** (orquestación, VPS self-hosted) + **Notion API** (datos) + **Groq**
(LLM, `openai/gpt-oss-120b`) + **Telegram Bot API** (canal).

## Regla no negociable

El dinero **nunca** se calcula en el LLM. El cálculo del copago
(`costo_base × regla_del_plan`) y la comparación entre los 3 hospitales para encontrar
el más económico se hacen en un nodo Code (JavaScript) de n8n. Groq solo recibe datos
ya calculados y los redacta en lenguaje natural; nunca hace aritmética ni compara
precios. Un LLM no es confiable para aritmética con dinero real.

## Flujo de 5 pasos

```
1. Telegram        Trigger: llega el mensaje del paciente (síntoma + plan).
        │
        ▼
2. n8n             Clasifica el síntoma por un diccionario de palabras clave en
                   JavaScript (nodo "Function - Detectar Especialidad por
                   Palabras Clave") — determinístico, SIN ningún LLM
                   involucrado en la decisión de especialidad. Groq no
                   interviene en este paso; entra recién en el paso 5, ya con
                   la especialidad y el plan resueltos, solo para redactar el
                   texto. Este paso también valida que haya plan y
                   especialidad. Si falta algo, responde pidiendo aclaración y
                   termina (sin consultar hospitales ni calcular).
        │
        ▼
3. Notion          Lookups (filtrando por Especialidad tipo select):
                   - Síntomas    → especialidad del síntoma
                   - Hospitales  → costo base de cada hospital para esa especialidad
                   - Planes      → regla (Porcentaje / Monto fijo + Valor) para (Plan, Especialidad)
        │
        ▼
4. Cálculo         Nodo Code (JS) determinístico:
   determinístico  - elige el hospital con menor costo base
   en JS           - aplica la regla del plan: Monto fijo → Valor;
                     Porcentaje → costo_base × Valor / 100
                   - produce { especialidad, hospital, costo_base, copago }
        │
        ▼
5. Groq redacta    HTTP Request a Groq con los datos ya calculados. El LLM solo
                   redacta la respuesta en español natural. Se envía por Telegram.
```

## Límites

- El MVP no hace diagnóstico médico: solo clasifica dentro del conjunto de síntomas
  contemplados y estima un copago con los datos configurados.
- Ante un síntoma ambiguo o fuera de alcance, el sistema pide más información o informa
  que no puede procesar el caso; nunca inventa especialidad, hospital o costo.
- Sin memoria de conversación entre mensajes: cada mensaje se procesa desde cero (ver
  [alcance.md](alcance.md)).

## Datos

Estructura y contenido de las 3 bases de Notion (Síntomas, Hospitales, Planes) en
[notion/schema.md](../notion/schema.md). Las propiedades `Especialidad` son tipo
`select` en las 3 bases porque los nodos de Notion en n8n filtran por select.
