# Infraestructura, deploy y APIs externas

Estado al 14/11/2026. Cubre los cuatro repositorios del proyecto Citax: API, Front, Documentación y DataSets.

## 1. Infraestructura

### Componentes

| Componente | Repositorio | Dónde corre | URL |
|---|---|---|---|
| API (Express + TypeScript) | NasaSpaceAppsCitax-Api | Cloudflare Workers | https://nasaspaceappscitax-api.nicotrib699.workers.dev |
| Histórico de FIRMS | NasaSpaceAppsCitax-Api (`migrations/`) | Cloudflare D1 (`nasaspaceappscitax-firms`) | — ([detalle](historico-firms-d1.md)) |
| Front (React 19 + Vite + Leaflet) | NasaSpaceAppsCitax-Front | Cloudflare Workers (assets estáticos) | https://nasaspaceappscitax-front.nicotrib699.workers.dev |
| Documentación | NasaSpaceAppsCitax-Documentacion | Solo GitHub | — |
| Datasets | NasaSpaceAppsCitax-DataSets | Solo GitHub (los .zip se guardan fuera de git) | — |

Flujo de una consulta:

Navegador → Front (Worker de assets) → API (Worker) → NASA FIRMS / Google Earth Engine

Cada respuesta de FIRMS queda guardada en D1 y `/prediction/risk` la reutiliza ([detalle](historico-firms-d1.md)).

### API en Cloudflare Workers

- La misma app Express (`src/app.ts`) se ejecuta de dos formas:
  - **Local:** `src/index.ts` levanta un servidor Node en el puerto 3000 (`npm run dev`).
  - **Producción:** `src/worker.ts` envuelve la app con `httpServerHandler` de `cloudflare:node`. Así Express funciona dentro del Worker sin reescribirlo.
- `wrangler.jsonc` activa los flags `nodejs_compat` y `enable_nodejs_http_server_modules`, que dan las APIs de Node (`http`, `crypto`, `stream`) dentro del runtime de Workers. La fecha de compatibilidad es `2026-11-01`.
- **Secretos del Worker**, leídos con `process.env`:
  - `EARTH_ENGINE_SERVICE_ACCOUNT_JSON`: JSON completo de la cuenta de servicio de Google Cloud.
  - `MAP_KEY`: clave de NASA FIRMS.
  - Se cargan con `npx wrangler secret put <NOMBRE>` y no se versionan. Si falta alguno, el endpoint correspondiente responde 503 en lugar de fallar en silencio.
- CORS abierto (`cors()`), para que el Front pueda consumir la API desde otro dominio.

### Front en Cloudflare Workers

- `vite build` genera `dist/`. Wrangler lo publica como assets estáticos con `not_found_handling: single-page-application`, así cualquier ruta devuelve `index.html`.
- La URL de la API se inyecta en el build con `VITE_API_URL`:
  - `.env.development` → `http://localhost:3000`
  - `.env.production` → URL del Worker de la API
- El mapa usa **Leaflet** con un GeoJSON local de Natural Earth (`src/assets/southern-south-america.json`). No consume servidores de tiles externos.

### Versiones y dependencias

- Node 24 fijado en `.nvmrc`, en API y Front. Es la única fuente de la versión: CI y deploy la leen de ahí.
- `package-lock.json` versionado. CI instala con `npm ci`, que falla si el lockfile y `package.json` no coinciden.
- Dependabot abre PRs semanales:
  - npm: API y Front.
  - GitHub Actions: los cuatro repositorios.

## 2. Deploy (CI/CD con GitHub Actions)

### Problemas que tenía el pipeline original

| Problema | Consecuencia |
|---|---|
| El deploy corría en cada push a `main` sin tests, lint ni type-check. | Se podía desplegar código roto. |
| `node --test` sin tests encontrados igual salía con éxito. | El paso de tests podía dar verde sin probar nada. |
| `src/worker.ts`, el archivo que se despliega, estaba excluido de `tsc`. | Nunca se revisaban sus tipos. |
| oxlint no fallaba con warnings. | Los avisos del linter no frenaban nada. |
| `cancel-in-progress: true` en el deploy. | Un deploy podía cortarse a la mitad. |

### Workflows actuales en API y Front

**`ci.yml`**: corre en cada pull request y también como primer paso del deploy.

1. Instalación estricta con `npm ci`.
2. Front: lint con `oxlint --deny-warnings`, para que los warnings hagan fallar el CI.
3. Type-check:
   - API: app Node y Worker. Antes se generan los tipos del runtime con `wrangler types` y se revisa `worker.ts` con `tsconfig.worker.json`.
   - Front: `tsc -b`.
4. API: tests con `node --test`. El CI exige que corra al menos un test; con cero tests, falla.
5. Front: chequeo de que ningún `.env` empiece con BOM UTF-8.
6. Build.
7. Front: verificación del bundle. Debe contener la `VITE_API_URL` de producción y no puede tener referencias a `localhost`.
8. `wrangler deploy --dry-run`: empaqueta igual que en producción, sin publicar.
9. `npm audit --audit-level=high`.
10. Árbol de trabajo limpio: el pipeline no puede dejar archivos modificados o sin versionar.

