# Informe — Pág. 4–6: RPO y fallas (Persona C)

> Borrador para unir al PDF final. Todas las cifras salen de `evidence/` (pasos 10–13).

## Entorno de la ejecución

| Elemento | Valor |
| --- | --- |
| Máquina | Intel Core i7-6500U @ 2.50 GHz, 4 hilos, 11 GiB RAM |
| SO | Ubuntu 24.04.5 LTS, kernel 7.0.0-38-generic |
| Docker | 29.9.2, sin límites de CPU/RAM; `--wiredTigerCacheSizeGB 0.25` por nodo |
| MongoDB | **7.0.45** (ver nota) |

**Nota sobre la versión.** `mongo:8.0` no arranca en este kernel: `mongod` termina con
*"Linux kernel versions 6.19 and newer has a known incompatibility with this version of
MongoDB"* (SERVER-121912, TCMalloc y el ABI de `rseq`). Para los pasos 10–13 se usó un
`docker-compose.override.yml` local (no versionado) que cambia solo la imagen a `mongo:7.0`.
La replicación, las elecciones, `w:majority`, `electionTimeoutMillis` y el *step down* por
pérdida de mayoría funcionan igual en 7.0, así que las conclusiones se mantienen.

## TODO 5 — Cómo se clasifica el RPO

`calcular_rpo` solo usa operaciones entre conjuntos:

| Clave | Fórmula | Significado |
| --- | --- | --- |
| `perdidos` | `confirmados − presentes` | se confirmaron (`ok`) y no están → **RPO > 0** |
| `desconocidos_presentes` | `sin_confirmar ∩ presentes` | dieron error pero **sí** quedaron escritas |
| `desconocidos_ausentes` | `sin_confirmar − presentes` | dieron error y no quedaron |
| `inesperados` | `presentes − confirmados − sin_confirmar` | nunca se intentaron (error de la prueba) |

`presentes` se lee con `readConcern: "majority"`: solo cuenta lo que tiene una mayoría y que,
por tanto, no se puede revertir en un *rollback*. Autotest: 3/3 casos `OK`.

**Línea base** (`rpo-baseline.txt`): 20 intentos, 20 confirmados, 20 presentes, `PERDIDOS 0`.
Con esto el instrumento queda validado antes de provocar fallas.

## Resumen de las tres fallas

| Escenario | Nodo(s) caído(s) | Intentos | ok | error | Tipos de error | RTO | RPO (confirmadas) | Error pero escrita |
| --- | --- | ---: | ---: | ---: | --- | ---: | ---: | ---: |
| A · secundario | mongo-2 (`docker stop`) | 80 | 80 | 0 | — | 0 (sin corte) | 0 | 0 |
| B · primario | mongo-1 (`docker kill`) | 104 | 101 | 3 | `ServerSelectionTimeoutError` ×3 | **10,01 s** | 0 | 0 |
| B5 · primario, `electionTimeoutMillis=5000` | mongo-3 | 113 | 112 | 1 | `ServerSelectionTimeoutError` ×1 | **4,99 s** | 0 | 0 |
| C · sin mayoría | mongo-1 + mongo-2 | 109 | 95 | 14 | `ServerSelectionTimeoutError` ×10, `WTimeoutError` ×3, `WriteConcernError` ×1 | ≈34 s (*) | 0 | **4** (seq 14–17) |

(*) En C el primario vuelve cuando se reinician los nodos caídos, no por una elección
automática: ese "RTO" depende del operador (`sleep 20 + 10` del guion), no de MongoDB.

---

## Falla A — cae un secundario

**Precondición** (`falla-a-before.txt`): mongo-1 PRIMARY, mongo-2 y mongo-3 SECONDARY.
**Falla:** `docker stop --timeout 0 ti4601-mongo-2` → señal `1792817808.142715757`
(`2026-10-23T22:56:48.14-06:00`, `falla-a-stop.txt`).
**Durante** (`falla-a-status.txt`): mongo-1 PRIMARY, mongo-2 `not reachable/healthy`, mongo-3 SECONDARY.

