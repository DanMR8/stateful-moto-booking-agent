# Especificación de Diseño y Comportamiento: Moto Booking Agent

**Propósito del Documento:**
Esta es la fuente de verdad (SSOT) para el desarrollo del proyecto `stateful-moto-booking-agent`. Define el comportamiento esperado, las reglas de negocio, la taxonomía de servicios y los límites operativos del agente conversacional. **No contiene código de implementación**, sino las pautas conceptuales y arquitectónicas que el código (LangGraph, Python, SQLite) debe satisfacer rigurosamente.

---

## 1. Patrón Arquitectónico Fundamental: "Thin LLM, Fat State" (Ejecución Bivalente)

El sistema debe desarrollarse delegando la menor carga cognitiva posible al LLM. 

* **Capa Determinística (Código):** Se encarga de la extracción de datos mediante Regex/Keywords, evaluación de reglas matemáticas, control de tiempos (TTL), manejo del estado, prevención de colisiones en la agenda y validación estricta.
* **Capa Semántica (LLM):** Se limita a entender intenciones complejas (cuando el determinismo falla), traducir el estado del sistema a lenguaje comercial (sin jerga técnica como "bloqueo de rampa") y depurar resúmenes informales.

---

## 2. Taxonomía de Servicios y Reglas de Negocio

El taller atiende tres familias exclusivas. Las intenciones cruzadas no se mezclan en el estado; el agente debe forzar al usuario a priorizar.

### Familia A: Mantenimiento Preventivo
Orientado a conservar la vida útil del motor.
* **Subtipos:** Básico (45 min), Menor (60 min), Mayor (150-180 min).
* **Regla Crítica:** El "Servicio Mayor" requiere motor frío. El sistema fuerza la sugerencia de horarios en las primeras horas de la mañana.

### Familia B: Desgaste Rápido y Rodamiento
Reemplazo de consumibles (sin abrir motor). Bloquea de 60 a 120 minutos.
* **Subtipos:** Llantas, Frenos, Kit de Arrastre.
* **Reglas Críticas:** 
    * Si el taller cotiza (`shop_quotes`), se inyecta la advertencia de que la cita está sujeta a confirmación de almacén.
    * Si el cliente suministra (`client_supplies`), el agente recuerda amablemente que es responsabilidad del usuario llevar piezas compatibles.

### Familia C: Diagnóstico Técnico
Revisión de fallas activas. No es una reparación inmediata (Bloquea 30-45 mins).
* **Subtipos:** Eléctrico/Carga, Mecánico/Tren Motriz, Fugas.
* **Reglas Críticas:** 
    * Nunca prometer entrega el mismo día. La cita es solo para inspección y cotización posterior.
    * Si ingresa en grúa (`towed`), se añade un flag invisible para que el taller prevea espacio en el estacionamiento exterior (no se cobra extra ni se restringe horario).
    * Si la fuga es de combustible, se activa bandera de seguridad.

---

## 3. Modelo de Estado Central (DialogueState)

El grafo opera sobre una **Unión Discriminada**. Solo una familia de servicio puede estar activa a la vez.

```text
[ METADATOS DE CONTROL ]
 ├─ active_family:      [ None | Familia_A | Familia_B | Familia_C ]
 ├─ missing_slots:      [ Lista de campos requeridos pendientes ]
 ├─ validation_errors:  [ Lista de incoherencias inyectadas por la capa determinística ]
 ├─ safety_flags:       [ Alertas de riesgo operativo ]
 └─ ready_to_schedule:  [ Booleano. Se activa solo si missing_slots = 0 ]

[ PAYLOAD (Ejemplo activo: Familia_B) ]
 ├─ wear_component:       [ Arrastre | Llantas | Frenos ]
 ├─ axle_position:        [ Delantero | Trasero | Ambos | No_Aplica ]
 ├─ parts_origin:         [ Cliente_Suministra | Taller_Cotiza ]
 ├─ chain_variant:        [ Con_O_Ring | Sin_O_Ring | Desconocido ]
 └─ brake_fluid:          [ DOT_4 | DOT_5_1 | Desconocido ]
```

---

## 4. Matrices de Ejecución Bivalente (Extracción y Validación)

La siguiente tabla dicta cómo la capa determinística intenta resolver cada slot antes de llamar al LLM.

### Matriz 1: Familia A (Mantenimiento)
| Campo (Slot) | Extracción Determinística (Pre-LLM) | Validación Lógica (Reglas de Negocio) | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Subtipo** | Keywords: "aceite" (Básico), "bujía" (Menor), "válvulas/carburador" (Mayor). | Asignación estricta. Si dice "afinación", marca error de ambigüedad. | Pregunta si la afinación requiere calibrar válvulas para definir alcance. |
| **Cilindrada** | Regex: `\b\d{2,4}\s*(cc|cm3)\b`. | Mapeo matemático: `<=250` (baja), `251-650` (media), `>650` (alta). | Pide los CC si el usuario solo dice "es una Ninja" o "Vento". |
| **Kilometraje** | Regex: limpia comas, traduce "12.5k" a `12500`. | Verifica entero `> 0` y `< 300,000`. Rechaza decimales/negativos. | Traduce frases como "recién salida" a número exacto, pidiendo ver el tablero. |

