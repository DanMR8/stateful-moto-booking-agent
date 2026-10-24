# Menú del Taller

Especificación prospectiva del menú para el modo con LLM: familias de servicio, entidades a extraer, casos de ambigüedad y límites de alcance.

## 1. Taxonomía de las Familias de Servicio

Cada familia define qué comprende el trabajo, su impacto en la agenda del taller y los datos mínimos que el LLM debe obtener antes de permitir el paso al agendamiento de fecha y hora.

```text
                         ┌─────────────────────────────────┐
                         │      Catálogo del Taller        │
                         └────────────────┬────────────────┘
                                          │
         ┌────────────────────────────────┼────────────────────────────────┐
         ▼                                ▼                                ▼
┌──────────────────┐             ┌──────────────────┐             ┌──────────────────┐
│  Mantenimiento   │             │ Desgaste Rápido  │             │   Diagnóstico    │
│   Preventivo     │             │  y Rodamiento    │             │     Técnico      │
└────────┬─────────┘             └────────┬─────────┘             └────────┬─────────┘
         │                                │                                │
         ├─ Servicio Básico               ├─ Kit de Arrastre               ├─ Fallas de Encendido
         ├─ Afinación Mayor               ├─ Llantas                       ├─ Ruidos de Motor
         └─ Cambio de Fluidos             └─ Frenos (Pastillas/Líquido)    └─ Fugas y Eléctrico
```

### Familia A: Mantenimiento Preventivo

Orientado a conservar la vida útil del motor y cumplir con los intervalos de servicio del fabricante.

Subtipos reconocidos:

- **Básico**: Cambio de aceite de motor y filtro de aceite.
- **Afinación Menor**: Aceite, filtro de aceite, filtro de aire y bujía.
- **Afinación Mayor**: Todo lo anterior más inspección/calibración de punterías (holgura de válvulas), lavado de cuerpo de aceleración o carburador, y revisión general de apriete de tornillería.

Slots obligatorios que el LLM debe resolver:

- `cilindrada_categoria`: Normalizado a tres rangos técnicos (`baja_250`, `media_650`, `alta_mas_650`).
- `kilometraje`: Número entero positivo extraído del texto (ej. si el usuario dice "ronda los doce mil quinientos", normalizar a `12500`).

Impacto operativo en agenda:

- **Básico**: Bloquea 45 minutos de rampa.
- **Mayor**: Bloquea 2.5 a 3 horas de rampa y requiere motor frío para calibración de válvulas (el LLM debe considerar esto para sugerir citas a primera hora del día).

### Familia B: Desgaste Rápido y Rodamiento

Orientado al reemplazo de consumibles críticos de tracción, frenado y contacto con el asfalto. No implica apertura de motor.

Subtipos reconocidos:

- **Kit de Arrastre**: Cadena (con o sin retenes/O-ring), piñón de ataque y corona/sprocket.
- **Llantas**: Montaje, desmontaje, balanceo estático y cambio de válvula.
- **Frenos**: Sustitución de pastillas/balatas, purga y cambio de líquido hidráulico (DOT 4 / DOT 5.1).

Slots obligatorios que el LLM debe resolver:

- `componente_desgaste`: Categorizado en `arrastre`, `llantas` o `frenos`.
- `posicion_eje`: Aplicable a llantas y frenos (`delantero`, `trasero`, `ambos`).
- `origen_refacciones`: Booleano estricto (`cliente_suministra` o `taller_cotiza`).

Impacto operativo en agenda:

- Bloquea 1 a 1.5 horas de rampa.
- Condición de negocio: Si `origen_refacciones` es `taller_cotiza`, el LLM debe marcar un aviso de que la cita queda sujeta a confirmación previa de existencia en almacén antes del día agendado.

### Familia C: Diagnóstico y Peritaje Técnico

Orientado a fallas activas, comportamiento errático o ruidos no identificados. No es una reparación inmediata, sino una ventana de inspección.

Subtipos reconocidos:

- **Sistema Eléctrico / Carga**: Batería descargada, falla de estator, regulador de voltaje o fusible principal.
- **Tren Motriz y Mecánica**: Cascabeleo interno, patinado de embrague (clutch), humo en el escape o sobrecalentamiento.
- **Fugas de Fluidos**: Pérdida de aceite por juntas/retenes, fuga de refrigerante o goteo de combustible.

Slots obligatorios que el LLM debe resolver:

- `sintoma_reportado`: Resumen técnico conciso de 1 a 2 oraciones depurado por el LLM a partir de la descripción informal del usuario.
- `condicion_ingreso`: Booleano o categoría (`rodando_por_cuenta_propia` vs `traslado_en_grua`).

Impacto operativo en agenda:

- Bloquea una ventana de recepción de 30 a 45 minutos para recepción técnica, levantamiento de inventario visual y prueba de campo inicial. No promete entrega el mismo día.