Filas del CSV alrededor de la señal (`falla-a.csv`, columnas phase, seq, status, latency_ms, completed_epoch):

```text
antes-falla,11,ok,6.0,1792817807.421098
antes-falla,12,ok,3.9,1792817807.919221
despues-falla,13,ok,5.2,1792817808.420788   ← 278 ms después de la señal
despues-falla,14,ok,4.7,1792817808.920558
despues-falla,15,ok,6.9,1792817809.422984
```

| Fase | n | p50 | p99 | máx |
| --- | ---: | ---: | ---: | ---: |
| antes-falla | 13 | 5,3 ms | 14,6 ms | 14,6 ms |
| después-falla | 67 | 4,7 ms | 8,4 ms | 8,4 ms |

**Interpretación.** Ningún error y ningún cambio apreciable de latencia. Con `w:majority`, el
primario necesita su propio *ack* más el de **un** secundario (2 de 3). Con mongo-2 caído
quedan mongo-1 + mongo-3 = 2 de 3 votos: hay mayoría para mantener al primario y para
confirmar escrituras.

**Contra la predicción** (`evidence/predicciones.md`): se cumplió que no habría errores y que la
p50 sería parecida (5,3 → 4,7 ms). No se cumplió que la p99 subiría algo: bajó de 14,6 a 8,4 ms,
y la escritura en curso al detener el nodo tampoco tuvo pico (seq 13, 5,2 ms). Con tres nodos en
la misma máquina, los dos secundarios responden casi al mismo tiempo, así que esperar justo a
mongo-3 no cuesta más que esperar al más rápido de los dos.

**Al volver** (`falla-a-recovered.txt`), mongo-2 entra como SECONDARY: retoma la replicación
desde su último `optime` leyendo el oplog del primario. No necesita *initial sync* porque
estuvo fuera solo ~15 s, muy por debajo de la ventana del oplog. `falla-a-rpo.txt`: 80/80
presentes, **PERDIDOS 0**.

---

## Falla B — cae el primario

**Precondición** (`falla-b-before.txt`): mongo-1 PRIMARY, **`term=1`**, `electionTimeoutMillis=10000`.
**Falla:** `docker kill ti4601-mongo-1` → señal `1792817872.249831348`
(`2026-10-23T22:57:52.27-06:00`).
**Durante** (`falla-b-status.txt`, 15 s después): mongo-1 caído, mongo-2 SECONDARY,
**mongo-3 PRIMARY, `term=2`**.

Filas del CSV (`falla-b.csv`):

```text
antes-falla,11,ok,6.7,1792817871.591525      ← última confirmada antes de la falla
antes-falla,12,error,3009.0,1792817875.095516,ServerSelectionTimeoutError
despues-falla,13,error,3001.2,1792817878.097063,ServerSelectionTimeoutError
despues-falla,14,error,3000.9,1792817881.098176,ServerSelectionTimeoutError
despues-falla,15,ok,1161.6,1792817882.260059 ← primera confirmada tras la señal
despues-falla,16,ok,5.3,1792817882.265952
```

**RTO** = `1792817882.260059 − 1792817872.249831` = **10,01 s**. La ventana sin escrituras
confirmadas, medida entre seq 11 y seq 15, es de 10,67 s.

El seq 12 aparece como `antes-falla` porque la sonda lo envió cuando el `kill` ya estaba en
curso, pero la señal se escribe recién cuando `docker kill` devuelve (pocos cientos de ms después).

**Por qué RTO > `electionTimeoutMillis`.** Los secundarios no ven el *heartbeat* del primario
(cada 2 s) y esperan `electionTimeoutMillis` (10 s, con una pequeña variación aleatoria) antes de
convocar la elección. Luego viene la votación (un nuevo `term`). Después, el driver debe
descubrir al nuevo primario (`heartbeatFrequencyMS=1000` en la sonda). Por último, la
operación pendiente se reintenta: el seq 15 tardó 1161,6 ms porque se emitió durante la
elección y terminó en el nuevo primario.

