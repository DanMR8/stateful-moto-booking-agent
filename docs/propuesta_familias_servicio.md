# Propuesta de familias de servicio

Documentación de diseño para las familias de servicio del agente de agendamiento. Los puntos marcados como pendientes aún no están implementados.

## Familia A — Mantenimiento Preventivo

### Alcance y subtipos

- `basic_oil_change`: cambio de aceite y filtro de aceite.
- `minor_tuneup`: aceite, filtro de aceite, filtro de aire y bujía.
- `major_tuneup`: `minor_tuneup` más calibración de válvulas, limpieza de cuerpo de aceleración/carburador y revisión de apriete.

Regla de inclusión: si el usuario pide "afinación" sin detalle, se clasifica provisionalmente como `minor_tuneup` y se pregunta si incluye válvulas/carburador para elevar a `major_tuneup`.

### Slots obligatorios

| Slot | Tipo | Normalización | Validación |
|---|---|---|---|
| `service_subtype` | enum | mapeo de intención a subtipo canónico | requerido |
| `cilindrada_categoria` | enum `baja_250`, `media_650`, `alta_mas_650` | rangos por cc declarados | requerido |
| `kilometraje` | int | parseo de números hablados/abreviados | `> 0`, entero, máximo configurable |

Slots operativos derivados:

- `duration_minutes`: 45 (`basic`), 60 (`minor`), 150–180 (`major`).
- `requires_cold_engine`: `true` solo para `major_tuneup`.
- `preferred_window`: `first_morning_slots` para `major_tuneup`.

### Reglas de normalización

- Cilindrada:
  - `<= 250` → `baja_250`
  - `251–650` → `media_650`
  - `> 650` → `alta_mas_650`
  - Si el usuario da modelo/marca pero no cc, pedir cc o categoría.
- Kilometraje:
  - "doce mil quinientos" → `12500`
  - "12.5k" → `12500`
  - "12,500 km" → `12500`
  - Rechazar decimales, negativos o valores fuera de rango operativo.
- Subtipo:
  - "cambio de aceite" → `basic_oil_change`
  - "afinación" + "bujía/filtro de aire" → `minor_tuneup`
  - "válvulas", "carburador", "cuerpo de aceleración" → `major_tuneup`

### Validaciones de negocio

- No pasar a fecha/hora hasta que los tres slots obligatorios estén válidos.
- `major_tuneup` debe sugerir las primeras citas de la mañana por motor frío.
- Advertencia suave si el kilometraje no es coherente con el intervalo típico del subtipo; no bloquear por sí sola.
- Si el usuario corrige un slot, recalcular duración y ventana preferida.
- Si falta un slot, preguntar solo por ese slot, no reiniciar la captura.

### Estado y transiciones

Estado mínimo por turno:

```text
family = preventive
service_subtype = ...
cilindrada_categoria = ...
kilometraje = ...
missing_slots = [...]
validation_errors = [...]
ready_to_schedule = bool
```

Transiciones:

1. Detectar familia A.
2. Extraer candidatos de slots.
3. Fusionar solo valores válidos, conservando los anteriores si el nuevo es inválido.
4. Recalcular `missing_slots`.
5. Si `missing_slots` vacío, marcar `ready_to_schedule = true`.
6. Si `major_tuneup`, añadir restricción de horario temprano.

Corrección retroactiva:

- Comando "corregir kilometraje a 15000" actualiza solo ese slot.
- Comando "volver" restaura el snapshot anterior del checkpoint.
- FAQ no debe limpiar slots ni cambiar estado de captura.

### Impacto operativo y agenda

- `basic_oil_change`: 45 min, una rampa.
- `minor_tuneup`: 60 min, una rampa.
- `major_tuneup`: 150–180 min, una rampa, preferencia de primera hora.
- Antes de confirmar, validar solapamiento por rampa y horario.
- Si no hay ventana válida para `major_tuneup`, proponer siguiente día en primer horario disponible.

### Casos de ambigüedad

