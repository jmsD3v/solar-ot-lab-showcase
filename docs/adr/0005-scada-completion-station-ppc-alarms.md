# ADR 0005 — SCADA completo: alarmas con ciclo de vida, estación transformadora acoplada, controlador de planta y medición fina

## Estado
Aceptado

## Contexto
Una revisión del laboratorio
mostró que el parque medía poco y que el "SCADA" tenía huecos que un operador real
notaría enseguida: 600.000 alarmas sin ningún mecanismo para reconocerlas, ningún
aviso cuando un equipo dejaba de reportar, una interconexión de 6 números calculada
de una curva de irradiancia (sin relación con lo que entregaban los inversores), y
ninguna capa de control entre el operador y los 150 inversores.

## Decisión

### Alarmas con ciclo de vida (ISA-18.2)
Una fila por *episodio*: nace una vez, se actualiza mientras la condición persiste
(con límite de escritura), pasa a *normalizada* cuando vuelve a normal, y se cierra
al reconocerse. Estados: activa sin reconocer → activa reconocida → normalizada sin
reconocer → cerrada. Los eventos puntuales (salto brusco, anomalía) se cierran solos
si no se repiten. Reconocer lo pueden hacer los dos roles (es operación, no control);
cada reconocimiento queda en el log de seguridad. Migración idempotente: las 600.000
filas del esquema anterior se cerraron.

### Calidad del dato y pérdida de comunicación
El field-agent valida cada magnitud contra su rango físico y adjunta `quality`/`bad_tags`;
el backend no evalúa alarmas de proceso sobre magnitudes malas y levanta una alarma
de calidad. Un monitor barre `device_status` (una fila por equipo) y marca la comunicación
como perdida a los 90 s (3 latidos); si calla casi todo un bloque, es una sola alarma
`agent_down`. Las APIs devuelven `comm_ok`/`age_s`: un dato viejo nunca se muestra como vivo.

### Medición ampliada, orientada a datos
`registers.py` define cada magnitud como `Field(nombre, registro, escala, offset)` y sirve
para codificar (simulador) y decodificar (agente): agregar una magnitud es una línea.
Inversor: tensión y corriente por fase, Q/S/cos φ, 9 MPPT, aislamiento, eficiencia, energía
del día, motivo de limitación. Tracker: controlador con objetivo, motor, batería, finales
de carrera. Meteorología: POA, lluvia, ensuciamiento, presión (~1.000 hPa a ~100 m), ráfaga.
STS: reactiva, nivel/presión de aceite, ventilación.

### Estación transformadora acoplada a los bloques
La estación 132/34,5 kV deja de inventar su potencia: **sondea por Modbus los STS** y suma lo
que entregan por sus tres colectores de MT (los tres ramales), así el medidor de frontera
cierra con la suma de los inversores menos las pérdidas del transformador. Modela la bahía
de línea (52L con protecciones 21, 50/51, 27/59, 81 y reenganche 79), el transformador
principal (regulación automática por cambiador de tomas, Buchholz, 87T, DGA en línea),
celdas de MT, compensación de reactiva, servicios auxiliares y comunicaciones. **El
acoplamiento es en los dos sentidos**: cada bloque consulta si la estación le permite
evacuar, así que un 52L disparado corta de verdad a los inversores.

### Controlador de planta (PPC)
Contenedor propio en la zona OT. Recibe consignas *de planta* (límite de MW, Q en Mvar o
factor de potencia, rampa) y las reparte entre los inversores, corrigiéndolas en lazo cerrado
contra el medidor del punto de interconexión: límite de rampa, anti-windup, inversores en
manual descontados del reparto. Un comando manual a un inversor lo saca del control
automático hasta que se lo libere (`set_control_mode`). Se comanda desde el backend con el
mismo modelo *pull* y control de acceso que el resto, y cada consigna queda auditada.

### Secuencia de eventos con hora del equipo
La estación registra cada cambio de estado en un buffer con su **propia hora, en
milisegundos**, ordenando causa → consecuencia (la protección antes que el interruptor).
El agente los reenvía una sola vez; el backend los guarda deduplicados y el dashboard los
muestra en orden real. (Solo la estación en esta etapa; el STS y los inversores quedan para después.)

### Retención, agregación e indicadores
Agregado por minuto (promedio/mín/máx) de las magnitudes clave; las lecturas crudas se purgan
a las 24 h (solo las ya agregadas, y nunca la última de cada equipo), y también las alarmas
cerradas, eventos de seguridad y SOE antiguos. Los indicadores (energía exportada, Performance
Ratio, factor de capacidad, rendimiento específico, disponibilidad) usan el mismo tiempo simulado
que los contadores de energía.

### Lectura del estado actual sin escanear el historial
`device_status` guarda el id de la última lectura de cada equipo, y el resumen de planta
y el detalle de bloque leen esas ~160 filas puntuales. La primera versión agrupaba toda la
tabla de lecturas en cada refresco del dashboard: con 2 millones de filas eso eran ~2 s por
pedido y saturaba el backend. También hay un índice `(device_id, ts)` para el historial de
un equipo (de ~1 s a decenas de ms). La agregación por minuto trabaja en tramos de 10 min
para acotar la memoria.

### Contexto de planta y desvío mínimo
Los saltos bruscos y el z-score se evaluaban equipo por equipo, y una transición que afecta a
toda la planta (el sol al amanecer y al atardecer, los trackers girando a su posición de defensa)
los disparaba en los 150 equipos. El inversor no mueve nada, solo convierte CC en CA: lo que gira
es el tracker. En este modelo cada equipo es "inversor + tracker" en un solo registro, por
simplicidad, y sus lecturas reflejan ambos.
`alarms/context.py` reconoce esas transiciones desde la estación meteorológica (pendiente de
irradiancia >= 3 W/m² por segundo, o señal de defensa, con 45 s de margen para que los inversores
sigan el cambio). Mientras dura, los eventos por equipo se suprimen y queda un único aviso
`plant_transition` (informativo). No suprime umbrales ni fallas de equipo. El detector estadístico
además exige un desvío mínimo por magnitud. Cada lectura, que llega por HTTP y por MQTT, se descarta
antes de cualquier consulta si ya se guardó por el otro canal.

## Consecuencias

- Los contadores de energía y las integraciones usan **tiempo simulado** (×144): 0,08 h por tick.
- La estación necesita ver a los bloques y los bloques a la estación: dependencia mutua, tolerante
  a que el otro extremo no esté (asume "sin potencia" o "puede evacuar" respectivamente).
- **Fuera de alcance y dicho con claridad**: no hay interfaz IEC 60870-5-104 hacia el despacho
  (el estado del enlace se simula, no se implementa), no hay sincronización horaria real
  (NTP/PTP) ni redundancia de servidores; el modelo de protecciones es funcional, no un
  relé con curvas. El modelado profundo de protecciones e IEC 61850 corresponde al proyecto
  separado de subestación.