### Matriz 2: Familia B (Desgaste Rápido)
| Campo (Slot) | Extracción Determinística (Pre-LLM) | Validación Lógica (Reglas de Negocio) | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Componente** | Keywords: "cadena/arrastre", "llanta", "freno/balata". | Pivote principal. Asignación a `chain_kit`, `tires`, `brakes`. | Clarifica ambigüedades. Redirige a Fam. C si describe fallas mecánicas. |
| **Eje** | Keywords: "delantera", "trasera", "ambas". | Ignorado para `chain_kit`. Obligatorio para frenos/llantas. | Pregunta qué llanta/freno si no se especificó. |
| **Refacciones** | Keywords: "llevo/compré" vs "cotiza/taller". | Asigna y activa bandera `parts_confirmation_required` si cotiza taller. | Pregunta explícitamente el origen si no hay mención. |
| **Cadena/Fluido** | Keywords específicas ("o-ring", "dot4"). | Evaluados solo si aplica al componente. No bloquean agenda. | Asume `unknown` si no hay datos; se revisa en recepción. |

### Matriz 3: Familia C (Diagnóstico Técnico)
| Campo (Slot) | Extracción Determinística (Pre-LLM) | Validación Lógica (Reglas de Negocio) | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Área** | Keywords: "batería" (Eléctrico), "ruido/humo" (Mecánico), "tira/charco" (Fugas). | Pivote principal de ruteo. | Clasifica síntomas vagos formulando preguntas dirigidas (motor, eléctrico o fuga). |
| **Síntoma** | *Ninguna. Texto libre.* | *Confianza en modelo.* | Depura descripción informal a resumen técnico (1-2 oraciones). Omite especulaciones. |
| **Ingreso** | Keywords: "rodando/manejando" vs "grúa/remolque". | Si es `towed`, inyecta flag de espacio exterior. | Traduce frases como "me empujó un amigo". |
| **Fluido** | Keywords: "aceite", "refrigerante", "gasolina". | Activa alerta de seguridad si es gasolina. | Pregunta color/textura si el cliente no sabe qué gotea. |

### Matriz 4: Módulo de Agendamiento
| Campo (Slot) | Extracción Determinística (Pre-LLM) | Validación Lógica (Reglas de Negocio) | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Día** | Keywords temporales ("hoy", "lunes"). | Convierte a fecha ISO (ej. `2026-10-27`). Rechaza días no laborables. | Interpreta días coloquiales o busca el próximo espacio si no hay preferencia. |
| **Turno** | Keywords: "mañana", "tarde", "pm". | Mapea a bloques. Fuerza "Mañana" si el estado es Servicio Mayor (Fam. A). | Pide aclarar frases como "después de comer". |
| **Tool** | Ruteo automático basado en parámetros provistos. | Envía duraciones calculadas por la Familia a la consulta DB. | Si no hay disponibilidad, ofrece proactivamente otra semana. |
| **Decisión** | Keywords: "sí", "ok" vs "no", "cambiar". | Si aprueba: Soft-lock de horario por 10 min. Activa módulo Checkout. | Si rechaza: Pregunta alternativas y retoma búsqueda. |

### Matriz 5: Cierre de Cita (Fast-Checkout)
| Campo (Slot) | Extracción Determinística (Pre-LLM) | Validación Lógica (Reglas de Negocio) | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Nombre** | Limpieza y Title Case. | Longitud > 2. Rechaza puros números. | Extrae nombre válido si el formato es muy coloquial. |
| **Teléfono** | Limpieza de no-numéricos. Regex 10 dígitos. | Rechaza si inicia con `0`. | Lee error de estado e insiste amablemente por 10 dígitos. |
| **Moto** | *Texto libre.* | Longitud > 3. | Extrae entidad de vehículo y exige al menos la marca. |

---

## 5. Módulo de Calendario y Propuesta Proactiva

El agente actúa como enrutador hacia las herramientas (Tools) de la Base de Datos y propone espacios **proactivamente**, inyectando los "Disclaimers Transaccionales" (advertencias del taller) antes de la confirmación:
* `buscar_proximo_espacio`: Por defecto.
* `buscar_por_dia_semana_actual`: Para días específicos.
* `buscar_proxima_semana`: Para preferencias diferidas.
* `buscar_desde_fecha_horario`: Para restricciones estrictas.