- "Afinación" sin alcance: preguntar si incluye válvulas/carburador.
- "Moto 650" sin cc exacto: clasificar como `media_650`, pero confirmar si es 600, 650 o 700.
- "Tiene como 12 mil" → confirmar `12000` antes de validar.
- Usuario mezcla familia A con desgaste: separar en dos citas o pedir prioridad.
- Usuario pide "lo completo" → asumir `major_tuneup` solo tras confirmación.

### Pruebas mínimas

- Normalización de cc hablado y numérico.
- Normalización de kilometraje hablado/abreviado.
- Clasificación de subtipo por palabras clave.
- Pregunta dirigida por slot faltante.
- Bloqueo de agendamiento con slots incompletos.
- Regla de primera hora para `major_tuneup`.
- Conflicto de capacidad por rampa.
- Corrección de un slot sin perder los demás.
- Rollback de turno.
- FAQ sin pérdida de estado.

### Riesgos y mitigaciones

- **Ambigüedad de "afinación"**: resolver con pregunta de alcance antes de duración.
- **Números hablados mal parseados**: pedir confirmación del valor normalizado.
- **Sobrepromesa de horario para válvulas**: forzar ventana temprana o siguiente día.
- **Mezcla de servicios**: no combinar automáticamente; proponer citas separadas.

## Familia B — Desgaste Rápido y Rodamiento

### Alcance y subtipos

- `chain_kit`: cadena, piñón de ataque y corona/sprocket; incluye variante con o sin retenes/O-ring.
- `tires`: montaje, desmontaje, balanceo estático y cambio de válvula.
- `brakes`: pastillas/balatas, purga y cambio de líquido hidráulico (DOT 4 / DOT 5.1).

Regla de inclusión: si el usuario solo dice "frenos" o "llantas", se clasifica provisionalmente y se pregunta por posición y origen de refacciones antes de agendar.

### Slots obligatorios

| Slot | Tipo | Valores canónicos | Validación |
|---|---|---|---|
| `wear_component` | enum | `chain_kit`, `tires`, `brakes` | requerido |
| `axle_position` | enum | `front`, `rear`, `both` | requerido para `tires` y `brakes`; no aplicable a `chain_kit` |
| `parts_origin` | enum | `client_supplies`, `shop_quotes` | requerido |
| `chain_variant` | enum | `with_o_ring`, `without_o_ring`, `unknown` | requerido solo si `wear_component = chain_kit` |
| `brake_fluid` | enum | `dot_4`, `dot_5_1`, `unknown` | opcional para `brakes`; confirmar si el usuario lo menciona |

Slots operativos derivados:

- `duration_minutes`: 60–90 según componente y posición.
- `ramp_minutes`: bloqueo de 1 a 1.5 horas.
- `parts_confirmation_required`: `true` si `parts_origin = shop_quotes`.
- `service_scope`: `consumables_only`, sin apertura de motor.

### Reglas de normalización

- Componente:
  - "kit de arrastre", "cadena", "piñón", "corona" → `chain_kit`
  - "llanta", "neumático", "montaje", "balanceo" → `tires`
  - "frenos", "pastillas", "balatas", "purga", "líquido" → `brakes`
- Posición:
  - "delantera", "frontal" → `front`
  - "trasera" → `rear`
  - "ambas", "las dos", "delante y atrás" → `both`
  - Si el usuario dice "cambio de llanta" sin posición, preguntar `front`, `rear` o `both`.
- Origen de refacciones:
  - "yo llevo las piezas", "las compro yo" → `client_supplies`
  - "ustedes cotizan", "lo consiguen en el taller" → `shop_quotes`
  - Si no se menciona, preguntar explícitamente antes de agendar.
- Cadena:
  - "con retenes", "O-ring", "sellada" → `with_o_ring`
  - "sin retenes", "estándar" → `without_o_ring`
  - Si no se sabe, marcar `unknown` y confirmar en recepción.

### Validaciones de negocio

- No agendar hasta resolver `wear_component`, `parts_origin` y, cuando aplique, `axle_position`.
- `chain_kit` no requiere posición de eje; si el usuario la menciona, registrarla como nota, no como slot obligatorio.
- `tires` y `brakes` requieren posición; `both` duplica tiempo operativo y debe reflejarse en duración.
- Si `parts_origin = shop_quotes`, la cita queda sujeta a confirmación previa de existencia en almacén antes del día agendado.
- Si `parts_origin = client_supplies`, el LLM debe recordar que el cliente debe llevar las piezas correctas y en buen estado.
- No prometer reparación de motor ni apertura de caja; si el usuario describe falla interna, redirigir a Familia C.

