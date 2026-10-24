# TODO — Laboratorio 3: Conjuntos de réplicas en MongoDB

> Guía oficial: [labs/lab3-mongo-replicas/README.md](labs/lab3-mongo-replicas/README.md)
> **Entrega:** martes 3 de noviembre de 2026, 11:00 pm — PDF (máx. 6 páginas) en TEC Digital + enlace al repo.
> La [Tarea 3](labs/lab3-mongo-replicas/TAREA.md) (mismo dominio, 1 nodo) vence **antes**: miércoles 28 de octubre.


---

## Reparto del trabajo

> Rellene los nombres. Cada sección de pasos más abajo indica su responsable con 👤.

| Persona | Nombre | Bloque | TODO de código | Informe |
| ------- | ------ | ------ | -------------- | ------- |
| **A** | Fabricio Mena | Pasos -1 a 6 (repo, clúster, datos, índices, oplog) | TODO 1 y 2 (`cargar.py`) | Pág. 1–2 (excepto lecturas) + **armar el PDF final** |
| **B** | _______ | Pasos 7 a 9 (lecturas, compra, write concern) | TODO 3 (`comprar.py`) y TODO 4 (`bench_write_concern.py`) | Lecturas + CAP/PACELC (pág. 2) y pág. 3 completa |
| **C** | _______ | Pasos 10 a 13 (RPO y fallas A, B, C) | TODO 5 (`verificar_rpo.py`) | Pág. 4–6 (fallas) |


### Checklist — Persona A
- [x] Paso 0 · Requisitos + `make build` + `evidence/comandos.md`
- [x] Paso 1 · Leer Compose y levantar → `cluster-start.txt`
- [x] Paso 2 · `rs.initiate()` → 1 PRIMARY + 2 SECONDARY
- [x] Paso 3 · Inspeccionar → `rs-inspect.txt`
- [x] Paso 4 · Semilla → `datos-manifest.txt`
- [x] Paso 5 · **TODO 1 y 2** → `config-check.txt` (6/6), `explain-indices.txt` · 💾 commit
- [x] Paso 6 · Oplog → `oplog.txt` · 💾 commit
- [x] Informe pág. 1–2: diagrama, `rs.initiate()`, `rs.status`/`rs.conf`, explain, oplog
      En `labs/lab3-mongo-replicas/REPORTE.tex`; la tabla de lecturas del paso 7 la agrega B.
- [ ] Unir el PDF + script "FALTA:" + lista de control final
      Pendiente: necesita la evidencia y las secciones de B (pasos 7 a 9) y C (pasos 10 a 13).

### Checklist — Persona B

- [x] Pasos 0–5 en su máquina (clúster propio, sin subir evidencia de esos pasos)
- [x] Paso 7 · Lecturas en secundario → `lectura-secundario.{txt,csv}` · 💾 commit
- [x] Paso 8 · **TODO 3** compra atómica → `comprar.txt` + `comprar-roto.txt` · 💾 commit
- [x] Paso 9 · **TODO 4** write concern → `write-concern.{txt,csv}` (1201 líneas) · 💾 commit
- [x] Informe: tabla de lecturas + CAP/PACELC; pág. 3 (decisiones TODO 1–5, versión rota, latencias + máquina)

### Checklist — Persona C

- [ ] Pasos 0–5 en su máquina (clúster propio, sin subir evidencia de esos pasos)
- [ ] Paso 10 · **TODO 5** RPO + línea base → `rpo-baseline.{csv,txt}` · 💾 commit
- [ ] Paso 11 · Falla A (secundario) → `falla-a*` · 💾 commit
- [ ] Paso 12 · Falla B (primario) → `falla-b*` (+ opcional `falla-b5.*`) · 💾 commit
- [ ] Paso 13 · Falla C (sin mayoría) → `falla-c*` · 💾 commit
- [ ] Informe pág. 4–6: RTO/RPO con filas del CSV, mayoría 2/3, "error ≠ no escrito"

---

## Paso -1 — Preparar esta carpeta como repo propio · 👤 A

Esta carpeta es copia del repo del curso **sin `.git`**. Para la entrega hay que tener un repo con
**commits de los tres integrantes** y **no todos el último día** (si no: "Configuración propia" = 0).

**HOST**