**Errores y `retryWrites`.** `retryWrites=True` reintenta **una sola vez**, y únicamente si
encuentra un primario seleccionable dentro de `serverSelectionTimeoutMS=3000`. Durante unos
10 s no hubo primario, así que tres intentos seguidos agotaron los 3 s de selección y
salieron como `ServerSelectionTimeoutError`. El reintento solo oculta fallas cortas, no una
ventana mayor que el timeout de selección.

**`term` y el viejo primario.** `term` pasó de 1 a 2. Al volver, mongo-1 ve un `term` mayor
y entra como SECONDARY (`falla-b-recovered.txt`). No recupera el rol: todos tienen la misma
`priority` (1), mongo-3 está sano, y un nodo con `term` viejo no puede ganar una elección
ni aceptar escrituras.

**RPO** (`falla-b-rpo.txt`): 101 confirmadas, 101 presentes, **PERDIDOS 0**. Los 3 errores
no quedaron escritos. Con `w:majority`, cada `ok` ya estaba en 2 de 3 nodos y lo tenía al
menos uno de los que eligieron al nuevo primario.

### B5 — misma falla con `electionTimeoutMillis = 5000` (opcional)

Precondición (`falla-b5-before.txt`): mongo-3 PRIMARY, `term=3`. Tras `docker kill ti4601-mongo-3`,
el nuevo primario es mongo-2 con `term=4`. Al final se restauró `electionTimeoutMillis=10000`
(última línea de `falla-b5-recovered.txt`).

```text
antes-falla,12,ok,4.7,1792818096.629883
despues-falla,13,error,3095.6,1792818100.221038,ServerSelectionTimeoutError
despues-falla,14,ok,1593.0,1792818101.814277   ← primera confirmada
```

Señal `1792818096.827114` (el `kill` se emitió en `…096.661276`).
**RTO = 101.814277 − 096.827114 = 4,99 s**, frente a 10,01 s con el valor por defecto.
El RTO escala casi 1:1 con `electionTimeoutMillis`: es el término dominante. Bajarlo
reduce el corte, a cambio de más elecciones innecesarias si la red tiene picos de latencia.
RPO 0, 1 error no escrito.

---

## Falla C — dos secundarios caídos, primario vivo (sin mayoría)

**Precondición** (`falla-c-before.txt`): mongo-3 PRIMARY, mongo-1 y mongo-2 SECONDARY.
Sonda con `--w majority --wtimeout-ms 2000`.
**Falla:** `docker kill ti4601-mongo-1 ti4601-mongo-2` → señal `1792817968.942381`
(`2026-10-23T22:59:28.94-06:00`).
**Durante** (`falla-c-status.txt`, 20 s después): mongo-1 y mongo-2 caídos,
**mongo-3 SECONDARY**. No queda primario.

Filas del CSV (`falla-c.csv`):

```text
antes-falla,13,ok,6.6,1792817959.666162                       ← última confirmada
antes-falla,14,error,2377.4,1792817962.537133,WTimeoutError
antes-falla,15,error,3290.6,1792817965.827976,WTimeoutError
antes-falla,16,error,2260.1,1792817968.088148,WTimeoutError
antes-falla,17,error,4583.9,1792817972.672220,WriteConcernError
despues-falla,18,error,3002.7,1792817975.675176,ServerSelectionTimeoutError
   … seq 19–27: ServerSelectionTimeoutError (~3000 ms cada uno) …
despues-falla,27,error,3003.7,1792818002.705004,ServerSelectionTimeoutError
despues-falla,28,ok,213.2,1792818002.918506                   ← primera confirmada
```

Seq 14–17 están marcados `antes-falla` porque `docker kill` de dos contenedores tardó varios
segundos en devolver: los nodos ya estaban caídos desde ~`…960`, pero la señal se escribió
en `…968.94`. Por los timestamps y el tipo de error, esas filas pertenecen a la falla.