### Estado y transiciones

Estado mínimo por turno:

```text
family = wear
wear_component = ...
axle_position = ... | not_applicable
parts_origin = ...
chain_variant = ... | not_applicable
brake_fluid = ... | unknown
missing_slots = [...]
validation_errors = [...]
ready_to_schedule = bool
parts_confirmation_required = bool
```

Transiciones:

1. Detectar familia B.
2. Extraer componente, posición y origen.
3. Validar aplicabilidad de posición según componente.
4. Fusionar solo valores válidos.
5. Si `parts_origin = shop_quotes`, marcar `parts_confirmation_required = true`.
6. Si faltan slots, preguntar solo por el faltante.
7. Si todos los slots obligatorios están resueltos, marcar `ready_to_schedule = true`.

Corrección retroactiva:

- "corregir posición a trasera" actualiza solo `axle_position`.
- "corregir origen a yo llevo las piezas" actualiza `parts_origin` y recalcula `parts_confirmation_required`.
- FAQ sobre precios o tiempos no debe limpiar slots.
- Rollback restaura el snapshot anterior del checkpoint.

### Impacto operativo y agenda

- `chain_kit`: 60–90 min, una rampa.
- `tires`: 60 min para una posición; 90–120 min para `both`.
- `brakes`: 60 min para una posición; 90 min para `both`.
- Bloqueo general: 1 a 1.5 horas de rampa.
- Si `parts_origin = shop_quotes`, no confirmar la cita como definitiva hasta validar inventario; mantener estado `pending_parts_confirmation`.
- Si el cliente lleva piezas, la cita puede confirmarse directamente si hay rampa disponible.

### Casos de ambigüedad

- "Cambio de llanta" sin posición: preguntar delantera, trasera o ambas.
- "Frenos" sin posición: preguntar delantera, trasera o ambos ejes.
- "Yo llevo la cadena" pero no especifica con/sin retenes: marcar `unknown` y confirmar en recepción.
- "Necesito frenos y llantas": tratar como una cita combinada solo si el taller permite; por defecto proponer servicios separados o una cita ampliada.
- "Ustedes consiguen todo": marcar `shop_quotes` y activar confirmación de inventario.
- Usuario describe "ruido en la cadena" como falla: si implica diagnóstico interno, redirigir a Familia C.
- Usuario pide "balanceo" sin cambio de llanta: clasificar como `tires` y preguntar si también incluye montaje/desmontaje.

### Pruebas mínimas

- Clasificación de componente por palabras clave.
- Normalización de posición.
- Normalización de origen de refacciones.
- Rechazo de posición para `chain_kit` como slot obligatorio.
- Duración correcta para `both`.
- Activación de `parts_confirmation_required` con `shop_quotes`.
- Pregunta dirigida por slot faltante.
- Bloqueo de agendamiento con slots incompletos.
- Corrección de posición sin perder componente.
- Corrección de origen y recálculo de confirmación.
- Rollback de turno.
- FAQ sin pérdida de estado.
- Redirección a Familia C ante falla interna.

### Riesgos y mitigaciones

- **Piezas incorrectas del cliente**: recordar compatibilidad y estado antes de confirmar.
- **Inventario no disponible**: mantener estado pendiente y notificar antes del día agendado.
- **Duración subestimada en `both`**: aplicar multiplicador operativo.
- **Ambigüedad entre mantenimiento y desgaste**: priorizar intención explícita y preguntar si hay mezcla.
- **Diagnóstico encubierto**: no prometer reparación; redirigir a Familia C.

## Familia C — Diagnóstico y Peritaje Técnico

### Alcance y subtipos

- `electrical_charging`: batería descargada, falla de estator, regulador de voltaje o fusible principal.
- `drivetrain_mechanical`: cascabeleo interno, patinado de embrague, humo en escape o sobrecalentamiento.
- `fluid_leaks`: pérdida de aceite por juntas/retenes, fuga de refrigerante o goteo de combustible.

