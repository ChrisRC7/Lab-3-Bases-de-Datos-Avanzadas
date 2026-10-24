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

*Pendiente (Persona C).*
