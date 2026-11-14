# Histórico de FIRMS en Cloudflare D1

Estado al 14/11/2026. Cada request que la API le hace a NASA FIRMS queda guardado en una base D1, para que el modelo
de predicción consulte los focos directo de la base en lugar de volver a pedírselos a FIRMS.

## Por qué

- FIRMS solo deja pedir de a 1 a 5 días por request y tiene límite de consultas por `MAP_KEY`.
- El modelo necesita ventanas de días ya descargadas, con todas las variables de cada foco (no solo lat/lon).
- Guardando cada respuesta se arma el histórico solo, a medida que la app se usa.

## Dónde está

| | |
|---|---|
| Base | `nasaspaceappscitax-firms` (Cloudflare D1, región ENAM) |
| `database_id` | `ae76b4db-bd62-4aec-8236-2b60d2c37984` (también en `wrangler.jsonc` de la API) |
| Binding en el Worker | `DB` |
| Esquema | `migrations/0001_firms_history.sql` en `NasaSpaceAppsCitax-Api` |
| Código | `src/firms-store.ts` (guardar y leer) y `src/app.ts` (cuándo se guarda) en `NasaSpaceAppsCitax-Api` |

## Qué requests se guardan

| Request | Qué se guarda |
|---|---|
| `GET /nasa/firms/hotspots` con `date` (el Front siempre lo manda: el día elegido y el anterior) | Los `days` días desde `date` |
| `GET /nasa/firms/hotspots` sin `date` | Los últimos `days` días hasta hoy (UTC), que es lo que devuelve FIRMS |
| `GET /prediction/risk?date=...` | La ventana de 5 días que termina en `date`, si hubo que pedírsela a FIRMS |

- Se guarda **después** de responder, en segundo plano (`waitUntil`): el usuario no espera a la base.
- Cada guardado es un `batch` de D1, que corre como una transacción: o entra todo o nada.
- Si D1 falla, se reintenta hasta 3 veces (las inserciones son idempotentes, así que reintentar no duplica).
  Si después de eso sigue fallando, queda el error en los logs del Worker.
- Si FIRMS devuelve algo que no es el CSV esperado, no se guarda nada.

## Tablas

### `firms_detections`: un foco por fila

Todas las columnas del CSV de VIIRS, más `source` y `fetched_at`.

| Columna | Tipo | Detalle |
|---|---|---|
| `source` | TEXT | Producto de FIRMS (`VIIRS_NOAA20_NRT`) |
| `latitude`, `longitude` | REAL | |
| `acq_date` | TEXT | `YYYY-MM-DD`, UTC |
| `acq_time` | TEXT | `HHMM`, UTC (siempre con 4 dígitos) |
| `satellite`, `instrument` | TEXT | `N20`, `VIIRS` |
| `confidence` | TEXT | `l`, `n`, `h` |
| `version` | TEXT | p. ej. `2.0NRT` |
| `bright_ti4`, `bright_ti5` | REAL | Temperatura de brillo (K) |
| `scan`, `track` | REAL | Tamaño del píxel |
| `frp` | REAL | Potencia radiativa del fuego (MW) |
| `daynight` | TEXT | `D` o `N` |
| `fetched_at` | TEXT | Última vez que FIRMS devolvió el foco (ISO 8601) |

Clave primaria: `(source, acq_date, acq_time, latitude, longitude, satellite)`. Si FIRMS vuelve a devolver un foco
(por ejemplo, con valores corregidos), la fila se actualiza en vez de duplicarse.

### `firms_requests`: un request a FIRMS por fila

`endpoint`, `source`, `west`, `south`, `east`, `north`, `start_date`, `days`, `end_date`, `detections` (cuántos focos
trajo) y `requested_at`.

### `firms_days`: qué días se descargaron

Una fila por día y bbox descargado, con `fetched_at`.

- Un día cuenta como **completo** si se descargó después de las **06:00 UTC del día siguiente**. FIRMS NRT publica
  con unas horas de demora, así que un día descargado antes puede estar incompleto.
- Sirve para distinguir "ese día no hubo focos" de "ese día nunca se descargó": sin fila en `firms_days`, que no haya
  focos no significa nada.

## Cómo lo usa `/prediction/risk`

1. Busca en `firms_days` si los 5 días de la ventana están completos para el bbox del modelo
   (`-73.75,-55.25,-53.5,-21.75`) o uno que lo contenga.
2. Si están, lee los focos de `firms_detections` (sin los de confianza baja) y no llama a FIRMS.
3. Si falta alguno, pide la ventana a FIRMS y la guarda.

El Front pide los focos con ese mismo bbox, así que cada día que alguien abre en el mapa también le sirve al modelo.

## Consultar desde el modelo

Con Wrangler (necesita sesión de Cloudflare o `CLOUDFLARE_API_TOKEN`):

```bash
npx wrangler d1 execute nasaspaceappscitax-firms --remote --command \
  "SELECT acq_date, latitude, longitude, confidence, frp FROM firms_detections WHERE acq_date BETWEEN '2026-11-10' AND '2026-11-14'"
```

Con la API REST de D1 (por ejemplo, desde Python), con un token de Cloudflare con permiso de lectura de D1:

```
POST https://api.cloudflare.com/client/v4/accounts/<account_id>/d1/database/ae76b4db-bd62-4aec-8236-2b60d2c37984/query
Authorization: Bearer <token>
{"sql": "SELECT * FROM firms_detections WHERE acq_date BETWEEN ?1 AND ?2", "params": ["2026-11-10", "2026-11-14"]}
```

Antes de usar un rango, conviene chequear que esté completo:

```sql
SELECT COUNT(DISTINCT acq_date) AS dias_completos FROM firms_days
WHERE source = 'VIIRS_NOAA20_NRT' AND acq_date BETWEEN '2026-11-10' AND '2026-11-14'
  AND west <= -73.75 AND south <= -55.25 AND east >= -53.5 AND north >= -21.75
  AND fetched_at >= date(acq_date, '+1 day') || 'T06:00:00.000Z';
-- tiene que dar 5
```

## Cambiar el esquema

1. Agregar `migrations/000N_<nombre>.sql` en la API (no editar las migraciones ya aplicadas).
2. Aplicar: `npx wrangler d1 migrations apply nasaspaceappscitax-firms --remote`.
3. Pushear el código que use las columnas nuevas recién después de aplicar la migración.

La migración `0001` ya está aplicada y registrada en la tabla `d1_migrations`.

## Verificación (14/11/2026)

- Tests de la API con SQLite en lugar de D1: guardado de todas las columnas, actualización de focos repetidos,
  cobertura por día y bbox, `/prediction/risk` leyendo de D1, requests con y sin fecha, y reintento ante una falla de D1.
- En producción: una ventana de `/prediction/risk` guardó 5025 focos y 5 días; el Front real, abierto en un navegador
  headless, guardó sus dos requests (día elegido y anterior).