**Disclaimer General Obligatorio:** "Tienes 30 minutos de tolerancia; si llegas un poco después, te atenderemos conforme a disponibilidad de rampas libres."

---

## 6. Manejo de Sesión, Tiempos y Colisiones

Dado el contexto asíncrono (ej. WhatsApp):

* **Interceptor TTL de 48 Horas:** Si `hora_actual - last_updated_at > 48h`, la capa determinística purga todo el `DialogueState`. La conversación reinicia obligatoriamente como una nueva intención.
* **Doble Validación (Stale Data):** Si el usuario tarda horas en aceptar una fecha propuesta y el TTL sigue activo, el sistema hace una *validación determinística silenciosa* en la DB antes de confirmar. Si el espacio ya se ocupó, el agente inyecta un error de colisión disculpándose y ofreciendo otro horario.
* **Soft-Lock (10 mins):** Durante la recolección de los datos de Checkout, el horario aceptado se pre-reserva por 10 minutos.

---

## 7. Operacional

### 7.1. Rollback de Estado (State Rollback)

El sistema mantiene checkpoints de estado en cada turno. Permite:

* **Comando "volver":** Restaura el snapshot anterior sin perder historial.
* **Corrección granular:** "corregir kilometraje a 15000" actualiza solo ese slot sin afectar los demás.
* **FAQ no destructiva:** Preguntas de ayuda no alteran slots ni estado de captura.

### 7.2. Validación con Pydantic

Todos los esquemas de estado (`DialogueState`) y slots se validan con Pydantic v2.x:

* **Integridad de datos:** Evita corrupción al guardar en DB o persistir en checkpoint.
* **Salida estructurada del LLM:** El modelo `TurnAnalysis` devuelve JSON válido sin parseo manual.
* **Errores legibles:** Mensajes de validación claros para el usuario ("kilometraje debe ser entero positivo").
* **Contrato compartido:** LangGraph, DB y herramientas hablan el mismo formato.

### 7.3. Cancelación y Reprogramación

### 7.2. Idempotencia de Citas

Para evitar duplicación en entornos asíncronos:

* Cada cita se identifica con `thread_id + turno`.
* El módulo de agendamiento verifica idempotencia antes de insertar.
* Si se recibe la misma cita en dos turnos, se actualiza en lugar de duplicar.

### 7.3. Cancelación y Reprogramación

El sistema distingue entre cancelación y reprogramación, manteniendo el historial de citas.

**Cancelación:**

| Campo | Extracción Determinística | Validación Lógica | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Intención** | Keywords: "cancelar", "no voy", "borrar cita" | Confirma que el usuario desea cancelar explícitamente | Pregunta "¿Seguro que deseas cancelar esta cita?" |
| **Cita a cancelar** | Extrae `thread_id` o ID de cita del contexto | Verifica que la cita exista y esté en estado `scheduled` | Si el ID no es válido, pide el número de cita o reanuda búsqueda |

**Reprogramación:**

| Campo | Extracción Determinística | Validación Lógica | Fallback Semántico (LLM) |
| :--- | :--- | :--- | :--- |
| **Intención** | Keywords: "reprogramar", "cambiar fecha", "otra hora" | Confirma que desea reprogramar la misma cita | Pregunta "¿Deseas reprogramar la cita en otra fecha/hora?" |
| **Nuevo horario** | Mismo flujo de agendamiento (día/turno) | Reutiliza `thread_id`, limpia `missing_slots`, mantiene datos de servicio | Interpreta preferencias vagas ("quién sabe, la próxima semana") |

**Soft-Lock expirado:** Si el usuario rechaza la primera propuesta, el soft-lock se libera y se busca espacio alternativo.

### 7.4. TTL de Sesión

* **Variable configurable:** `SESSION_TTL_HOURS` con default de 48 horas (ajustable).
* **Comportamiento:** Si el usuario no retoma la conversación en `SESSION_TTL_HOURS`, el sistema purga el estado y reinicia la intención.
* **UX:** Antes de exceder el TTL, el sistema recuerda amablemente la conversación pendiente.
* **TODO:** Documentar políticas de UX en `proposal_familias_servicio.md`

* **FAQ con Retorno Activo (Answer & Re-prompt):** Si el usuario interrumpe para preguntar "¿aceptan tarjeta?", el agente responde usando RAG/Knowledge Base y, en el mismo mensaje, vuelve a preguntar por el slot faltante. **El estado no se altera.**
* **Diagnóstico Encubierto:** Si el usuario exige una reparación compleja inmediata ("cámbiale el estator hoy"), el sistema bloquea, cambia a Familia C, asume área eléctrica y corrige la expectativa (solo inspección).
* **Multi-intención:** Si mezcla mantenimientos (Fam A) con fallas graves (Fam C), el agente detiene la recolección y fuerza al usuario a decidir cuál de los dos servicios quiere agendar para esa cita.