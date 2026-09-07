# Hallazgo del 2026-09-06: bug estructural en el gate de emergencia con memoria de sesión

> Post-mortem incluido para transparencia. El 2026-09-06 se intentó agregar memoria de
> conversación de corto plazo al workflow; se detectó un bug estructural en el flujo de
> emergencia durante las pruebas y se revirtió. **Lo que se entrega es el backup sin
> memoria**, verificado contra el bot en producción.

**Estado:** memoria REVERTIDA. Producción volvió al backup `Copayo - BACKUP pre-memoria`
(27 nodos, sin memoria). El workflow con memoria (28 nodos) quedó **desactivado**, no
borrado. El envío del hackIAthon es el backup — verificado (casos base contra el bot en
producción).

**Qué sí funcionó** (checkpoints con dato crudo de ejecución): lectura + fusión de
memoria, TTL 5 min deslizante, barrido de expirados, regla de aporte mínimo, tope de un
solo nivel (el síntoma guardado se ancla al primero, no crece), y el recableo del paso
de clasificación (0 referencias viejas en `Calcular Copago`). El cálculo determinístico
del copago dio 6/6 correcto en todos los checkpoints.

**Qué falló:** el checkpoint 5b — el gate de emergencia.

---

## El bug en una frase

Con memoria activa, un turno de seguimiento que **solo aporta el plan y ningún síntoma**
(ej. `premium` después de `me cuesta respirar`) hace que el
`IF - ¿Especialidad Y Plan Encontrados?` tome la rama de **CÁLCULO**, se compute un
`copagoFinal`, y lo único que impide que el bot muestre un precio en plena emergencia es
que Groq *decida* — de forma no determinística — respetar la cláusula de emergencia de
su propio prompt.

## Por qué pasa (arquitectura)

El workflow **no tiene ningún gate de emergencia determinístico**. "Emergencia" existe
únicamente como una instrucción de texto dentro del system prompt de los dos nodos Groq
(`Redactar Respuesta` y `Pedir Aclaración`): *"si sintomaOriginal suena a emergencia,
interrumpe todo"*. La evalúa el modelo, no el flujo.

Flujo relevante:

```
... → Function - Evaluar Coincidencia → Function - Memoria de Sesión
      → IF - ¿Especialidad Y Plan Encontrados?  (typeValidation strict, AND de 2 booleanos)
           ├─ true  → Notion×2 → Merge → Calcular Copago → Groq (Redactar Respuesta) → ...
           └─ false → Groq (Pedir Aclaración) → ...
```

**Sin memoria:** `premium` solo → `Detectar Especialidad` no encuentra keyword →
`especialidadEncontrada = false` → rama **FALSE** → `Groq (Pedir Aclaración)`, cuya
cláusula de emergencia dispara de forma consistente con síntomas
torácicos/respiratorios. El follow-up de una alarma se queda en la rama de aclaración.

**Con memoria:** la fusión completa el campo faltante —
`especialidadFinal = especialidadDetectadaAhora ?? memoria.especialidad` → Neumología
(de memoria). Ahora `especialidadEncontrada = true` Y `planEncontrado = true` → rama
**TRUE** → `Calcular Copago` produce el número (ej. $30 Neumología Premium) →
`Groq (Redactar Respuesta)` recibe el JSON con el copago ya calculado + el
`sintomaOriginal` fusionado.

La memoria no "resolvió" el síntoma ni se saltó una lógica de emergencia:
**reenrutó el turno de la rama de aclaración a la rama de cálculo de copago**, y en esa
rama la única barrera es la variabilidad de Groq.

Además: **todas** las keywords del diccionario de Neumología (`respirar`,
`falta de aire`, `ahogo`, `pulmon`…) son señales de alarma respiratoria. El sistema no
tiene el concepto "esta especialidad implica posible emergencia".

## Evidencia cruda (ejecuciones de producción, 2026-09-06)

| Exec | Turno | Msg | Rama IF | `sintomaOriginal` que recibió Groq | copago calculado | Bot |
|---|---|---|---|---|---|---|
| 112 | 1 | `Me cuesta respirar` | **CÁLCULO** | fusionado con sesión vieja (`"Tengo fiebre alta plan premium"`) | **57.5** | emergencia (Groq disparó) |
| 113 | 2 | `Premium` | **CÁLCULO** | fusionado | **30** | **copago $30** (Groq NO disparó — este fue el fallo original) |
| 115 | 1 | `me cuesta respirar` | Aclaración | crudo (sin sesión) | — | emergencia |
| 116 | 2 | `premium` | **CÁLCULO** | `El paciente describió antes: "me cuesta respirar". Ahora dice: "premium"` | **30** | emergencia (Groq disparó) |

