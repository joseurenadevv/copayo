# 🏥 Copayo

> Agente conversacional en Telegram que estima el copago de un plan de salud y recomienda el hospital más económico de la red para un síntoma dado.

![estado](https://img.shields.io/badge/estado-en%20producci%C3%B3n-success)
![MVP](https://img.shields.io/badge/MVP-activo-blue)
![stack](https://img.shields.io/badge/stack-n8n%20%2B%20Notion%20%2B%20Groq%20%2B%20Telegram-lightgrey)

---

## 🎯 Objetivo del Proyecto

Los usuarios de seguros de salud tienen dificultades para estimar cuánto pagarán por una atención médica antes de acudir a un hospital: el costo depende de la especialidad, del hospital disponible y de las condiciones del plan. Copayo reduce esa incertidumbre:

1. **Clasificar el síntoma en una especialidad** a partir del mensaje del paciente, sin inventar datos cuando la información no alcanza.
2. **Encontrar el hospital más económico de la red** para esa especialidad, comparando los costos base de los 3 hospitales.
3. **Estimar el copago exacto** aplicando la regla del plan (Básico / Premium) de forma determinística en n8n — el dinero **nunca** se calcula en el LLM.

---

## 🛠️ Tecnologías del Proyecto

| Tecnología | Descripción |
|---|---|
| **n8n** | Orquestación del workflow (VPS self-hosted). Recibe el mensaje, hace los lookups en Notion, ejecuta el cálculo determinístico en un nodo Code (JavaScript) y llama a Groq. |
| **Notion API** | Almacén de datos estructurados: 3 bases (Síntomas, Hospitales, Planes). La propiedad `Especialidad` es tipo `select` en las 3 porque los nodos de Notion en n8n filtran por select. |
| **Groq — `openai/gpt-oss-120b`** | LLM. Solo recibe datos ya calculados y redacta la respuesta en español natural; nunca hace aritmética ni compara precios. |
| **Telegram Bot API** | Canal de conversación con el paciente. |

Flujo de 5 pasos: **Telegram → n8n (clasifica y valida) → Notion (lookups por `select`) → cálculo determinístico en JS (elige hospital más económico y aplica la regla del plan) → Groq redacta la respuesta**. Detalle en [docs/arquitectura.md](docs/arquitectura.md).

---

## 📦 Alcance del MVP

El MVP contempla:

- **5 síntomas** mapeados a 5 especialidades médicas:
  - dolor de pecho → Cardiología
  - fiebre alta → Medicina General
  - dolor abdominal → Gastroenterología
  - fractura/trauma → Traumatología
  - dificultad respiratoria → Neumología
- **3 hospitales** ficticios (Hospital A, B, C) con distinto costo base por especialidad.
- **2 planes** de seguro: Básico y Premium, con reglas de copago por combinación de plan y especialidad.
- Datos estructurados en Notion, orquestación en n8n, clasificación de síntomas con un modelo de lenguaje y cálculo determinista del copago.

Con los costos base reales, **Hospital B es el más económico en las 5 especialidades** (correcto por diseño; ver [notion/schema.md](notion/schema.md)).

### Limitaciones conocidas del MVP

- Sin sesión ni memoria de conversación: cada mensaje se procesa desde cero. Si el bot pide una aclaración (falta especialidad o plan) y el paciente responde en un mensaje nuevo, esa respuesta se procesa como una consulta independiente, sin recordar la pregunta anterior. Es una decisión de scope, no un bug pendiente.
- No hace diagnóstico médico: solo clasifica dentro del conjunto de casos contemplados.
- Ante un síntoma ambiguo o fuera de alcance, el sistema pide más información o informa que no puede procesar el caso, en lugar de inventar una especialidad, hospital o costo.

Detalle completo en [docs/alcance.md](docs/alcance.md).

---

## 🧪 Casos de prueba

Reglas de copago por plan/especialidad y los 15 casos de prueba (flujo normal + casos de síntoma o plan ambiguo) en [docs/casos-de-prueba.md](docs/casos-de-prueba.md).

---

## 💻 Guía de Comandos

### 1. Clonar el repositorio

```bash
# Traer el código y entrar a la carpeta
git clone https://github.com/joseurenadevv/copayo.git
cd copayo
```

### 2. Configurar las variables de entorno

```bash
# Copiar la plantilla y completar las 3 claves
cp .env.example .env

# .env debe quedar con:
#   NOTION_API_KEY=...       token de integración de Notion con acceso a las 3 bases
#   GROQ_API_KEY=...         API key de console.groq.com (modelo openai/gpt-oss-120b)
#   TELEGRAM_BOT_TOKEN=...   token del bot creado con @BotFather
```

### 3. Crear las 3 bases de Notion

```bash
# Crear en Notion: Síntomas, Hospitales y Planes
# con el esquema y los datos de notion/schema.md.
# La propiedad "Especialidad" debe ser tipo SELECT en las 3 bases.
```

### 4. Crear el bot de Telegram

```bash
# En Telegram, hablar con @BotFather:
#   /newbot  ->  seguir los pasos  ->  copiar el token a TELEGRAM_BOT_TOKEN
```

### 5. Importar y activar el workflow en n8n

```bash
# En n8n: Workflows -> Import from File -> n8n/workflow-copago.json
# Configurar las credenciales de Telegram, Notion y Groq en sus nodos.
# Activar el workflow y escribirle al bot en Telegram.
```

---

## 👤 Equipo

| Integrante | Rol |
|---|---|
| **Jose** | n8n / integración — workflow, lookups de Notion, nodo de cálculo, conexión con Groq y Telegram. |
| **Euribiades** | Prompts / demo — system prompts de Groq (Redactar Respuesta y Pedir Aclaración) y guía de pruebas. |
| **Yassell** | Datos / pruebas — esquema y datos de las bases de Notion, casos de prueba. |
