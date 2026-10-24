# Predicciones del Laboratorio 3

Se escriben antes de ejecutar cada paso y no se editan después. Van al informe.

## Paso 4: semilla y carga

Fabricio Mena · 22/10/2026, 14:24 · antes de correr `generar_datos.py` y `cargar.py`

- Cada documento de `eventos` va a quedar en los **3 nodos**: se escribe en el primario y los dos
  secundarios lo copian del oplog.
- Antes de que `insert_many` responda tienen que tenerlo **2 nodos** (el primario y al menos un
  secundario), porque la conexión usa `w: "majority"` y la mayoría de 3 votantes es 2.
- Con 400 eventos y lotes de 500, la carga entera es un solo `insert_many`, así que se espera a la
  mayoría una sola vez.

## Paso 7: lecturas en secundario

*Pendiente (Persona B).*

## Paso 9: costo del write concern

*Pendiente (Persona B).*

## Pasos 11 a 13: fallas

Christopher Rodriguez · predicción a partir de la teoría (mayoría 2/3, `electionTimeoutMillis`,
`w: "majority"`), sin usar los resultados de las corridas

### Paso 11 · Falla A: cae un secundario

- **¿Errores?** No. Quedan el primario y un secundario: 2 de 3 votos, que es mayoría. El primario
  sigue siendo primario y las escrituras `w: "majority"` se siguen confirmando.
- **¿Latencia?** p50 parecida. Antes bastaba con el *ack* del secundario más rápido; ahora hay que
  esperar justo al que queda, así que se espera algo más de variación (p99 un poco mayor). También
  puede haber un pico en la escritura que estaba en curso cuando se detuvo el nodo.
- **¿Nodos para la mayoría?** Quedan 2 y la mayoría es 2: se cumple, pero **sin margen**. Si cayera
  otro nodo se perdería la mayoría.
- **RPO esperado:** 0. Al volver, el secundario entra como `SECONDARY` y se pone al día leyendo el
  oplog, sin *initial sync*, porque estuvo fuera pocos segundos.

### Paso 12 · Falla B: cae el primario

- **¿Cuántos segundos sin primario?** Unos **11 a 13 s**. Los secundarios esperan
  `electionTimeoutMillis` (10 s) sin *heartbeats* antes de convocar la elección. A eso se suma la
  votación (≈1 s) y que el driver descubra al nuevo primario (≈1 s con `heartbeatFrequencyMS=1000`).
- **¿De qué parámetro depende?** Sobre todo de `electionTimeoutMillis`, y en menor medida de
  `heartbeatIntervalMillis` y del monitoreo del driver. Con `electionTimeoutMillis = 5000` se espera
  un corte de unos **6 a 8 s**.
- **¿Qué verá la sonda?** Una racha de errores `ServerSelectionTimeoutError`, de unos 3 s cada uno
  (`serverSelectionTimeoutMS=3000`): unos 3 o 4 seguidos. `retryWrites` no los va a ocultar, porque
  reintenta una sola vez y el corte dura más que el tiempo de selección. Después, una primera
  escritura lenta y luego latencias normales.
- **`term`:** sube en 1. El viejo primario vuelve como `SECONDARY`: ve un `term` mayor y ya hay un
  primario sano con la misma prioridad.
- **RPO esperado:** 0 sobre lo confirmado. Cada `ok` con `w: "majority"` ya estaba en 2 nodos y al
  menos uno de ellos participa en la elección.

### Paso 13 · Falla C: caen los dos secundarios, el primario sigue vivo

- **¿Habrá primario?** **No.** El nodo sobreviviente solo ve 1 de 3 votos, así que, pasado
  `electionTimeoutMillis`, hace *step down* y queda como `SECONDARY` aunque esté sano. Así se evita
  que un primario aislado acepte escrituras que la mayoría no ve.
- **¿Qué pasa con cada insert `w: "majority"`?** Hay dos fases:
  1. Mientras el nodo todavía es primario (los primeros segundos), aplica la escritura **localmente**
     pero no consigue el segundo *ack*: después de `wtimeout` (2 s) devuelve `WTimeoutError`. El
     resultado es **desconocido**: el dato está en el primario aunque el cliente reciba error.
  2. Después del *step down* ya no hay primario: `ServerSelectionTimeoutError` cada ~3 s, y esas
     escrituras no llegan a ningún nodo.
  
  No se espera **ninguna** escritura confirmada hasta que vuelvan los secundarios. Al volver, se
  espera que las escrituras de la fase 1 se repliquen y aparezcan en la base: errores que sí se
  escribieron (`desconocidos_presentes` > 0). RPO sobre lo confirmado: 0.
- **Comparación con la Falla B del Lab 1 (CockroachDB):** en los dos casos, sin quórum no se
  confirma nada; los dos priorizan la consistencia (CP). La diferencia esperada es que en Raft el
  líder no aplica la escritura hasta que el quórum la acepta, así que la petición queda bloqueada o
  da timeout. MongoDB, en cambio, aplica primero en el primario y espera la mayoría después, y eso
  deja escrituras ambiguas: "error" no significa "no escrito".
