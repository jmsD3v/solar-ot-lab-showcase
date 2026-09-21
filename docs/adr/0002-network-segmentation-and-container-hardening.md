# ADR 0002 — Segmentación de red y endurecimiento de contenedores (reutilizado)

## Estado
Aceptado. **Nota (ADR 0004):** los nombres de servicio de abajo
(`inverter-simulator`, `field-agent` en singular) son los del v1 de este
proyecto (un solo inversor). El patrón de segmentación/hardening sigue
vigente sin cambios; lo que cambió son los nombres reales de los
servicios que lo implementan — ahora `block-simulator-1..6` +
`central-simulator` en `ot-net`, y `field-agent-block-1..6` +
`field-agent-central` como los puentes OT↔DMZ (antes: uno de cada).

## Contexto
Mismo problema que en `critical-infra-secure-lab` (proyecto hermano):
separar zona OT/DMZ/IT de verdad, y decidir qué endurecimiento de
contenedores es seguro aplicar sin romper el arranque de las imágenes de
terceros.

## Decisión
Se reutiliza el patrón ya verificado en el proyecto hermano, sin
redescubrirlo:

- Tres redes Docker (`ot-net` internal, `dmz-net`, `it-net`).
  `field-agent` es el único puente OT↔DMZ; `backend` el único DMZ↔IT.
- `cap_drop: ALL` solo en `inverter-simulator`, `field-agent` y `backend`
  (código propio, sin patrón root→chown→setuid en runtime).
  `inverter-simulator` recupera `NET_BIND_SERVICE` para el puerto Modbus
  502.
- `postgres`, `redis`, `mosquitto`, `dashboard` **sin** `cap_drop`: sus
  entrypoints oficiales necesitan `chown`/`setuid` como root al
  arrancar — se probó lo contrario en el proyecto hermano y rompió los
  cuatro contenedores.
- `dashboard` (nginx) escucha en 8080, no 80, para no necesitar ninguna
  capability especial.

## Consecuencias
Se aplica de entrada lo que en el proyecto del pozo costó una ronda real
de `docker compose up` fallido para descubrir — acá no hace falta
repetir esa investigación.
