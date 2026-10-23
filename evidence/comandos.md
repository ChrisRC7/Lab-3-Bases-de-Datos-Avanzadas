# Registro de comandos — Laboratorio 3

- 2026-10-22 14:22:25 · Paso 0 · `docker compose version / make build` · OK
- 2026-10-22 14:23:25 · Paso 1 · `make lab3-up / lab3-status` · OK
- 2026-10-22 14:23:48 · Paso 2 · `rs.initiate rs-lab3 (mongo-1, mongo-2, mongo-3)` · OK
- 2026-10-22 14:24:00 · Paso 3 · `rs-inspect` · OK
- 2026-10-22 14:24:17 · Paso 4 · `generar_datos.py` · OK
- 2026-10-22 14:25:09 · Paso 5 · `cargar.py (5 OK) + lab3-check 6/6 + explain` · OK
- 2026-10-22 14:25:50 · Paso 6 · `oplog + escritura en secundario rechazada (NotWritablePrimary)` · OK
- 2026-10-23 · Paso 7 · `lectura_secundario.py --n 500` · OK.
  - `w1_secundario`: 499/500 lecturas obsoletas (99,8 %).
  - `majority_secundario`: 3/500 lecturas obsoletas (0,6 %).
  - `causal_majority`: 0/500 lecturas obsoletas (0 %).
  - Entorno: Windows + Docker Desktop + WSL 2, MongoDB 8.0.34.
  - Evidencias: `lectura-secundario.csv` y `lectura-secundario.txt`.