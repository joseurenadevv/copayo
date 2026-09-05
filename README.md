## Problema

Los usuarios de seguros de salud pueden tener dificultades para estimar cuánto deberán pagar por una atención médica antes de acudir a un hospital. El costo puede depender de la especialidad médica, del hospital disponible y de las condiciones del plan de seguro.

El MVP busca reducir esta incertidumbre mediante un sistema que reciba un síntoma y un plan de seguro, determine la especialidad correspondiente, encuentre el hospital más económico disponible y estime el copago aplicable.

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

El MVP no pretende realizar diagnósticos médicos. Su función es clasificar entradas dentro del conjunto de casos contemplados y realizar una estimación de copago basada en los datos configurados.

Cuando un síntoma sea ambiguo o esté fuera del alcance del MVP, el sistema debe solicitar información adicional o informar que no puede procesar el caso, en lugar de inventar una especialidad, hospital o costo.
