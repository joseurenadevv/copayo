# Schema de Notion

Documentación del esquema de las 3 bases de Notion tal como están hoy en producción.
Este documento describe **estructura y datos de referencia**; la fuente de verdad
sigue siendo Notion.

> **Nota importante — las propiedades `Especialidad` son tipo `Select` en las 3 bases**
> (no texto libre). Esto fue una corrección deliberada: los nodos de Notion en n8n
> filtran por `select`, y un filtro por texto libre no matchea de forma fiable. Al
> agregar o editar filas, la opción de `Especialidad` debe elegirse de la lista de
> `select` existente y escribirse exactamente igual en las 3 bases.

Opciones de `Especialidad` (idénticas en las 3 bases):
`Cardiología`, `Medicina General`, `Gastroenterología`, `Traumatología`, `Neumología`.

---

## Base 1 — Síntomas (5 filas)

| Propiedad     | Tipo   | Notas |
|---------------|--------|-------|
| `Síntoma`     | Title  | Nombre del síntoma. |
| `Especialidad`| Select | Una de las 5 opciones. Mapea el síntoma a la especialidad médica que lo atiende. |

Uso en el flujo: n8n busca el síntoma clasificado por Groq y obtiene su `Especialidad`
para consultar la base de Hospitales.

### Datos completos

| Síntoma                  | Especialidad       |
|--------------------------|--------------------|
| dolor de pecho           | Cardiología        |
| fiebre alta              | Medicina General   |
| dolor abdominal          | Gastroenterología  |
| fractura/trauma          | Traumatología      |
| dificultad respiratoria  | Neumología         |

---

## Base 2 — Hospitales (15 filas = 3 hospitales × 5 especialidades)

| Propiedad      | Tipo   | Notas |
|----------------|--------|-------|
| `Hospital`     | Title  | Nombre del hospital. |
| `Especialidad` | Select | Una de las 5 opciones. |
| `Costo Base`   | Number | Costo base de la atención para ese hospital y especialidad (USD). |

Cada fila representa la combinación (hospital, especialidad) con su costo base.
n8n filtra por `Especialidad` (select), compara los `Costo Base` de los 3 hospitales
y elige el más económico. La comparación y el cálculo se hacen en un nodo Code de n8n,
nunca en el LLM.

### Datos completos

| Hospital   | Especialidad       | Costo Base |
|------------|--------------------|-----------:|
| Hospital A | Cardiología        | 150 |
| Hospital A | Medicina General   | 50 |
| Hospital A | Gastroenterología  | 100 |
| Hospital A | Traumatología      | 120 |
| Hospital A | Neumología         | 130 |
| Hospital B | Cardiología        | 130 |
| Hospital B | Medicina General   | 40 |
| Hospital B | Gastroenterología  | 90 |
| Hospital B | Traumatología      | 110 |
| Hospital B | Neumología         | 115 |
| Hospital C | Cardiología        | 200 |
| Hospital C | Medicina General   | 60 |
| Hospital C | Gastroenterología  | 125 |
| Hospital C | Traumatología      | 150 |
| Hospital C | Neumología         | 160 |

### Hospital más económico por especialidad (derivado de los datos de arriba)

| Especialidad      | Gana       | Costo Base |
|-------------------|------------|-----------:|
| Cardiología       | Hospital B | 130 (vs. 150 de Hospital A) |
| Medicina General  | Hospital B | 40 |
| Gastroenterología | Hospital B | 90 |
| Traumatología     | Hospital B | 110 |
| Neumología        | Hospital B | 115 (vs. 130 de Hospital A) |

Con los precios reales, **Hospital B es el más económico en las 5 especialidades**.
Esto es correcto por diseño, no un error de datos.

---

## Base 3 — Planes (10 filas = 2 planes × 5 especialidades)

| Propiedad        | Tipo   | Notas |
|------------------|--------|-------|
| `Regla`          | Title  | Nombre descriptivo de la regla (p. ej. "Básico - Cardiología"). |
| `Plan`           | Select | `Básico` o `Premium`. |
| `Especialidad`   | Select | Una de las 5 opciones. |
| `Tipo de regla`  | Select | `Porcentaje` o `Monto fijo`. |
| `Valor`          | Number | Si `Tipo de regla` = `Porcentaje`: porcentaje del costo base (p. ej. `50` = 50 %). Si `Monto fijo`: monto en USD. |

Uso en el flujo: n8n busca la fila que matchea (`Plan`, `Especialidad`) y aplica la
regla sobre el `Costo Base` del hospital más económico.

### Datos completos

| Regla                          | Plan    | Especialidad       | Tipo de regla | Valor |
|--------------------------------|---------|--------------------|---------------|------:|
| Básico - Medicina General      | Básico  | Medicina General   | Monto fijo    | 20 |
| Básico - Cardiología           | Básico  | Cardiología        | Porcentaje    | 50 |
| Básico - Gastroenterología     | Básico  | Gastroenterología  | Porcentaje    | 50 |
| Básico - Traumatología         | Básico  | Traumatología      | Porcentaje    | 50 |
| Básico - Neumología            | Básico  | Neumología         | Porcentaje    | 50 |
| Premium - Medicina General     | Premium | Medicina General   | Monto fijo    | 0 |
| Premium - Cardiología          | Premium | Cardiología        | Monto fijo    | 30 |
| Premium - Gastroenterología    | Premium | Gastroenterología  | Monto fijo    | 30 |
| Premium - Traumatología        | Premium | Traumatología      | Monto fijo    | 30 |
| Premium - Neumología           | Premium | Neumología         | Monto fijo    | 30 |

Resumen de las reglas:

- **Básico + Medicina General:** monto fijo de $20.
- **Básico + resto de especialidades:** 50 % del costo base.
- **Premium + Medicina General:** monto fijo de $0.
- **Premium + resto de especialidades:** monto fijo de $30.

> Los nombres de la columna `Regla` (title) son descriptivos y pueden diferir en
> Notion; lo que el flujo usa para el match son las propiedades `Plan` y
> `Especialidad` (ambas `select`).