**`deploy.yml`**: corre en cada push a `main` y también se puede lanzar a mano.

- **Job `ci`:** reutiliza `ci.yml`. Si falla, no se despliega.
- **Job `deploy`:** depende de `ci` (`needs: ci`).
  - Usa el environment `production` de GitHub, que muestra la URL desplegada y permite exigir una aprobación manual.
  - Concurrencia `deploy-production` con `cancel-in-progress: false`: los deploys se encolan y ninguno se corta a la mitad.
  - Publica con `cloudflare/wrangler-action@v3`:
    - Secretos de GitHub: `CLOUDFLARE_API_TOKEN` y `CLOUDFLARE_ACCOUNT_ID`.
    - Comando: `wrangler deploy --tag <SHA del commit> --message gh-run-<id del run>`.
    - Así, cada versión del Worker en Cloudflare queda ligada al commit y al run de GitHub que la generaron.
- Permisos mínimos: `contents: read`.

### Documentación y DataSets: `checks.yml`

Corre en cada PR y en cada push a `main`. Revisa:

- Marcadores de conflicto de merge sin resolver.
- Archivos de más de 50 MB. GitHub advierte a partir de 50 MB y rechaza archivos de más de 100 MB.
- Links relativos rotos en archivos Markdown.
- Solo en DataSets: que no haya `.zip`, `.7z`, `.rar` ni `.tar*` versionados. El dataset FIRMS de 139 MB se comparte por fuera de git.

### Problemas reales que detectó el nuevo pipeline

1. **Front sin conexión a la API en producción.**
   - `.env.production` y `.env.development` empezaban con un BOM UTF-8, un carácter invisible que agregan algunos editores.
   - Por eso Vite no reconocía `VITE_API_URL`, el bundle quedaba con `API_URL = undefined` y el front llamaba a `undefined/health`.
   - El deploy anterior salía en verde igual.
   - Solución: se guardaron los archivos sin BOM y se agregó el chequeo del bundle.
