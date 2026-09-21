# ADR 0003 — Reglas de alarma sin umbrales fijos día/noche, y curtailment con RBAC

## Estado
Aceptado. **Nota (ADR 0004):** `rules.yaml` pasó a tener una sección por
`device_kind` (inverter/sts/weather/substation) en vez de una lista plana
de tags — las reglas de abajo son las de `inverter`, y siguen vigentes
sin cambios. El comando de actuador se sumó `set_reactive_power`
(soporte de reactiva, -100..+100%), con el mismo modelo *pull* y el mismo
RBAC que `set_power_limit`.

## Contexto
A diferencia del pozo (variables de proceso relativamente estables), la
generación solar varía en varios órdenes de magnitud entre la noche
(0 kW) y el mediodía (potencia nominal) — un umbral fijo de "potencia
baja" dispararía una alarma crítica cada noche, puro ruido.

También había que decidir cómo modelar la escritura de actuador de este
dominio: un inversor solar no tiene "start/stop" como una bomba, pero sí
un comando real que los operadores de red usan — el límite de potencia
remoto (curtailment).

## Decisión
- **Sin `high`/`low` fijos en `ac_power_kw` ni `dc_voltage_v`** en
  `rules.yaml` — solo `max_jump` (salto brusco, ej. paso de nube) y el
  detector de anomalías (z-score sobre ventana móvil), que sí entiende
  "distinto de lo normal para este momento del ciclo" sin necesitar un
  umbral absoluto.
- **`grid_frequency_hz` sí tiene banda fija** (49.5-50.5 Hz): a
  diferencia de la potencia, la frecuencia de red *debe* mantenerse en
  una banda angosta todo el tiempo en operación normal — un umbral fijo
  ahí sí es correcto y es exactamente lo que un inversor real reporta
  como condición de calidad de red.
- **Curtailment (`set_power_limit`) como comando de actuador**, mismo
  patrón de seguridad que el pozo: modelo *pull* (field-agent lo va a
  buscar), solo rol `admin` puede emitirlo, todo intento no autorizado
  queda auditado.

## Consecuencias
- El detector de anomalías (no el motor de reglas) es el que realmente
  cubre "potencia o irradiancia rara para esta hora del día" — separar
  ambas capas (ver también la misma decisión en el proyecto del pozo)
  fue igual de necesario acá, por un motivo de dominio distinto.
- Reducir la potencia de un inversor real (curtailment no autorizado)
  es un vector de ataque real documentado en la literatura de seguridad
  de redes eléctricas — la protección RBAC de este comando no es
  decorativa.
