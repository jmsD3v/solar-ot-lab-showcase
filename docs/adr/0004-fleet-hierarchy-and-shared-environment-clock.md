# ADR 0004 — Flota jerárquica real (150 inversores) y reloj de entorno compartido

## Estado
Aceptado

## Contexto
La v1 de este proyecto simulaba **un solo inversor**. Un parque solar
utility-scale real no tiene un inversor: tiene decenas por bloque
agrupados en un STS (estación transformadora de bloque), varios bloques
interconectados en media tensión hacia una estación transformadora
central, y una estación meteorológica independiente que gobierna la
posición de defensa de los trackers ante viento fuerte — jerarquía
descrita en detalle por el usuario a partir de un parque real (25
inversores Huawei por bloque, 6 bloques, 6 STS Huawei Jupiter de
6000/5800 kVA, 3 ramas de media tensión con sus respectivos anillos de
fibra hacia la estación central).

Modelar un solo inversor no solo es menos realista: no deja demostrar
nada del trabajo real de un integrador OT/SCADA — agregación jerárquica,
direccionamiento de una flota grande, correlación de eventos entre
niveles.

## Decisión

**Escala real, sin escalar contenedores 1:1.** Se simulan los 150
inversores reales (25 por bloque × 6 bloques), pero un solo proceso por
bloque (`block_simulator.py`) sirve sus 25 inversores + su STS como un
único servidor Modbus TCP con **unit ID** por equipo (1-25 los
inversores, 100 el STS) — no 150 contenedores. Cada inversor sigue siendo
individualmente direccionable (`block-N/inv-NN`) y visible con
drill-down real en el dashboard.

**Direccionamiento jerárquico por device_id**: `block-3/inv-14`,
`block-3/sts`, `central/weather`, `central/substation`. El backend guarda
`device_id` + `device_kind` + `block_id` y agrega por cualquiera de los
tres.

**Un field-agent por bloque** (`field-agent-block-N`), matching el límite
físico real del RTU de un STS: sondea sus 25 inversores + el STS,
reporta por excepción por equipo, bufferiza en SQLite por equipo. Un
field-agent central hace lo mismo para meteorología + interconexión.

**Reloj de entorno compartido sin red entre procesos**
(`field_agent/environment.py`): en vez de que los 6 bloques + la central
se sincronicen entre sí (algo que un parque real tampoco hace — el sol no
manda un mensaje), cada proceso deriva el sol y el viento de una función
pura de `time.time()`. Misma fórmula, mismo reloj de pared → quedan de
acuerdo gratis, sin acoplarlos. Esto también permitió corregir un defecto
real del v1: la curva de irradiancia original (`sin(pi * fase)` sobre
todo el ciclo) casi no tenía noche real, solo un instante de irradiancia
baja en el borde del ciclo. Ahora el 70% del ciclo es luz de día
(curva completa amanecer→pico→atardecer) y el 30% restante es noche
sostenida — necesario además para el comportamiento de tracker que pidió
el usuario: el tracker vuelve a posición 0 (`RESETTING`) unos instantes
antes de perder la última luz, y permanece en 0 (`PARKED`) toda la noche,
retomando el seguimiento (`TRACKING`) recién al amanecer siguiente.
Una tormenta (`DEFENSE`, viento sobre umbral) puede interrumpir cualquiera
de los otros tres estados en cualquier momento del ciclo.

**STS de bloque simulado con transformador + protección**: temperatura de
aceite (función de la carga agregada de sus 25 inversores), alarma
Buchholz (evento raro, falla incipiente), relé de protección (dispara por
sobrecarga o por Buchholz). La interconexión central mide de forma
**físicamente independiente** de la suma de los bloques (tensión,
frecuencia, sincronismo, evento LVRT) — así es en la realidad: el punto
de medición de frontera no es una suma de telemetría, es una medición
eléctrica propia.

**Segundo comando real, no solo curtailment**: `set_reactive_power`
(-100..+100%), reflejando el soporte de reactiva que un operador de red
real puede pedirle a un parque con generación distribuida — mismo modelo
*pull* y mismo RBAC que el curtailment.

## Consecuencias

- El payload de telemetría deja de tener un schema fijo de inversor: pasa
  a ser un `dict` con campos comunes (`device_id`, `device_kind`,
  `block_id`, `timestamp`, `sequence`) + el resto libre, persistido como
  JSON (`TelemetryReading.payload`). El motor de alarmas
  (`alarms/rules.yaml`) pasa a tener una sección por `device_kind`.
- El dashboard pasa de un panel único a un drill-down real: planta (MW
  totales, meteorología, interconexión, 6 bloques) → bloque (STS + grilla
  de 25 inversores) → inversor individual (gauges, trend charts,
  comandos) — ver `dashboard/src/components/{PlantOverview,BlockView,InverterDetail}.tsx`.
- `docker-compose.yml` pasa de 8 a 18 servicios. Cada field-agent de
  bloque necesita su **propio** volumen de buffer SQLite — compartir uno
  entre agentes distintos pisa el archivo de otro (ver bugs reales más
  abajo, README principal).