Regla de inclusión: si el usuario describe una falla activa, comportamiento errático o ruido no identificado, se clasifica provisionalmente como diagnóstico y se depura el síntoma antes de agendar.

### Slots obligatorios

| Slot | Tipo | Valores canónicos | Validación |
|---|---|---|---|
| `diagnostic_area` | enum | `electrical_charging`, `drivetrain_mechanical`, `fluid_leaks` | requerido |
| `symptom_summary` | string | resumen técnico de 1–2 oraciones | requerido, 20–240 caracteres |
| `intake_condition` | enum | `rides_under_own_power`, `towed` | requerido |
| `symptom_onset` | enum | `recent`, `intermittent`, `persistent`, `unknown` | opcional, recomendado |
| `warning_lights` | bool | presencia de testigo de falla | opcional |
| `fluid_type` | enum | `engine_oil`, `coolant`, `fuel`, `unknown` | requerido solo si `diagnostic_area = fluid_leaks` |

Slots operativos derivados:

- `duration_minutes`: 30–45 para recepción técnica.
- `service_scope`: `inspection_only`, no reparación inmediata.
- `same_day_delivery`: `false`.
- `requires_visual_inventory`: `true`.
- `requires_field_test`: `true` cuando la motocicleta ingresa rodando.

### Reglas de normalización

- Área diagnóstica:
  - "no enciende", "batería", "estator", "regulador", "fusible" → `electrical_charging`
  - "cascabeleo", "embrague patina", "humo", "se calienta" → `drivetrain_mechanical`
  - "pierde aceite", "fuga de refrigerante", "gotea gasolina" → `fluid_leaks`
- Síntoma:
  - Depurar lenguaje informal a resumen técnico conciso.
  - Eliminar especulaciones del usuario ("creo que es el motor") y conservar observaciones ("humo blanco al acelerar").
  - Mantener 1–2 oraciones; si excede, resumir y guardar detalle original en notas.
- Condición de ingreso:
  - "llego manejando", "sí arranca", "la traigo rodando" → `rides_under_own_power`
  - "grúa", "no arranca", "la remolcan" → `towed`
- Fluido:
  - "aceite" → `engine_oil`
  - "refrigerante", "anticongelante" → `coolant`
  - "gasolina", "combustible" → `fuel`
  - Si no se identifica, marcar `unknown` y confirmar en recepción.

### Validaciones de negocio

- No agendar hasta resolver `diagnostic_area`, `symptom_summary` y `intake_condition`.
- `symptom_summary` debe ser técnico, no una transcripción literal ni una conclusión de reparación.
- No prometer entrega el mismo día.
- No cotizar reparación antes de inspección.
- Si el usuario pide "arreglar" de inmediato, aclarar que primero se agenda ventana de diagnóstico.
- Si `intake_condition = towed`, registrar necesidad de espacio de descarga y posible apoyo para ingreso.
- Si hay fuga de combustible, marcar precaución de seguridad y no prometer manejo interno sin verificación.

### Estado y transiciones

Estado mínimo por turno:

```text
family = diagnostic
diagnostic_area = ...
symptom_summary = ...
intake_condition = ...
symptom_onset = ...
warning_lights = ...
fluid_type = ... | not_applicable
missing_slots = [...]
validation_errors = [...]
ready_to_schedule = bool
safety_flags = [...]
```

Transiciones:

1. Detectar familia C.
2. Extraer área diagnóstica.
3. Depurar `symptom_summary`.
4. Extraer condición de ingreso.
5. Extraer fluido si aplica.
6. Fusionar solo valores válidos.
7. Calcular `missing_slots`.
8. Si todos los slots obligatorios están resueltos, marcar `ready_to_schedule = true`.
9. Si hay riesgo de seguridad, añadir `safety_flags`.

Corrección retroactiva:

- "corregir síntoma: humo negro al acelerar" actualiza solo `symptom_summary`.
- "corregir ingreso: la traigo en grúa" actualiza `intake_condition`.
- FAQ sobre costos o tiempo no debe limpiar slots.
- Rollback restaura el snapshot anterior del checkpoint.

### Impacto operativo y agenda

