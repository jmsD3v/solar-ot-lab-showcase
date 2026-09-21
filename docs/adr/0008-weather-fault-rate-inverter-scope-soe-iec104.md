# ADR 0008 — Clima real, fallas ajustables, un inversor que solo informa lo suyo, SOE de bloque e IEC 104

## Estado
Aceptado (2026-09-20).

## Contexto
Cinco cosas del simulador no se parecían a un parque real, y las cinco se notaban en el dashboard:

1. **El cielo era siempre despejado.** Lo único que bajaba la generación era la ventana de tormenta
   con viento. No había días nublados ni de lluvia, así que ni los KPI de desempeño ni el detector
   de rampas se ejercitaban con lo más común en la vida real: nubes que pasan.
2. **Fallar era lo normal.** En una sesión de 15 minutos aparecían disparos de protección, pérdidas
   de sincronismo y eventos de baja tensión. Está bien para una demo de alarmas, pero no hay forma
   de ver el parque "funcionando bien".
3. **El registro del inversor llevaba irradiancia y temperatura de módulo.** Un inversor no mide
   eso: lo mide la estación meteorológica (POA y sensor trasero de temperatura). Era el mismo tipo
   de error que se corrigió con los trackers (ADR 0007).
4. **La secuencia de eventos (SOE) solo cubría la estación transformadora.** Un disparo de
   protección del STS o un motor trabado en un tracker quedaban con la hora en que el agente los leyó.
5. **El enlace con el despacho era un dibujo:** un estado "activo/caído" sin protocolo detrás.

## Decisión

### Clima (`environment.py`)
El cielo de cada día simulado es **despejado, parcialmente nublado, cubierto o con lluvia**, elegido
por una función determinista del número de día (mismo principio que el sol, ADR 0004: cada proceso
llega al mismo valor sin hablar con los demás). El primer día es siempre despejado.

- `cloud_factor()` es la fracción de luz de cielo despejado que llega al suelo. Parcial: nubes que
  pasan (mezcla de ondas lentas con umbral, cae hasta ~0,28). Cubierto: ~0,3. Lluvia: ~0,16, y llueve.
- **Las nubes cruzan el parque**: cada bloque las ve con un retraso (2,5 s de reloj por bloque, más
  una pizca por inversor), como un frente que avanza. La estación meteorológica va primera, así que el
  contexto de planta (ADR 0005) ya vio la rampa cuando los inversores empiezan a moverse.
- Bajo nubes también cambia el resto de la estación: ambiente más frío, más humedad, POA sin ganancia
  de tracker (la luz es difusa), y la lluvia de los días de lluvia se suma a la de las tormentas.
- El dashboard muestra el estado del cielo (índice de claridad = irradiancia medida / la de cielo
  despejado a esa hora), sin necesitar un dato nuevo del equipo.
- `SIM_SKY=clear|partly|overcast|rainy` fija el cielo; `auto` (por defecto) lo deja variar.

### Fallas ajustables
`SIM_FAULT_RATE` multiplica **todas** las fallas aleatorias (inversores, STS, estación, trackers,
sensores): 1 = ritmo de demo, 0,3 = parque tranquilo, 0 = todo perfecto. Las fallas provocadas a mano
(comandos, escenarios de incidente) no se afectan. No cambia el valor por defecto: cambia que se pueda
elegir.

### El inversor informa solo lo suyo
Se quitan del inversor `module_temp_c` e `irradiance_wm2` (registros 3 y 4 quedan reservados). Su
temperatura propia es `internal_temp_c`, que ya existía. La irradiancia y la temperatura de módulo
salen de la **estación meteorológica** (`irradiance_wm2`, `poa_irradiance_wm2`, `module_temp_ref_c`).
Consecuencias:

