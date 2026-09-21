# ADR 0001 — Mapa de registros inspirado en SunSpec, no el modelo SunSpec completo

## Estado
Aceptado

## Contexto
SunSpec Modbus es el protocolo real y vigente para inversores solares
conectados a la red en EE.UU. (exigido por la norma California Rule 21 e
IEEE 1547-2018/2030.5) y ampliamente usado en plantas utility-scale reales
— confirmado con búsqueda antes de decidir, no asumido de memoria (ver
[sunspec.org/modbus](https://sunspec.org/modbus/),
[REIG — plantas reales 5-250 MW](https://www.reig-us.com/sunspec-modbus-inverter-communications-utility-plants/)).

El modelo SunSpec completo, sin embargo, es un protocolo de *descubrimiento*
de modelos: un header común (`SunS`), un mapa de modelos numerados (1,
101/103, 122/124...) con longitud variable, pensado para que un maestro
genérico pueda auto-descubrir qué mide cualquier inversor de cualquier
fabricante sin configuración previa. Implementar ese mecanismo de
descubrimiento completo no aporta valor pedagógico extra para este
laboratorio (un solo inversor, un solo maestro que ya sabe qué está
leyendo) y sí una complejidad considerable.

## Decisión
Un mapa de registros Modbus **fijo e inspirado en los modelos SunSpec de
inversor (101/103) y acumulador de energía (122/124)**, no el protocolo de
descubrimiento completo:

- Magnitudes continuas (potencia AC, tensión/corriente DC, temperatura de
  módulo) escaladas x10 como enteros de 16 bits, igual convención que
  usan los modelos SunSpec reales.
- **Acumulador de energía de 32 bits partido en dos registros de 16**
  (`REG_ENERGY_WH_HIGH`/`REG_ENERGY_WH_LOW`) — así es como SunSpec
  representa contadores que superan el rango de un registro simple, y es
  el detalle más "real" de este mapa.
- Un registro de **límite de potencia remoto (curtailment)**, escribible
  — modela un comando real que los operadores de red sí envían a
  inversores utility-scale (reducir generación por orden de la red), y
  es el análogo de este proyecto al "comando de actuador" del pozo
  petrolero: mismo patrón de seguridad (modelo *pull*, RBAC admin-only),
  dominio distinto.

## Consecuencias
- No hay auto-descubrimiter de modelo: el agente conoce el mapa de
  memoria (documentado en `field_agent/models.py`), como conocería el
  mapa de un inversor específico ya integrado en producción.
- Queda documentado como "inspirado en SunSpec", no "SunSpec-compliant"
  — honesto sobre el alcance real, mismo criterio que se usó con
  DNP3/pymodbus en el proyecto hermano del pozo.
- El modelo de descubrimiento SunSpec completo queda en el roadmap si
  algún día se quiere generalizar a "conectate a cualquier inversor".