- El detector de anomalías (z-score por dispositivo, sin cambios de
  diseño) expone una limitación real a esta escala: una tormenta activa la
  defensa de todos los trackers (que giran a la posición de defensa) y cambia a
  la vez las lecturas de los 150 equipos (mismo reloj de entorno, por diseño), así
  que dispara la misma anomalía de tensión DC en toda la flota
  simultáneamente — ruido, no 150 fallas independientes. Ver roadmap.

## Adenda: 40 MW reales y el STS como equipo de primera clase

Dos omisiones de la primera versión de este ADR, señaladas por el usuario:

**La planta no llegaba a los 40 MW.** Los inversores seguían la curva del
sol tal cual (`irradiancia × potencia nominal`) y el parque se quedaba en
~33 MW. Un parque real instala más potencia pico en paneles que la
potencia AC de sus inversores (relación DC/AC típica de 1.2 a 1.4), así que
al mediodía cada inversor *planchea* en su nominal. Se modela con
`DC_AC_RATIO = 1.3` y `SYSTEM_LOSSES = 0.95`, y la potencia nominal pasa a
ser `40 000 kW / 150 = 266.67 kW` por inversor (antes 266 kW). El plateau
a potencia nominal es visible en la curva de generación.

**El STS Jupiter no se veía.** Se lo modelaba como una caja con cinco
números. Ahora tiene sus tres partes reales:

- **Cámara de BT (800 V):** un interruptor por alimentador (uno por
  inversor, 25 por bloque), interruptor general de BT, y tensión de línea
  (R-S, S-T, T-R) y corriente por fase (R, S, T).
- **Transformador BT/MT:** temperatura de aceite y de devanado, relé
  Buchholz.
- **Cámara de MT (34.5 kV):** interruptor de MT y tensión/corriente por
  fase hacia la estación central.

Los interruptores no son decorativos, tienen consecuencias:

- El interruptor de un alimentador **dispara** cuando su inversor tiene una
  falla que en un parque real activa la protección (sobrecorriente AC,
  defecto a tierra, falla de aislamiento DC, arco DC) y **recierra** cuando
  la falla se despeja. Las fallas que el inversor maneja solo (ventilador,
  comunicación, red fuera de rango) no lo disparan.
- El relé del transformador (Buchholz o sobrecarga > 115 %) **dispara los
  interruptores de BT y de MT** y los rearma tras unos ticks con la causa
  despejada. Con cualquiera de los dos abiertos, los inversores del bloque
  pierden el camino de evacuación y dejan de entregar.
- Las corrientes salen de la potencia real del bloque
  (`P / (√3 · V · cos φ)`, con un leve desbalance entre fases), y valen
  cero si un interruptor está abierto.

**Rating de los STS.** Los 6000/5800 kVA que se recordaban del parque real
(34.6 MVA en total) no alcanzan para evacuar 40 MW en 6 bloques: cada
bloque entrega ~6.7 MW, que a cos φ ≈ 0.95 son ~7 MVA, y con 5800 kVA el
transformador operaría sobrecargado casi todo el día. Se usan 7000 kVA
(bloques 1-2) y 6800 kVA (3-6). Es un supuesto de la simulación, no un dato
del parque real.

En el dashboard, el STS aparece en el diagrama unifilar entre cada bloque y
la troncal de MT (con sus dos interruptores a la vista), y al abrir un
bloque se ve su unifilar BT → transformador → MT, la tabla por fase y la
tira de los 25 interruptores de alimentador.

## Adenda: hora simulada, 12,5 h de sol y tiempo acelerado

Dos omisiones señaladas por el usuario: en ningún lado se veía qué hora era en
la simulación, y el día tenía 70 % de luz (16,8 h), que no se parece a un día
real.

- **Horas de sol reales.** `DAYLIGHT_HOURS = 12.5`, con el amanecer a las 06:00
  y el ocaso a las 18:30 (latitud de Jujuy, cerca del equinoccio). El resto del
  día, 11,5 h, es noche real. La fracción de luz pasa de 0,70 a 0,52 del ciclo;
  el mediodía solar cae en la fase 0,26 (~12:15).
- **Tiempo acelerado, x144.** Sigue habiendo un día de 24 h cada 600 s de reloj:
  una hora simulada dura 25 s. La hora simulada es una función pura del reloj
  de pared (`sim_hour_of_day`), igual que el resto del entorno.
- **La base de tiempo es un dato de la estación meteorológica** (tres registros
  Modbus: segundos por día, hora de amanecer y horas de luz), como el maestro de
  hora de una planta real. El backend calcula la hora con esos tres valores
  (`app/core/sim_clock.py`, `GET /api/v1/sim/clock`) sin duplicar constantes, y
  cada alarma lleva su hora simulada (`sim_time`).
- **Dashboard:** reloj en el encabezado (ícono de sol o luna, HH:MM, barra de 24 h
  con el tramo de luz) que avanza localmente a x144 y se re-sincroniza cada 15 s.