```bash
cd ~/Escritorio/Tec/BasesDeDatosAvanzados/Lab-3-Bases-de-Datos-Avanzadas

# 1) La carpeta evidence/ trae archivos del Lab 1 con nombres que el Lab 3 reutiliza
#    (cluster-start.txt, config-check.txt, comandos.md). Muévalos para no mezclarlos:
mkdir -p evidence/lab1
mv evidence/*.* evidence/lab1/ 2>/dev/null
ls evidence evidence/lab1
```

> ⚠️ **`.env.example` tiene la licencia de CockroachDB pegada** (`COCKROACH_LICENSE=crl-...`).
> `.env` está en `.gitignore`, pero `.env.example` **no**. Antes del primer commit, deje
> `.env.example` con la línea comentada (`# COCKROACH_LICENSE=`) o no lo suba.

```bash
git init
git add .
git status            # revise que NO aparezca .env ni la licencia
git commit -m "Base del Laboratorio 3"
# crear repo en GitHub (privado) y conectar:
# gh repo create Lab-3-Bases-de-Datos-Avanzadas --private --source=. --push
```

Cuando el repo esté en GitHub, invite a B y C como colaboradores. Reparto y calendario: ver [Reparto del trabajo](#reparto-del-trabajo).

---

## Paso 0 — Requisitos y build · 👤 A

**HOST**

```bash
docker compose version
make lab1-down; make lab2-down        # liberar RAM si estaban levantados
make build                            # la imagen ahora incluye pymongo
mkdir -p evidence
test -f evidence/comandos.md \
  || printf '# Registro de comandos — Laboratorio 3\n\n' > evidence/comandos.md
```

Ayuda para llevar el registro (péguelo en cada terminal HOST nueva):

```bash
log() { printf -- '- %s · Paso %s · `%s` · %s\n' "$(date '+%F %T')" "$1" "$2" "${3:-OK}" >> evidence/comandos.md; }
# ejemplo:  log 0 "make build" OK
```

Anote también en el informe: **máquina, CPU, RAM, SO** (`lscpu | head; free -h; lsb_release -a`).

---

## Paso 1 — Leer Compose y levantar · 👤 A

Tabla para el informe (sale de `docker-compose.yml`):

| Servicio  | `--replSet` | `hostname` | volumen  | ¿publica puertos? |
| --------- | ----------- | ---------- | -------- | ----------------- |
| `mongo-1` | `rs-lab3`   | `mongo-1`  | `mongo1` | no                |
| `mongo-2` | `rs-lab3`   | `mongo-2`  | `mongo2` | no                |
| `mongo-3` | `rs-lab3`   | `mongo-3`  | `mongo3` | no                |

**HOST**

```bash
date --iso-8601=seconds | tee evidence/cluster-start.txt
make lab3-up
make lab3-status | tee -a evidence/cluster-start.txt
log 1 "make lab3-up / lab3-status" OK
```

Esperado: 3 contenedores corriendo y `conjunto no inicializado ... NotYetInitialized`.
Si dice `starting`, espere 10–20 s y repita `make lab3-status`.

Responder en el informe:
- ¿Por qué 3 `mongod` con el mismo `--replSet` aún no forman un conjunto? (falta `rs.initiate`)
- ¿Por qué el `_id` de la config debe ser `rs-lab3`?
- ¿Por qué `mongo-1:27017` y no `localhost:27017`? (DNS de la red Compose; cada nodo debe alcanzar a los otros)
- ¿Qué se pierde si se borran los volúmenes? (datos + configuración del conjunto)

---

## Paso 2 — `rs.initiate()` · 👤 A

**HOST** → `make lab3-shell`

**MONGOSH**

```javascript
rs.status()     // debe decir NotYetInitialized

rs.initiate({
  _id: "rs-lab3",
  members: [
    { _id: 0, host: "mongo-1:27017" },
    { _id: 1, host: "mongo-2:27017" },
    { _id: 2, host: "mongo-3:27017" }
  ]
})

// esperar 5–15 s y repetir:
rs.status().members.map(m => ({ name: m.name, state: m.stateStr, health: m.health }))
```

Esperado: `{ ok: 1 }`, luego 1 `PRIMARY` + 2 `SECONDARY`, prompt `rs-lab3 [direct: primary] test>`.
**Copie este `rs.initiate()` al informe** y explique cada campo.

Informe: ¿por qué número impar de votantes? ¿Qué campos completó MongoDB solo (ver Paso 3)? ¿Qué pasaría con `localhost`?

```bash
log 2 "rs.initiate rs-lab3" OK
```

---

## Paso 3 — Inspeccionar el conjunto · 👤 A

**MONGOSH**

```javascript
rs.status().members.map(m => ({ name: m.name, state: m.stateStr, optime: m.optimeDate, syncSource: m.syncSourceHost }))
rs.status().term
rs.conf().settings
rs.conf().members.map(m => ({ host: m.host, votes: m.votes, priority: m.priority }))
db.hello()
exit
```

**HOST**

```bash
docker exec ti4601-mongo-1 mongosh --quiet --eval '
printjson(rs.status().members.map(m => ({ name: m.name, state: m.stateStr, health: m.health, syncSource: m.syncSourceHost })));
printjson({ term: rs.status().term });
printjson(rs.conf().settings);
printjson(rs.conf().members.map(m => ({ host: m.host, votes: m.votes, priority: m.priority })));
' | tee evidence/rs-inspect.txt
log 3 "rs-inspect" OK
```

Anote: primario, `syncSourceHost` de cada secundario, `term` (guárdelo, se compara en Falla B),
`electionTimeoutMillis` (10000), `heartbeatIntervalMillis` (2000), `votes`, `priority`.

---

## Paso 4 — Generar la semilla · 👤 A

📝 Predicción en el informe: ¿en cuántos nodos queda cada documento y cuántos deben tenerlo antes de que `insert_many` responda? (3 / 2 con `w:majority`)

**APP** (`make lab3-app`)

```bash
python3 labs/lab3-mongo-replicas/generar_datos.py | tee evidence/datos-manifest.txt
```

Esperado: `semilla: 2026`, `eventos: 400`, `zonas`, `capacidad_total`, `sha256_eventos_jsonl`.
Compare el hash con otro equipo.

---

## Paso 5 — TODO 1 y TODO 2 (`cargar.py`) · 👤 A

Archivo: [labs/lab3-mongo-replicas/cargar.py](labs/lab3-mongo-replicas/cargar.py) — reemplazar los `raise NotImplementedError`.

**TODO 1 — `cargar_eventos`** (lotes con `insert_many`, `ordered=False`, conservar `_id`):

```python
    total = 0
    buffer: list[dict] = []
    with ruta.open(encoding="utf-8") as f:
        for linea in f:
            if not linea.strip():
                continue
            buffer.append(json.loads(linea))
            if len(buffer) >= lote:
                total += len(db.eventos.insert_many(buffer, ordered=False).inserted_ids)
                buffer = []
    if buffer:  # último lote incompleto
        total += len(db.eventos.insert_many(buffer, ordered=False).inserted_ids)
    return total
```

**TODO 2 — `crear_indices`** (igualdad primero, rango/orden después):

```python
    return [
        db.eventos.create_index([("categoria", ASCENDING), ("fecha", ASCENDING)]),
        db.ordenes.create_index([("evento_id", ASCENDING), ("estado", ASCENDING)]),
    ]
```

Para el informe: `ordered=False` → si un documento falla (p. ej. `_id` duplicado) el resto del lote
se sigue insertando y el servidor puede paralelizar; con `ordered=True` se detiene en el primer error.
Orden del índice: `categoria` (igualdad) antes que `fecha` (rango + sort) → el sort sale gratis del índice.

**APP**

```bash
python3 labs/lab3-mongo-replicas/cargar.py
```

Esperado: 5 líneas `OK:` + línea `Carga: ... documentos/s`.

**HOST**

```bash
set -o pipefail
make lab3-check | tee evidence/config-check.txt     # meta: 6/6
make lab3-status                                    # ver qué N es PRIMARY
```

**Explain** (sustituya `N` por el primario) — **HOST**:

```bash
docker exec ti4601-mongo-N mongosh boleteria --quiet --eval '
const q = { categoria: "concierto", fecha: { $gte: "2026-12-01" } };
print("== CON ÍNDICE (esperado IXSCAN) ==");
const a = db.eventos.find(q).sort({ fecha: 1 }).explain("executionStats");
printjson(a.queryPlanner.winningPlan);
printjson(a.executionStats);
print("== SIN ÍNDICE, hint $natural (esperado COLLSCAN) ==");
const b = db.eventos.find(q).sort({ fecha: 1 }).hint({ $natural: 1 }).explain("executionStats");
printjson(b.queryPlanner.winningPlan);
printjson(b.executionStats);
' | tee evidence/explain-indices.txt
log 5 "cargar.py + lab3-check 6/6 + explain" OK
```

Extraiga para el informe: `totalDocsExamined`, `totalKeysExamined`, `nReturned`, `executionTimeMillis`,
etapa `IXSCAN` vs `COLLSCAN`. **Argumente con documentos examinados, no con ms.**

> 💾 Buen momento para un commit (TODO 1 y 2 + evidencia).

---

## Paso 6 — Oplog y retraso · 👤 A

**MONGOSH en el primario** (`make lab3-shell NODE=mongo-N`, prompt `[direct: primary]`):

```javascript
use boleteria
db.ordenes.insertOne({ _id: "ORD-MANUAL-1", evento_id: "EV-0001", zona: "General", cantidad: 2, estado: "confirmada" })
db.getSiblingDB("local").oplog.rs.find({ ns: "boleteria.ordenes" }).sort({ $natural: -1 }).limit(3)
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()
```

**MONGOSH en un secundario** (otra terminal, `make lab3-shell NODE=mongo-M`, prompt `[direct: secondary]`):

```javascript
use boleteria
db.ordenes.find({ _id: "ORD-MANUAL-1" })              // debe aparecer
db.ordenes.insertOne({ _id: "ORD-NO", estado: "x" })  // anote el error (NotWritablePrimary)
```

**HOST** (N = primario):

```bash
docker exec ti4601-mongo-N mongosh --quiet --eval '
db.getSiblingDB("local").oplog.rs.find({ ns: "boleteria.ordenes" }).sort({ $natural: -1 }).limit(3).forEach(printjson);
rs.printReplicationInfo();
rs.printSecondaryReplicationInfo();
' | tee evidence/oplog.txt
log 6 "oplog + escritura en secundario rechazada" OK
```

Informe: campos `op`, `ns`, `o`, `ts`, `t`; por qué el oplog es capado y qué pasa si un secundario se atrasa más que la ventana (necesita *initial sync*); el error del secundario.

---

## Paso 7 — Lecturas en secundario · 👤 B

📝 Predicción ANTES: % de lecturas obsoletas para `w1_secundario`, `majority_secundario`, `causal_majority`.

**APP**

```bash
set -o pipefail
python3 labs/lab3-mongo-replicas/lectura_secundario.py --n 500 \
  --csv evidence/lectura-secundario.csv | tee evidence/lectura-secundario.txt
```

Esperado: 3 filas × 500 intentos; `causal_majority` = 0 %.

Informe: por qué `majority` mejora pero no llega a 0 % (2 de 3 tienen el dato, la lectura pudo ir al
secundario que aún no lo tiene); clasificación CAP/PACELC de cada modo; qué usar para "mis entradas"
(causal/primario) vs "eventos destacados" (secundario, tolera obsoleto).

```bash
log 7 "lectura_secundario --n 500" OK   # en HOST
```

---

## Paso 8 — TODO 3: compra atómica (`comprar.py`) · 👤 B

Archivo: [labs/lab3-mongo-replicas/comprar.py](labs/lab3-mongo-replicas/comprar.py)

```python
    # 1) registrar la orden: el _id único detecta reintentos (idempotencia)
    try:
        db[ordenes].insert_one({
            "_id": orden_id, "evento_id": evento_id, "zona": zona,
            "cantidad": cantidad, "estado": "pendiente",
            "creada_en": datetime.now(timezone.utc),
        })
    except DuplicateKeyError:
        return db[ordenes].find_one({"_id": orden_id})["estado"]

    # 2) descontar en UNA operación atómica: filtro y $inc sobre el MISMO elemento del arreglo
    res = db[eventos].update_one(
        {"_id": evento_id,
         "zonas": {"$elemMatch": {"nombre": zona, "disponibles": {"$gte": cantidad}}}},
        {"$inc": {"zonas.$.disponibles": -cantidad}},
    )
    estado = "confirmada" if res.modified_count == 1 else "agotada"

    # 3) cerrar la orden
    db[ordenes].update_one({"_id": orden_id}, {"$set": {"estado": estado}})
    return estado
```

**APP**

```bash
set -o pipefail
python3 labs/lab3-mongo-replicas/comprar.py | tee evidence/comprar.txt
```

Esperado: 6 líneas `OK:` y `todas las verificaciones pasaron`.

**Versión rota a propósito** (para el informe): reemplace temporalmente el paso 2 por
"leer con `find_one` → decidir en Python → `update_one` con `$set`", ejecute, y guarde la salida:

```bash
python3 labs/lab3-mongo-replicas/comprar.py | tee evidence/comprar-roto.txt
```

Luego **restaure la versión correcta** y vuelva a correr `comprar.py | tee evidence/comprar.txt`.

Informe:
1. Atomicidad de un documento vs. varios documentos.
2. `zonas.nombre` + `zonas.disponibles` separados vs `$elemMatch`: construya un contraejemplo
   (p. ej. zona "General" con 0 disponibles y "VIP" con 10 → sin `$elemMatch` el filtro hace match y
   el `$` puede apuntar a la zona equivocada / vender sin stock).
3. Si el proceso muere entre descontar y marcar `confirmada`: la orden queda `pendiente` con
   entradas ya descontadas. Detección: buscar órdenes `pendiente` viejas y reconciliar contra el
   evento. Garantía resultante: "como máximo una vez" para el descuento por orden (gracias al `_id`),
   pero el estado puede quedar inconsistente sin un proceso de reconciliación.

```bash
log 8 "comprar.py OK + versión rota" OK   # en HOST
```

> 💾 Commit (TODO 3).

---

## Paso 9 — TODO 4: costo de las garantías (`bench_write_concern.py`) · 👤 B

📝 Predicción ANTES: orden de más rápida a más lenta y cuántas veces más lenta es `majority` que `w1`.

Archivo: [labs/lab3-mongo-replicas/bench_write_concern.py](labs/lab3-mongo-replicas/bench_write_concern.py)

```python
    vista = coleccion.with_options(
        write_concern=WriteConcern(w=w, j=j, wtimeout=wtimeout_ms)
    )
    latencias: list[float] = []
    for i in range(n):
        doc = {"i": i, "relleno": "x" * 100}
        t0 = time.perf_counter()
        vista.insert_one(doc)
        latencias.append((time.perf_counter() - t0) * 1000)
    return latencias
```

**APP**

```bash
set -o pipefail
python3 labs/lab3-mongo-replicas/bench_write_concern.py --n 300 \
  --csv evidence/write-concern.csv | tee evidence/write-concern.txt
wc -l evidence/write-concern.csv     # debe ser 1201
```

Informe: tabla (n=300, warmup=30, p50, p95, p99, ops/s) + máquina/RAM/SO; por qué `w3` ≈ `majority`
en 3 nodos locales; qué agrega `j:true`; por qué con regiones distintas `majority` costaría mucho más
(y `w1` no cambiaría).

```bash
log 9 "bench_write_concern --n 300 (1201 líneas)" OK   # en HOST
```

> 💾 Commit (TODO 4).

---

## Paso 10 — TODO 5: RPO (`verificar_rpo.py`) + línea base · 👤 C

Archivo: [labs/lab3-mongo-replicas/verificar_rpo.py](labs/lab3-mongo-replicas/verificar_rpo.py)

```python
    return {
        "perdidos": confirmados - presentes,
        "desconocidos_presentes": sin_confirmar & presentes,
        "desconocidos_ausentes": sin_confirmar - presentes,
        "inesperados": presentes - confirmados - sin_confirmar,
    }
```

**APP**

```bash
python3 labs/lab3-mongo-replicas/verificar_rpo.py --autotest     # 3 × OK: caso sintético

set -o pipefail
python3 labs/lab3-mongo-replicas/sonda_escrituras.py --duration 10 --label "Base" \
  --csv evidence/rpo-baseline.csv | tail -5
python3 labs/lab3-mongo-replicas/verificar_rpo.py --csv evidence/rpo-baseline.csv \
  | tee evidence/rpo-baseline.txt
```

Esperado: todo confirmado y presente, `PERDIDOS 0`. **Si no, no continúe.**

**HOST**

```bash
make lab3-status            # anote quién es PRIMARY antes de las fallas
log 10 "autotest + rpo-baseline PERDIDOS 0" OK
```

> 💾 Commit (TODO 5).

---

## Fallas — reglas comunes · 👤 C

- Se necesitan **dos terminales HOST** (A = sonda, B = provocar la falla). Pegue `log()` en ambas.
- Comando de **rescate** (téngalo a mano y úselo al terminar cada escenario):

  ```bash
  docker start ti4601-mongo-1 ti4601-mongo-2 ti4601-mongo-3
  ```
- **No pase al siguiente escenario** hasta que `make lab3-status` muestre 1 `PRIMARY` + 2 `SECONDARY`.
- Si `mongo-1` está caído, use `make lab3-status NODE=mongo-2`.
- Siempre `--no-deps` en `docker compose run`.

---

## Paso 11 — Falla A: cae un secundario · 👤 C

📝 Predicción: ¿errores? ¿cambia la latencia? ¿cuántos nodos quedan para mayoría?

**Terminal B (HOST)** — precondición:

```bash
make lab3-status | tee evidence/falla-a-before.txt     # elija un SECUNDARIO = N, y un vivo = M
```

**Terminal A (HOST)**

```bash
set -o pipefail
rm -f evidence/falla-a-stop.epoch
docker compose --profile lab3 run --rm --no-deps app-mongo \
  python3 labs/lab3-mongo-replicas/sonda_escrituras.py \
    --duration 40 --label "Falla A" \
    --signal-file evidence/falla-a-stop.epoch \
    --csv evidence/falla-a.csv \
  | tee evidence/falla-a.txt
```

Espere ~5 s de líneas `antes-falla ok`, luego **Terminal B** (uno por uno, sustituya N y M):

```bash
docker stop --timeout 0 ti4601-mongo-N
date +%s.%N > evidence/falla-a-stop.epoch
date --iso-8601=ns | tee evidence/falla-a-stop.txt
make lab3-status NODE=mongo-M | tee evidence/falla-a-status.txt
sleep 15
docker start ti4601-mongo-N
```

Cuando termine A:

```bash
make lab3-status | tee evidence/falla-a-recovered.txt
docker compose --profile lab3 run --rm --no-deps app-mongo \
  python3 labs/lab3-mongo-replicas/verificar_rpo.py --csv evidence/falla-a.csv \
  | tee evidence/falla-a-rpo.txt
log 11 "Falla A secundario mongo-N" OK
```

Esperado: sin errores, quizá un pico de latencia, `PERDIDOS 0`, nodo vuelve como `SECONDARY`.
Informe: p50/p99 antes y después (filas del CSV), continuidad **por mayoría 2 de 3** (si dice otra cosa, Fallas a la mitad), qué hace el secundario al volver.

---

## Paso 12 — Falla B: cae el primario · 👤 C

📝 Predicción: ¿cuántos segundos sin primario? ¿de qué parámetro depende? ¿qué verá la sonda?

**Terminal B (HOST)**

```bash
make lab3-status | tee evidence/falla-b-before.txt     # PRIMARIO = N, otro = M
# term de SU clúster antes de la falla (no use el de rs-inspect.txt de A: es otro clúster)
docker exec ti4601-mongo-N mongosh --quiet --eval 'print("term=" + rs.status().term)' \
  | tee -a evidence/falla-b-before.txt
```

**Terminal A (HOST)**

```bash
set -o pipefail
rm -f evidence/falla-b-stop.epoch
docker compose --profile lab3 run --rm --no-deps app-mongo \
  python3 labs/lab3-mongo-replicas/sonda_escrituras.py \
    --duration 60 --label "Falla B" \
    --signal-file evidence/falla-b-stop.epoch \
    --csv evidence/falla-b.csv \
  | tee evidence/falla-b.txt
```

**Terminal B (HOST)**

```bash
docker kill ti4601-mongo-N
date +%s.%N > evidence/falla-b-stop.epoch
date --iso-8601=ns | tee evidence/falla-b-stop.txt
sleep 15
make lab3-status NODE=mongo-M | tee evidence/falla-b-status.txt
docker exec ti4601-mongo-M mongosh --quiet --eval 'print("term=" + rs.status().term)' \
  | tee -a evidence/falla-b-status.txt
sleep 15
docker start ti4601-mongo-N
```

Cuando termine A:

```bash
make lab3-status | tee evidence/falla-b-recovered.txt
docker compose --profile lab3 run --rm --no-deps app-mongo \
  python3 labs/lab3-mongo-replicas/verificar_rpo.py --csv evidence/falla-b.csv \
  | tee evidence/falla-b-rpo.txt
log 12 "Falla B primario mongo-N" OK
```

Esperado: racha de errores (`ServerSelectionTimeoutError`), nuevo PRIMARY con `term` mayor que el de `falla-b-before.txt`, el viejo vuelve como SECONDARY, `PERDIDOS 0`.

Informe:
- **RTO** = `ts` de la primera fila `ok` después de la falla − valor de `falla-b-stop.epoch`. **Pegue las filas del CSV** (no solo el número de la sonda). Explique por qué RTO > `electionTimeoutMillis` (detección + elección + descubrimiento del driver + reintento).
- **RPO** sobre confirmados (`ok`) únicamente.
- Cantidad/tipo de errores y por qué `retryWrites` no los ocultó.
- Cambio de `term` y por qué el viejo primario no recupera el rol.

**Opcional (recomendado para P2):** en MONGOSH del primario

```javascript
cfg = rs.conf(); cfg.settings.electionTimeoutMillis = 5000; rs.reconfig(cfg)
```

Repita con archivos `falla-b5.*`, compare RTO, y **restaure a 10000** al terminar.

---

## Paso 13 — Falla C: dos secundarios caídos, primario vivo · 👤 C

📝 Predicción ANTES (3 preguntas del README §13: ¿habrá primario?, ¿resultado de cada insert `w:majority`?, comparación con Falla B del Lab 1).

**Terminal A (HOST)**

```bash
set -o pipefail
rm -f evidence/falla-c-stop.epoch
docker compose --profile lab3 run --rm --no-deps app-mongo \
  python3 labs/lab3-mongo-replicas/sonda_escrituras.py \
    --duration 90 --label "Falla C" \
    --w majority --wtimeout-ms 2000 \
    --signal-file evidence/falla-c-stop.epoch \
    --csv evidence/falla-c.csv \
  | tee evidence/falla-c.txt
```

**Terminal B (HOST)** — X, Y = secundarios; P = primario:

```bash
make lab3-status | tee evidence/falla-c-before.txt
docker kill ti4601-mongo-X ti4601-mongo-Y
date +%s.%N > evidence/falla-c-stop.epoch
date --iso-8601=ns | tee evidence/falla-c-stop.txt
sleep 20
make lab3-status NODE=mongo-P | tee evidence/falla-c-status.txt
sleep 10
docker start ti4601-mongo-X ti4601-mongo-Y
```

Cuando termine A y el conjunto se estabilice:

```bash
make lab3-status | tee evidence/falla-c-recovered.txt
docker compose --profile lab3 run --rm --no-deps app-mongo \
  python3 labs/lab3-mongo-replicas/verificar_rpo.py --csv evidence/falla-c.csv \
  | tee evidence/falla-c-rpo.txt
log 13 "Falla C sin mayoría" OK
```

Esperado: ninguna escritura confirmada sin mayoría; el sobreviviente pasa a `SECONDARY`;
probablemente `errores que SÍ se escribieron > 0`.

Informe: por qué el sobreviviente baja (step down, vs `electionTimeoutMillis`); `WTimeoutError` vs
`ServerSelectionTimeoutError`; **"error ≠ no escrito"** y su efecto en un cliente que compra;
por qué el RPO es sobre lo confirmado; por qué el P2 exige operaciones **idempotentes**.

**Al final:** `docker start ti4601-mongo-1 ti4601-mongo-2 ti4601-mongo-3` y `make lab3-status` → 1 PRIMARY + 2 SECONDARY.

> 💾 Commit de toda la evidencia de fallas.

---

## Entregable

### Archivos en `evidence/`

```text
comandos.md
cluster-start.txt          rs-inspect.txt           datos-manifest.txt
config-check.txt           explain-indices.txt      oplog.txt
lectura-secundario.txt     lectura-secundario.csv   comprar.txt
write-concern.txt          write-concern.csv        rpo-baseline.csv   rpo-baseline.txt
falla-a.txt falla-a.csv falla-a-before.txt falla-a-stop.txt falla-a-status.txt falla-a-recovered.txt falla-a-rpo.txt
falla-b.txt falla-b.csv falla-b-before.txt falla-b-stop.txt falla-b-status.txt falla-b-recovered.txt falla-b-rpo.txt
falla-c.txt falla-c.csv falla-c-before.txt falla-c-stop.txt falla-c-status.txt falla-c-recovered.txt falla-c-rpo.txt
```

Comprobar que no falte nada (**HOST**):

```bash
cd evidence && for f in comandos.md cluster-start.txt rs-inspect.txt datos-manifest.txt config-check.txt \
  explain-indices.txt oplog.txt lectura-secundario.txt lectura-secundario.csv comprar.txt \
  write-concern.txt write-concern.csv rpo-baseline.csv rpo-baseline.txt \
  falla-{a,b,c}.txt falla-{a,b,c}.csv falla-{a,b,c}-{before,stop,status,recovered,rpo}.txt; do
  test -s "$f" || echo "FALTA: $f"; done; cd ..
```

### Código con TODO resueltos (commits de los 3 integrantes)

`cargar.py` (TODO 1, 2) · `comprar.py` (TODO 3) · `bench_write_concern.py` (TODO 4) · `verificar_rpo.py` (TODO 5)

### PDF (máx. 6 páginas)

- **Pág. 1–2 (pasos 1–7):** diagrama 3 nodos; `rs.initiate()` propio y qué hace cada campo; lectura de `rs.status()`/`rs.conf()` (primario, `term`, `electionTimeoutMillis`, `votes`); explain de índices; oplog; tabla de lecturas obsoletas + CAP/PACELC.
- **Pág. 3 (pasos 8–9):** decisiones TODO 1–5 (`ordered=False`, orden de índices, por qué la compra es atómica, qué mide cada variante, cómo clasifica el RPO); versión rota de la compra; tabla de latencias con n, warmup, máquina y límites de Docker.
- **Pág. 4–6 (fallas):** A, B, C con precondición, marcas de tiempo, errores, **RTO y RPO con filas del CSV**; mayoría 2/3; "error ≠ no escrito".
- Todas las predicciones y respuestas de cada paso (se pueden agrupar por paso).

### Lista de control final

- [ ] `config-check.txt` termina en `6/6`
- [ ] `comprar.txt` termina en `todas las verificaciones pasaron`
- [ ] `write-concern.csv` tiene 1201 líneas
- [ ] Cada RTO muestra las filas del CSV
- [ ] RPO calculado sobre confirmadas, no sobre intentos
- [ ] Predicciones escritas antes de ejecutar
- [ ] `evidence/comandos.md` con hora y `OK` por paso
- [ ] Conjunto restaurado (1 PRIMARY, 2 SECONDARY)
- [ ] Máquina, RAM y SO declarados
- [ ] Commits de los 3 integrantes, repartidos en el tiempo
- [ ] `.env.example` sin la licencia en el repo

---

## Errores comunes (atajos)

| Síntoma | Acción |
| --- | --- |
| `ServerSelectionTimeoutError` / `connection refused` | `make lab3-status`, esperar 10–20 s, que haya PRIMARY |
| `NotYetInitialized` | falta `rs.initiate()` (Paso 2) |
| `already initialized` | no lo repita; use `rs.status()` |
| `make lab3-status` cuelga | `make lab3-status NODE=mongo-2` |
| `NotWritablePrimary` | está en un secundario; abra el shell en el primario |
| `NotPrimaryNoSecondaryOk` | `db.getMongo().setReadPref("secondaryPreferred")` |
| `compose run` espera a nodos caídos | olvidó `--no-deps` |
| `No existe data/MANIFEST.json` | correr `generar_datos.py` (Paso 4) |
| sonda terminó antes de recuperar | repetir con `--duration` mayor y otro nombre de archivo |

Logs sin borrar datos: `docker compose --profile lab3 logs --tail 50 mongo-1 mongo-2 mongo-3`
Apagar sin borrar: `make lab3-down` · **No usar `make lab3-down-v`** salvo reinicio total (obliga a repetir desde el Paso 1).