- La falla de instrumentación (cable cortado, piranómetro muerto, lee fondo de escala) pasa a la
  estación meteorológica. El agente la marca `quality: bad` y el backend no genera alarmas de proceso
  con ese dato; y **un piranómetro roto no cuenta como rampa de planta** en el contexto que suprime
  alarmas simultáneas (si no, un dato basura silenciaría alarmas reales).
- Las reglas del inversor pasan a vigilar lo suyo (`internal_temp_c`, `ac_power_kw`, frecuencia,
  aislamiento, desbalance MPPT); la de temperatura de módulo se mueve a la estación.
- El detalle del inversor muestra corriente DC y temperatura interna en lugar de irradiancia y
  temperatura de módulo.

### SOE de bloque y de trackers (`soe.py`)
Un registrador reutilizable (`SoeRecorder`) y su formato de registros (ranuras de 6 registros con
número, código, hora del equipo con milisegundos y estado). Se usa en:

- **STS** (8 ranuras): relé Buchholz, válvula de sobrepresión, disparo de la protección, interruptor
  general de BT, interruptor de MT y cada uno de los 25 alimentadores de BT. Orden causa → consecuencia
  (4 ms entre señales de una misma pasada).
- **NCU** (8 ranuras, a continuación de la tabla de 75 trackers: 606 registros en total, 6 tramos de
  lectura): cambio de modo, defensa por viento, y por tracker motor trabado y pérdida de radio.
  La batería baja **no** entra: es una condición que aparece en los 75 a la vez de noche, no una
  secuencia, y saturaría el buffer.

El backend ya guardaba `soe_events` de cualquier equipo; el dashboard suma etiquetas legibles y un
filtro por origen (estación, bloques, trackers). La estación mantiene su propia implementación.

### IEC 60870-5-104 (`iec104.py`)
El enlace con el despacho de la red usa IEC 104 sobre TCP/2404, como en un parque real. Implementación
mínima, real y sin dependencias:

- **Estación controlada** (en el simulador central): tramas I/S/U, STARTDT/STOPDT/TESTFR,
  interrogación general (C_IC_NA_1), medidas en coma flotante (M_ME_NC_1) y señales simples
  (M_SP_NA_1), **envío espontáneo** con banda muerta por punto. 8 puntos (P, Q, tensión, frecuencia,
  factor de potencia, interruptor 52L, interruptor 52M, sincronismo) tomados de la estación.
- **Estación de control** (en el agente central): se conecta, hace STARTDT y la interrogación general,
  y reporta lo que el despacho ve como equipo `central/dispatch`.
- El evento simulado `operator_link` de la estación ahora **corta el enlace de verdad**: cierra las
  conexiones y rechaza las nuevas. El agente pierde la conexión y deja de recibir datos del despacho;
  el panel del despacho dice "CAÍDO" enseguida (por el estado del enlace que informa la estación) y,
  si el corte supera el umbral del monitor de comunicación (90 s), el equipo `central/dispatch` genera
  su alarma de "sin comunicación". Al volver el enlace, el agente reconecta y recibe todo de nuevo.

## Consecuencias
- Los tests fijaron cada pieza (clima, perilla de fallas, calidad del piranómetro, registrador SOE,
  códec y enlace 104 de punta a punta). El SOE del NCU y el enlace IEC 104 se verificaron además en
  el stack levantado; MQTT con TLS 1.3 también.
- **No implementado a propósito:** comandos por 104 (consignas del despacho, C_SE_NC_1), sincronización
  de reloj 104, ventanas k/w de control de flujo, secuencias (SQ=1) y SOE por inversor.
- **El clima es una función del reloj, no un pronóstico:** no hay modelo de nubes espacial real (un
  frente lineal con retraso por bloque), ni humedad/temperatura correlacionadas más allá de lo dicho.
- Casi 3 de cada 10 días son cubiertos o de lluvia, así que la energía diaria varía mucho; los KPI de
  desempeño (PR, factor de capacidad) ahora se mueven con el cielo, como en un parque real.