- **113 y 116 son el mismo bug** con resultado distinto por azar de Groq. 113 mostró
  "$30" en una emergencia. 116 mostró emergencia pero **el $30 ya estaba calculado y a
  un paso de enviarse**.
- 112 muestra una variante peor: **incluso el turno 1** de 5b salta la rama de
  aclaración si hay cualquier sesión previa viva que aporte un plan.
- El "5b repetición 1 PASA" reportado durante la sesión era falso positivo: se miró el
  texto del bot (116), no la rama del IF ni el copago calculado.

## Criterio de test correcto para cuando se retome

5b NO se valida por el texto del bot (Groq dispara emergencia igual de forma consistente,
con o sin memoria rota). Criterio estructural:

- ✅ PASA = el IF **no** llega a CÁLCULO en el turno de seguimiento, **o**
  `Calcular Copago` **no** produce `copagoFinal` para ese turno.
- ❌ FALLA = se calculó un copago, aunque el bot haya dicho emergencia.

## Mitigaciones propuestas (NO implementadas — decisión pendiente)

Ninguna toca `prompts/` ni `temperature`.

Lista de keywords de alarma (normalizada, sin tildes), tomada de lo que los propios
prompts de Groq ya llaman emergencia — solo señales rojas reales, **sin**
`hueso`/`fractura`/`fiebre`/`abdomen`: `pecho`, `respirar`, `respiracion`,
`falta de aire`, `aire`, `ahog`, `asfixia`, `no puedo respirar`, `me cuesta respirar`,
`dificultad para respirar`, `sangrado`, `hemorragia`, `desmay`, `inconsciente`,
`perdida de conocimiento`, `convulsion`, `derrame`.

### Opción C — mínima (recomendada como primer paso)

En `Function - Memoria de Sesión`, después de construir el texto fusionado: si matchea
alarma → forzar `planEncontrado: false` + `faltantes: ['plan']`. El
`IF ¿Especialidad Y Plan?` existente lo manda a la rama FALSE (`Groq Pedir Aclaración`)
— exactamente la ruta del turno 1, que dispara emergencia de forma consistente.

- Nodos que toca: **1** (el de memoria).
- Dependencia de Groq: sí (misma cláusula que el sistema ya usa hoy).
- Riesgo: casi nulo — no crea ramas ni reconecta nada. Restaura el comportamiento
  pre-memoria.
- Tiempo: ~30 min + checkpoints.

### Opción B — determinística de verdad

`Function - Memoria de Sesión` emite `emergenciaDeterministica: true`. Nuevo
`IF - ¿Emergencia Determinística?` después del nodo de memoria → rama true a un nuevo
`Function - Mensaje de Emergencia` (texto fijo, **cero Groq**) → entra al
`Telegram - Responder al Paciente` existente. Rama false → sigue al
`IF ¿Especialidad Y Plan?` de siempre.

- Nodos: **1 modificado + 2 nuevos + reconexión** de la arista memoria→IF y una arista
  nueva hacia `Telegram - Responder`.
- Dependencia de Groq: **no**.
- Riesgo: medio — hay que replicar el shape `{mensaje: "..."}` que hoy arma
  `Formatear Respuesta de Groq` y que espera `Telegram - Responder`. Requiere
  re-verificar los 6 casos base + 5a/5b/5c completos.
- Tiempo: ~1.5–2 h + checkpoints completos.
- Nota: B también mejora el baseline preexistente (el pequeño porcentaje de fallo de
  Groq que ya existía antes de la memoria, en cualquier caso torácico/respiratorio que
  llegara a `Redactar Respuesta`).

## Orden de reversión (por si hay que volver a tocarlo)

Producción y backup comparten el `webhookId` del Telegram Trigger. **El orden importa:**

1. Desactivar el workflow con memoria PRIMERO.
2. Activar el backup DESPUÉS.

Validado en vivo el 2026-09-06, ~1 s por operación, sin conflicto de webhook.

## Artefactos fuera del repo

El export del workflow con memoria (desactivado, 28 nodos, con la versión instrumentada
del nodo de memoria) y el código JavaScript del nodo de memoria (versión limpia de
producción y versión instrumentada para checkpoints) quedan con el equipo, fuera del
repo — no forman parte de la entrega porque la entrega es el backup sin memoria.
