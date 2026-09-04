---
name: copayo-context
description: Contexto y reglas fijas del proyecto Copayo — úsala siempre 
que trabajes en este repo.
---

# Copayo — Estimador Agéntico de Copago y Cobertura

Agente conversacional (Telegram) que recibe un síntoma del paciente y 
responde con el copago exacto y el hospital más económico de su red.

## Stack
- n8n (VPS propio) — orquestador
- Notion API — base de datos (síntomas, hospitales, planes)
- Groq (modelo: openai/gpt-oss-120b) — solo redacta, no calcula
- Telegram Bot API — canal del usuario

## Regla de arquitectura no negociable
El cálculo del copago (costo_base × regla_del_plan) y la comparación entre 
los 3 hospitales para encontrar el más económico SIEMPRE se hace en un nodo 
Function/Code de n8n con JavaScript puro. NUNCA se le pide al LLM (Groq) 
que calcule o compare — un LLM no es confiable para aritmética con dinero 
real. Groq solo recibe los datos ya calculados y los redacta en lenguaje 
natural.

## Flujo
Telegram Trigger → Notion (lookup) → Function/Code (cálculo determinístico) 
→ HTTP Request a Groq (redacción) → Telegram (respuesta)

## Estructura del repo
/docs      — arquitectura, alcance, casos de prueba (transversal)
/n8n       — export del workflow.json (mío)
/prompts   — system prompt de Groq (compañero 2)
/notion    — documentación del esquema, NO datos reales (compañero 3)

## Deadline
Domingo 6 de septiembre de 2026. Entrega: link del bot + link del repo a 
hackiathon@viamatica.com
