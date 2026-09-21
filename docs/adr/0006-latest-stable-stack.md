# ADR 0006 — Siempre la última versión estable del stack

## Estado
Aceptado (2026-09-20)

## Contexto
El laboratorio arrastraba versiones que ya no eran las actuales: Python 3.12, React 18,
Vite 5, PostgreSQL 16, Redis 7, nginx 1.27, y librerías fijadas por lecciones viejas
(`pymodbus <3.8`, `bcrypt <5`, `redis <6`). Una versión fijada "porque andaba" no se
vuelve a probar nunca y termina siendo deuda de seguridad y de mantenimiento.

## Decisión
Se usa **la última versión estable** de cada pieza, verificada contra el registro
(PyPI, npm, Docker Hub, GitHub), nunca de memoria. "Estable" es la línea que el propio
proyecto marca como estable o de soporte largo:

| Pieza | Versión | Nota |
|---|---|---|
| Python | 3.14 (3.14.7) | 3.15 todavía no salió |
| Node | 24 (LTS) | la 26 es "Current" hasta octubre |
| pnpm | 12.5 | |
| PostgreSQL | 18.6 | salto mayor: se recreó el volumen (datos de simulación) |
| Redis | 8.10 | cliente `redis-py` 8.1 |
| Mosquitto | 2.1.2 | |
| nginx | 1.30 (rama *stable*) | |
| React / Vite / TypeScript | 19.3 / 8.3 / 7.0 | |
| FastAPI / Starlette | 0.141 / 1.6 | |
| bcrypt / PyJWT / pytest | 5.0 / 2.14 / 9.1 | |
| GitHub Actions | checkout 7, setup-python 7, setup-node 7, pnpm 6 | |

Las imágenes base siguen fijadas por **digest** (reproducible), pero se eligen con la
última versión estable, no la que "ya estaba". Los límites superiores de las
dependencias (`<4.0`, etc.) evitan un salto mayor no probado, no congelan el presente.

## pymodbus 3.15: se portó, no se fijó
La restricción `pymodbus <3.8` venía del proyecto hermano: desde esa versión el datastore
clásico (`ModbusSlaveContext` y los bloques de datos) quedó deprecado. En 3.15 esas clases
ya no exponen `getValues`/`setValues` y **se eliminan en la v4**. En vez de seguir fijando una
versión vieja, se migró a la API vigente (`SimDevice`/`SimData`) sin tocar la física de los
simuladores: cada equipo mantiene su estado en un `RegisterStore` propio
(`field_agent/modbus_store.py`, con la interfaz de siempre) y un adaptador lo sirve por
Modbus TCP: las lecturas de un cliente se atienden desde el almacén y sus escrituras
(comandos) caen ahí, donde las encuentra la conciliación de cada simulador. El cliente pasó de
`slave=` a `device_id=`. Un test de integración (`tests/test_modbus_store.py`) levanta el
servidor real y lo consulta con el cliente.

## Consecuencias
- Mantener esto al día es parte del trabajo, no un proyecto aparte: al tocar el repo se
  revisan los registros y se prueba todo antes de subir versiones.
- Un salto de versión mayor de PostgreSQL requiere migrar datos (`pg_upgrade` o
  volcado/restauración); acá, al ser una simulación, se recreó el volumen.
- La CI ahora corre con Python 3.14 y Node 24, iguales a las imágenes.