2. **Error de tipos en el proxy de FIRMS.**
   - Un commit que entró directo a `main` usaba `Readable.fromWeb(response.body)`.
   - Con los tipos de Workers, ese `ReadableStream` no es compatible con el de Node, así que el type-check del Worker falló y el deploy se frenó.
   - En ejecución son el mismo objeto. Se corrigió con una conversión de tipo explícita (PR #3) y el deploy salió bien.

### Historial de deploys

| Fecha | Repositorio | Cambio | Resultado |
|---|---|---|---|
| 14/11/2026 | API | PR #1: pipeline reforzado | CI frenó el deploy por el error de tipos de un commit de FIRMS |
| 14/11/2026 | API | PR #3: corrección de tipos | CI y deploy OK |
| 14/11/2026 | Front | PR #1: pipeline + corrección del BOM | CI y deploy OK |
| 14/11/2026 | Front | Mapa de focos FIRMS (`7ea9e01`) | CI y deploy OK |

### Pendientes

- **Proteger `main`** en Settings → Branches de cada repositorio, exigiendo el check de CI:
  - API y Front: **CI / Verify**.
  - Documentación y DataSets: **Checks / Higiene del repositorio**.
  - Hoy se puede pushear directo a `main` sin pasar por el CI del PR. El deploy igual se frena si algo falla, pero el error aparece recién en `main`.
- Opcional: exigir aprobación manual en el environment `production`.
- Borrar la rama `claude/youthful-cannon-gdvray`, ya mergeada, en los cuatro repositorios.
- El Front todavía no tiene tests.

## 3. APIs externas

### 3.1 NASA FIRMS (focos de calor)

| | |
|---|---|
| Qué es | Fire Information for Resource Management System: detecciones satelitales de anomalías térmicas casi en tiempo real. |
| Fuente usada | `VIIRS_NOAA20_NRT` (sensor VIIRS del satélite NOAA-20, Near Real Time) |
| Endpoint externo | `GET https://firms.modaps.eosdis.nasa.gov/api/area/csv/{MAP_KEY}/VIIRS_NOAA20_NRT/{oeste,sur,este,norte}/{días}[/{fecha}]` |
| Autenticación | `MAP_KEY` gratuita de FIRMS, en la ruta. Se guarda como secreto del Worker y nunca llega al navegador. |
| Formato | CSV |

**Endpoint propio:** `GET /nasa/firms/hotspots`

- **Parámetros:**
  - `bbox=oeste,sur,este,norte` (obligatorio).
  - `days` de 1 a 5 (por defecto 1).
  - `date=YYYY-MM-DD` (opcional).
- **Validación:** antes de llamar a NASA, se verifica que el bbox tenga coordenadas válidas, con oeste < este y sur < norte, que `days` esté entre 1 y 5, y que la fecha exista en el calendario.
  - Datos inválidos → 400.
  - Sin `MAP_KEY` → 503.
- **Proxy:**
  - La API hace `fetch` a FIRMS con un timeout de 30 s y `redirect: 'manual'`.
  - En producción lee el CSV entero, lo responde (`text/csv`) y lo guarda en D1 en segundo plano ([detalle](historico-firms-d1.md)).
  - Sin base (local, `npm run dev`) lo reenvía como stream con `pipeline(Readable.fromWeb(body), res)`.
  - Si FIRMS responde con error o no hay respuesta → 502.

**Uso en el Front** (`src/api.ts` y `src/firms.ts`):

- Pide `bbox=-73.75,-55.25,-53.5,-21.75`, que cubre Argentina y es el mismo bbox del modelo de riesgo (así cada día cargado sirve para el pronóstico), con `days=1` y la fecha elegida (UTC). Timeout de 35 s.
- `parseHotspots` lee el CSV por nombre de columna: `latitude`, `longitude`, `acq_date`, `acq_time`, `satellite`, `instrument`, `confidence`, `frp`, `daynight`.
- Descarta filas inválidas o de otra fecha.
- Con la geometría de Natural Earth, se filtran solo los puntos dentro del límite de Argentina.
- El mapa colorea cada foco por confianza: alta, nominal o baja.

**Ejemplo:**

```
GET /nasa/firms/hotspots?bbox=-73.75,-55.25,-53.5,-21.75&days=1&date=2026-11-14
→ 200 text/csv
latitude,longitude,bright_ti4,scan,track,acq_date,acq_time,satellite,instrument,confidence,version,bright_ti5,frp,daynight
...
```

### 3.2 Google Earth Engine (escenas Sentinel-2)

| | |
|---|---|
| Qué es | Catálogo de imágenes satelitales de Google. Se consulta la colección pública `COPERNICUS/S2_SR_HARMONIZED`: Sentinel-2, reflectancia de superficie armonizada. |
| Endpoint externo | `GET https://earthengine.googleapis.com/v1alpha/projects/earthengine-public/assets/COPERNICUS/S2_SR_HARMONIZED:listImages` |
| Autenticación | OAuth 2.0 con cuenta de servicio de Google Cloud, flujo JWT Bearer. Scope `earthengine.readonly`. |
| Formato | JSON |

**Autenticación** (`src/earth-engine.ts`), sin SDK de Google, solo con `node:crypto`:

1. Se lee `EARTH_ENGINE_SERVICE_ACCOUNT_JSON` y se extraen `project_id`, `client_email` y `private_key`.
2. Se arma un JWT con `iss` = email de la cuenta, `scope` = `https://www.googleapis.com/auth/earthengine.readonly`, `aud` = `https://oauth2.googleapis.com/token` y una validez de 1 hora.
3. Se firma con RS256 (`crypto.sign('RSA-SHA256', ...)`).
4. Se canjea en `POST https://oauth2.googleapis.com/token` con `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`.
5. El access token queda en memoria, ligado a un hash de la cuenta, y se reutiliza hasta 60 s antes de vencer.
6. La consulta a Earth Engine lleva `Authorization: Bearer <token>` y `x-goog-user-project: <project_id>` para facturar la cuota al proyecto propio.

**Endpoint propio:** `GET /earth-engine/scenes`

- **Parámetros:**
  - `lat` y `lon` (obligatorios).
  - `start` y `end` en formato `YYYY-MM-DD`. `end` es exclusivo.
  - `pageToken` (opcional).
- **Validación:** latitud y longitud dentro de rango, fechas válidas, `start` anterior a `end`, y `pageToken` de 2048 caracteres como máximo.
  - Datos inválidos → 400.
  - Sin la cuenta de servicio → 503.
- **Consulta:**
  - Se piden 20 imágenes por página, filtradas por fechas y por la región `{"type":"Point","coordinates":[lon,lat]}`. Timeout de 15 s.
  - La respuesta es `{ images: [...], nextPageToken }`. Para pedir la página siguiente, se reenvía `pageToken`.
  - Si Google falla → 502.

**Ejemplo:**

```
GET /earth-engine/scenes?lat=-34.6037&lon=-58.3816&start=2024-01-01&end=2024-02-01
→ 200 { "images": [ { "id": "...", "startTime": "...", ... } ], "nextPageToken": "..." }
```

**Estado:** el Front todavía no consume este endpoint.

### 3.3 Otros servicios

- **Natural Earth:** límites de países. Se usa como archivo GeoJSON local versionado en el Front y no se consulta en tiempo de ejecución.
- **Cloudflare API:** solo la usa el pipeline de deploy, a través de `wrangler-action` con el token guardado en los secretos de GitHub.
- **Endpoint `/health` de la API:** devuelve `{ "status": "ok" }` y sirve para verificar que el Worker esté arriba.
