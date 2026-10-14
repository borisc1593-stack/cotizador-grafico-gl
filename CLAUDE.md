# Cotizador Multiservicios Gráficos — Gráficas Liz

App de cotización para un negocio de servicios gráficos (Gráficas Liz, Portoviejo, Ecuador). Single-file vanilla JS (`index.html`), todo en `localStorage`, sin backend. Repo: `borisc1593-stack/cotizador-grafico-gl`.

## Identidad de marca

- Logo: `logo-graficas-liz.png` (ya en el repo, fondo transparente). Usado en el header de la app y en el encabezado de la proforma impresa.
- Colores del logo: cian, magenta, amarillo (estilo CMYK) — ya reflejados en las variables CSS `--cyan`/`--magenta`/`--yellow`.
- Nunca reemplazar este logo por uno genérico ni quitarlo del header/proforma sin que el usuario lo pida.

## Formato de proforma

El formato de la proforma impresa/PDF debe replicar el de las proformas reales del negocio (ejemplo de referencia: proforma para Comercial Silva, `PR-2026-0914-A`, enviada 2026-09-14). Estructura:

1. Header: logo a la izquierda, título "PROFORMA" + N° Ref + Fecha a la derecha.
2. Barra divisoria azul marino.
3. Sección "INFORMACIÓN DEL CLIENTE Y PROYECTO": caja con cliente/concepto a la izquierda, área total/alto uniforme/cantidad a la derecha.
4. Sección "DESGLOSE DE MEDIDAS Y ESPECIFICACIONES": tabla agrupada por material, columnas CANT./ITEM/ANCHO/ALTO/ÁREA.
5. Sección "PRESUPUESTO / COTIZACIÓN": tabla ÁREA·CANT./DESCRIPCIÓN/P.UNIT./SUBTOTAL + totales (subtotal, IVA 15%, TOTAL en caja azul marino).
6. Sección "TÉRMINOS Y CONDICIONES": validez de oferta, forma de pago, alcance, medidas.
7. Pie: "Gracias por su preferencia · Gráficas Liz".

N° de referencia: formato `PR-AAAA-MMDD-Letra` (letra incrementa por cada proforma emitida ese mismo día: A, B, C...).

Implementado 2026-09-16 en `renderPrint()` + CSS `@media print`. No revertir a un formato de texto plano simple — este es el formato real que el negocio usa y envía a clientes.

## Módulo: Impresión UV sobre PVC (cama plana)

La impresora UV es de cama plana. El cobro es por área de la plancha y es proporcional: si se usa media cama, se cobra la mitad.

### Parámetros (editables en la UI, NO hardcodear)
- `anchoCama` = 90 cm
- `altoCama` = 60 cm
- `precioCama` = 30.00 (USD, precio final al cliente)

Estos tres valores viven en el material del catálogo (unit `cama`): `anchoCama`/`altoCama` son campos propios del material, y `precioCama` reutiliza los campos existentes `designer`/`public` del material (precio de la cama completa en cada nivel).

### Fórmula
```
areaCama   = anchoCama * altoCama            // 5400 cm² con los valores por defecto
divisor    = areaCama / precioCama           // 180 cm² por dólar
precioPza  = (anchoPieza * altoPieza) / divisor
subtotal   = precioPza * cantidad
```
Equivale a $55.56/m² con los valores por defecto. Si cambia `precioCama`, el divisor se recalcula solo (no cachear el divisor).

### Campos del ítem (en la pantalla Cotizar, cuando el material seleccionado es tipo `cama`)
- ancho (cm), alto (cm), cantidad
- checkbox "Imprimir ambos lados" → multiplica cantidad × 2 para el cálculo
- campo "Cobro mínimo por pieza": se guarda en el material (`minPieza`, default $1.00) y se precarga al cotizar (editable por ítem). Si `precioPza` calculado es menor, se cobra el mínimo en su lugar

### Salidas a mostrar
- Precio unitario por pieza (2 decimales)
- Subtotal del ítem
- Referencia: cuántas piezas entran por cama (`floor(anchoCama/anchoPieza) * floor(altoCama/altoPieza)`) y % de aprovechamiento de esa cama completa

### Casos de prueba (con anchoCama=90, altoCama=60, precioCama=30.00)
| Pieza | Precio esperado |
|---|---|
| 90 × 60 | $30.00 (cama completa) |
| 90 × 30 | $15.00 (media cama) |
| 55 × 56 | $17.11 |
| 55 × 55 | $16.81 |
| 28 × 29 | $4.51 |
| 56 × 28 | $8.71 |

Implementado 2026-09-16: nuevo `unit: 'cama'` en el modelo de materiales, junto a los ya existentes `m2` y `unidad`. Cada ítem de cotización guarda su propio `unit` (no solo `matId`) para que el desglose/impresión sigan siendo correctos aunque el material se edite o borre después.

### Desglose visual (2026-10-13)
`renderCamaInfo()` muestra en la pantalla Cotizar un dibujo a escala de la cama con las piezas y el paso a paso del precio (cama completa → % que ocupa la pieza → precio por pieza → × impresiones → total). Las "piezas por cama" prueban la pieza normal y girada y usan la que más entra (solo referencia; el precio sigue siendo proporcional al área).

## Catálogo y precios reales (2026-10-14)

Precios dados por el usuario (diseñador / público, por m²): Lona 7/12, Lona traslúcida 12/22, Vinil 7/12, Vinil laminado 12/18, Microperforado 12/22, Letrero tipo pantalla 30/45, Letrero luminoso tubos LED 90/140.

UV Roland por rollo: UN solo material "UV Roland rollo (CMYK)" a $15/m² con `addons` por m² que se marcan como casillas al cotizar: Blanco +5, Barniz +5, Corte +5, Troquelado +3 (editables en el modal del material, sección "Adicionales por m²"; cualquier material m² puede tener `addons`). Migración `cotizador_seed_2026_10_14b` reemplaza las 5 variantes viejas por este.

Migración única con flag `cotizador_seed_2026_10_14`: actualiza precios por nombre e inserta los materiales nuevos (`NEW_MATS_V1`) sin duplicar ni revivir lo que el usuario borre después.

### Instalación y extras (bloque de casillas en Cotizar, para ítems m² y unidad)
- "Instalación / tensado": por m², precargado con `cfg.instRate` (default $4, editable en Materiales y por ítem). Solo en ítems m².
- "Fuera de la ciudad", "Personal adicional", "Andamio": monto fijo por ítem; el campo se habilita al marcar la casilla y va vacío (se escribe el valor).
- Todo se suma igual a diseñador y público y queda escrito en la descripción del ítem (sale en la proforma). Se resetea al agregar el ítem y se restaura al editarlo. Los antiguos materiales "Extra: …" se eliminaron en la migración b.
