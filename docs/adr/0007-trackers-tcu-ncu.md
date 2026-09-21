# ADR 0007 — Trackers: un TCU por tracker y un NCU por bloque

## Estado
Aceptado (2026-09-20). Implementado en sus cuatro fases.

## Contexto
El simulador modelaba cada equipo como "inversor + tracker" en un solo registro Modbus:
el inversor "giraba" los paneles, "tenía" la batería del motor y una irradiancia. En una
planta real el **inversor solo convierte** la corriente continua de los paneles en alterna
y la manda por baja tensión al STS. Lo que sigue al sol es el **tracker**, y lo mueve su
propio controlador. Esa simplificación confundía el modelo, el dashboard y hasta la
redacción de la documentación.

## Decisión
Se modelan los dos equipos reales del seguimiento solar (nombres verificados en documentación
del rubro):

- **TCU** (Tracker Control Unit): el controlador de *cada* tracker, con su motor, su panel
  solar y su batería. Acá hay **3 trackers por inversor** (75 por bloque, 450 en el parque).
- **NCU** (Network Control Unit): la unidad con la antena grande, **una por bloque**. Habla
  por radio con todos los TCU de su zona, tiene anemómetro propio (ordena la defensa ante
  viento fuerte sin esperar al SCADA) y habla con el SCADA por Modbus.

`field_agent/trackers.py` tiene la física (antes vivía dentro del inversor): seguimiento,
defensa a 75°, reposo nocturno, ventana de reset al atardecer, batería que carga con el sol,
motor trabado y pérdida del enlace de radio. El aprovechamiento de la luz de cada tracker es
el **coseno del error** entre su ángulo y el del sol; el de un inversor es el promedio de sus
3 trackers. Así un motor trabado le resta potencia solo al inversor que alimenta, y la defensa
a 75° casi no cuesta nada con el sol bajo del mismo lado (física real, no un error).

### Interfaz Modbus
El SCADA no ve 450 equipos: ve **un NCU por bloque** (unit ID 101): 32 registros de
encabezado (modo, conteos por estado, ángulo medio, calidad del radio, viento) más la tabla
de sus 75 trackers, 7 registros cada uno (ángulo, objetivo, estado, motor, corriente, batería,
banderas): 557 registros, que el agente lee **en tramos** (Modbus permite 125 por lectura). El
registro de modo se puede escribir: automático, defensa forzada o plano (limpieza).

### Backend
Tipo de equipo `tracker_gateway`. La lectura lleva el resumen como magnitudes sueltas y la
tabla como un arreglo por magnitud (compacto: ~340 bytes por lectura almacenada). Las alarmas
son de **conteo** por bloque (cuántos trackers tienen el problema), no una por tracker:
motores trabados, radio caído, baterías bajas y modo forzado por el operador. El comando
`set_tracker_mode` es solo de administrador. `/plant/summary` trae los conteos del NCU y
`/blocks/{id}` la tabla completa.

### Qué cambió en el inversor
Su registro ya no trae ángulo, estado, motor ni batería del tracker (los registros 13, 14 y
50..54 quedaron reservados). Recibe el aprovechamiento de sus trackers desde el NCU del bloque
en cada tick. Las reglas de alarma de tracker del inversor se quitaron: las cubre el NCU.

### Interfaz
En la ficha de cada bloque, una fila plegable "Trackers" (NCU): cuántos siguen al sol, están en
defensa o fallan, el modo vigente y, desplegada, los 75 trackers uno por uno (agrupados de a 3 por
inversor) con sus estadísticas; el administrador puede pasar el bloque a defensa forzada o a plano.
Cada inversor muestra en su ícono el ángulo medio de **sus** 3 trackers, y su detalle lista esos 3
(ángulo, estado, motor, batería, radio). Las alertas activas incluyen motores trabados, radio
degradado y modo forzado por el operador.

## Lo que queda simplificado (dicho con claridad)
- La irradiancia y la temperatura de módulo se siguen informando dentro del registro del
  inversor. En una planta real vienen de la estación meteorológica y de sensores de módulo,
  no del inversor. Moverlas es una limpieza pendiente.
- El enlace de radio NCU-TCU se modela como una pérdida aleatoria por tracker, no como una red
  con topología (malla, repetidores).

## Consecuencias
- Los comandos de campo pasan por el mismo modelo *pull* y control de acceso que el resto.
- Al probar esto apareció un error previo: el agente descartaba los comandos de todo equipo
  que no fuera inversor, así que las consignas de planta del PPC no llegaban nunca. Corregido,
  con test de regresión y verificado en vivo.