- Bloquea 30–45 minutos de recepción técnica.
- Incluye levantamiento de inventario visual y prueba de campo inicial.
- No promete entrega el mismo día.
- Si `intake_condition = rides_under_own_power`, reservar espacio para prueba de campo.
- Si `intake_condition = towed`, reservar apoyo de descarga y no asumir prueba de manejo inmediata.
- Si `fluid_leaks` con combustible, priorizar recepción segura y aislamiento básico.

### Casos de ambigüedad

- "Hace un ruido raro" sin área: preguntar si es eléctrico, mecánico o fuga.
- "No arranca" puede ser batería, estator o combustible: clasificar provisionalmente y pedir síntoma adicional.
- "Pierde líquido" sin tipo: preguntar si es aceite, refrigerante o gasolina.
- Usuario concluye "es el motor": convertir a observación y no registrar conclusión como diagnóstico.
- Usuario mezcla diagnóstico con reparación: separar inspección de reparación y explicar que la reparación se cotiza después.
- Usuario pide entrega el mismo día: aclarar que el diagnóstico no garantiza reparación inmediata.
- Usuario describe múltiples síntomas: resumir los principales y dejar el resto en notas.

### Pruebas mínimas

- Clasificación de área diagnóstica por palabras clave.
- Depuración de síntoma informal a resumen técnico.
- Normalización de condición de ingreso.
- Normalización de tipo de fluido.
- Rechazo de conclusión de reparación como síntoma.
- Pregunta dirigida por slot faltante.
- Bloqueo de agendamiento con slots incompletos.
- Duración de recepción 30–45 min.
- `same_day_delivery = false`.
- Activación de `safety_flags` para fuga de combustible.
- Corrección de síntoma sin perder área diagnóstica.
- Rollback de turno.
- FAQ sin pérdida de estado.
- Redirección desde familia A/B cuando se describe falla interna.

### Riesgos y mitigaciones

- **Sobrediagnóstico por lenguaje del usuario**: depurar y no registrar conclusiones.
- **Promesa de entrega**: bloquear explícitamente entrega el mismo día.
- **Riesgo de seguridad por combustible**: marcar bandera y priorizar recepción segura.
- **Síntomas múltiples o vagos**: resumir y dejar detalle en notas.
- **Confusión con reparación**: separar diagnóstico de reparación y cotización posterior.

## Evaluación conjunta

Las tres familias están bien separadas por intención y cubren preventivo, desgaste y diagnóstico, con slots claros y reglas operativas diferenciadas. Como unidad, comparten un patrón común: detección de familia, extracción/fusión de slots, validación, transición a agendamiento, persistencia transaccional y rollback mediante checkpoint.

## Tablas compartidas

- `appointments`: tabla común de agenda con `thread_id`, `family`, `service_code`, `start_at`, `duration_minutes`, `resource_id`, `status`, `created_at`.
- `appointment_details`: JSON validado por Pydantic según familia, o tablas específicas por familia si se necesita consulta/reportes fuertes.
- `customers` y `vehicles`: entidades compartidas que faltan en el menú, pero necesarias para agendar citas reales.
- `resources`: rampas/bahías de recepción, con capacidad y tipo de recurso.
- `appointment_status_history`: auditoría de cambios de estado.

## Puntos de mejora pendientes de implementación

- [ ] Usar un `DialogueState` con unión discriminada por `family`, evitando campos opcionales ambiguos.
- [ ] Centralizar duraciones, recursos y reglas de horario en un módulo de scheduling.
- [ ] Agregar `customer_id`, `vehicle_id`, idempotencia por `thread_id + turno` y política de cancelación/reprogramación.
- [ ] Diferenciar estados genéricos (`scheduled`, `cancelled`) de estados familiares (`pending_parts_confirmation`, `received`, `inspected`).
- [ ] Mantener checkpoint solo para estado conversacional; la cita se crea transaccionalmente al confirmar.
- [ ] Añadir pruebas de contrato compartidas: fusión de slots, FAQ no destructiva, rollback y capacidad de recursos.
- [ ] Definir migraciones SQLite para tablas compartidas y detalles por familia.
- [ ] Definir enums centralizados para familias, subtipos, estados y recursos.