**Por qué el sobreviviente baja a SECONDARY.** Un primario debe ver a una mayoría de votantes
(2 de 3). Al dejar de recibir *heartbeats* de los otros dos, mongo-3 hace *step down*
pasado el `electionTimeoutMillis`, aunque él mismo siga vivo. Esto evita el *split-brain*:
si quedara primario aislado podría aceptar escrituras que otra mayoría nunca vería. No es
una elección: es el primario renunciando por no ver la mayoría. Y como un solo nodo no
puede ganar ninguna elección, nadie más asume.

**Dos fases de error:**

1. **`WTimeoutError` / `WriteConcernError` (seq 14–17).** mongo-3 todavía era primario y
   **aplicó** la escritura localmente, pero no consiguió el *ack* de un segundo nodo dentro de
   `wtimeout=2000 ms`. El seq 17 terminó como `WriteConcernError` cuando el primario hizo
   *step down* con la operación esperando.
2. **`ServerSelectionTimeoutError` (seq 18–27).** Ya no hay primario: el driver no encuentra a
   quién mandar la escritura y nada llega al servidor.

**"Error ≠ no escrito"** (`falla-c-rpo.txt`):

```text
intentos 109 · confirmados 95 · con error 14 · presentes 99
PERDIDOS 0
errores que SÍ se escribieron ........... 4  [14, 15, 16, 17]
errores que NO se escribieron ........... 10
```

Los 4 `WTimeoutError`/`WriteConcernError` **sí están** en la base, leídos con
`readConcern: majority`: cuando volvieron mongo-1 y mongo-2, replicaron esas entradas del
oplog de mongo-3 y quedaron mayoritarias. Los 10 `ServerSelectionTimeoutError` no llegaron
nunca a un servidor. **Ninguna escritura se confirmó sin mayoría**, y el RPO sobre lo
confirmado es 0.

**Efecto en un cliente que compra.** Si `comprar_entradas` recibe un `WTimeoutError` y el
cliente concluye "no se compró" y reintenta con un **nuevo** `orden_id`, el descuento puede
quedar duplicado: el primero sí se aplicó. Por eso:

* el **RPO se mide sobre lo confirmado**. Lo que dio error tiene resultado desconocido y no se
  puede contar como perdido ni como guardado sin consultar la base;
* el P2 exige operaciones **idempotentes**: el reintento debe usar el **mismo** `orden_id`
  (`_id` único → `DuplicateKeyError` → se devuelve el estado existente), para que repetir
  tras un error ambiguo no descuente dos veces.

**Comparación con Falla B del Lab 1** (CockroachDB, Raft). En ambos sistemas, sin quórum
no hay escrituras confirmadas (CP en CAP). En Raft el *leaseholder* aislado tampoco puede
confirmar y la petición queda colgada o con timeout. En MongoDB el primario aplica
localmente y devuelve `WTimeoutError`, lo que deja escrituras "huérfanas" que luego se
replican (o se revierten con *rollback* si otro nodo se hubiera elegido). Desde el cliente,
los dos casos son ambiguos.

**Recuperación** (`falla-c-recovered.txt`): al reiniciar mongo-1 y mongo-2 vuelve la
mayoría, mongo-3 gana la elección y queda 1 PRIMARY + 2 SECONDARY.

---

## Mayoría 2 de 3 — resumen

| Escenario | Votos vivos | ¿Mayoría (≥2)? | ¿Primario? | ¿Escrituras `w:majority`? |
| --- | ---: | :---: | :---: | :---: |
| Normal | 3 | sí | sí | sí |
| A (1 secundario caído) | 2 | sí | sí, el mismo | sí, sin corte |
| B (primario caído) | 2 | sí | sí, tras elección (~10 s) | sí, después del RTO |
| C (2 caídos) | 1 | **no** | **no** (*step down*) | **no** |

Con 3 nodos se tolera **una** falla. Un cuarto nodo no aumentaría la tolerancia: la mayoría
de 4 es 3, y se seguiría tolerando una sola falla. De ahí el número impar de votantes.
