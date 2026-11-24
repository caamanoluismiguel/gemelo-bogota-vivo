# Bogotwin: datos vivos

Este repo publica los datos que cambian cada pocos minutos de [Bogotwin](https://github.com/caamanoluismiguel/gemelo-bogota), un gemelo digital abierto e independiente de Bogotá. Los JSON se arman en GitHub Actions y se publican en GitHub Pages sin hacer commit. A git solo entra el manifiesto mensual de los días cerrados, como mucho una vez al día.

Este proyecto es independiente y no es de la Alcaldía de Bogotá ni de sus entidades. No emite alertas: para eso están el IDIGER y el IDEAM.

## Cómo se actualiza

- Un Worker de Cloudflare (`worker/src/cosecha.ts` en el repo principal) lanza `cosecha.yml` cada 10 minutos.
- `vigia.yml` es el respaldo: si `indice.json` tiene más de 20 minutos y no hay una cosecha en marcha, la relanza. GitHub atrasa y se salta muchas corridas programadas, por eso no es el motor.
- Cada corrida recupera lo publicado (con `indice.json`, comprobando bytes y sha256), actualiza solo las partes a las que les toca según la edad de su archivo y vuelve a publicar todo. Si no puede recuperar, no publica y queda lo anterior.
- Cuando algo falla, el archivo se queda con su hora vieja. Todos traen `generado`; quien los lea debe mostrar la edad y decir «sin dato reciente» pasado el doble de `cadencia_min`.

GitHub Pages responde con `Cache-Control: max-age=600`. Para leer lo último, agrega `?t=<número>` a la URL.

## Archivos

Base: `https://caamanoluismiguel.github.io/gemelo-bogota-vivo/`

| Archivo | Cadencia | Qué es |
|---|---|---|
| `indice.json` | 10 min | Lista de archivos publicados con bytes, sha256, hora y cadencia |
| `cielo/ahora.json` | 20 min | Tarjeta «el cielo de Bogotá ahora» (METAR, UV, lluvia SAB, radar, GOES‑19, focos, sismos) |
| `bicis/ahora.json` | 10 min | Estado de cada estación de bicicletas compartidas |
| `bicis/serie-48h.ndjson.gz` | 10 min | Las últimas 48 h de estados, una fila por corrida |
| `pronostico/ahora.json` | 60 min | Pronóstico de temperatura y lluvia a 48 h en El Dorado |
| `pronostico/emitidos-5d.ndjson.gz` | 60 min | Cada pronóstico emitido en los últimos 5 días, con 55 h |
| `pronostico/acierto.json` | 1 día | Cuánto acertó el pronóstico contra el METAR de El Dorado |
| `aire/pronostico.json` | 60 min | PM2,5 de CAMS a 5 días, un valor para toda la ciudad |

Los días cerrados van como activos de Releases mensuales: `bicis-AAAA-MM` y `pronostico-AAAA-MM`, un `ndjson.gz` por día de Bogotá. Las huellas están en `manifiestos/<release>.json`. Los activos de Releases no mandan cabeceras CORS: sirven para archivo y análisis, no para leerlos desde el navegador.

## Convenciones comunes

- Horas en UTC con `Z` (`2026-11-24T23:20:00Z`). Los días (`fecha_bogota`) son de Bogotá, UTC−5.
- `esquema` dice el formato y su versión. Un cambio que rompa sube el número.
- `generado`: cuándo se armó el archivo. `cadencia_min`: cada cuánto debería renovarse.
- `etiqueta`: `de_fuente` (lo reporta un tercero tal cual), `medido`, `estimado` (salida de un modelo) o `inferido` (cálculo nuestro sobre datos de otros).
- `corrida`: `{motor, registro, commit_codigo}`. `motor` es quién lanzó la corrida (`worker-cloudflare`, `vigia-github`, `manual`), `registro` es la URL de la corrida de Actions y `commit_codigo` es el commit del repo principal que armó el dato. Con eso se rehace cualquier archivo.
- Los `ndjson.gz` se comprimen sin hora en la cabecera gzip: el mismo contenido da el mismo sha256.

## `indice.json`, esquema `bogotwin/indice/1`

```json
{
  "esquema": "bogotwin/indice/1",
  "generado": "2026-11-24T23:30:12Z",
  "bytes_total": 412345,
  "archivos": {
    "bicis/ahora.json": {"bytes": 31201, "sha256": "…", "generado": "2026-11-24T23:30:05Z", "cadencia_min": 10}
  },
  "corrida": {"motor": "worker-cloudflare", "registro": "…", "commit_codigo": "…"}
}
```

Tope del sitio: 3 MB. Si se pasa, la corrida falla y no publica.

## `bicis/ahora.json`, esquema `bogotwin/bicis/1`

Fuente: feed GBFS 3.0 del sistema de bicicletas compartidas de Bogotá (`https://bogota.publicbikesystem.net/customer/gbfs/v3.0/gbfs.json`). El feed no declara `license_id` ni `license_url`; la especificación GBFS dice que en ese caso los datos son CC0 1.0. Si el feed llega a declarar una licencia que no esté en la lista aceptada, la cosecha se detiene sola.

```json
{
  "esquema": "bogotwin/bicis/1",
  "generado": "2026-11-24T23:30:05Z",
  "hora_dato": "2026-11-24T23:29:40Z",
  "cadencia_min": 10,
  "etiqueta": "de_fuente",
  "fuente": {"nombre": "…", "url": "…", "sistema": "…", "licencia": {"nombre": "CC0 1.0", "url": "…", "por_que": "…"}, "nota": "…"},
  "campos": ["bicis", "electricas", "anclajes_libres", "bicis_fuera_de_servicio", "anclajes_fuera_de_servicio", "estado", "min_desde_reporte"],
  "estado_bits": {"instalada": 1, "presta": 2, "recibe": 4, "dato_dudoso": 8},
  "totales": {"estaciones": 258, "con_estado": 256, "bicis": 1567, "electricas": 243, "anclajes_libres": 0, "vacias": 11, "llenas": 25, "sin_servicio": 0, "dudosas": 2},
  "estaciones": [
    {"id": "1", "nombre": "001 - …", "lat": 4.66, "lon": -74.05, "capacidad": 19, "v": [12, 3, 7, 0, 0, 7, 4]}
  ],
  "corrida": {}
}
```

- `v` sigue el orden de `campos`. `hora_dato` es el `last_updated` del feed; `generado` es cuando lo leímos.
- `estado` es una suma de bits: 7 = instalada, presta y recibe. Con el bit 8 la estación es `dato_dudoso`.
- `dato_dudoso`: cuentas que no cuadran (bicis más anclajes libres por encima del doble de la capacidad más 5, o el desglose por tipo no suma las bicis disponibles) o coordenadas fuera de la Sabana (lat 4,2 a 5,2; lon −74,5 a −73,7). Hoy son dos estaciones con capacidad 1 que reportan 198 bicis y llevan años sin reportar. No entran en `totales` y sus `electricas` van en `null`.
- `electricas`: bicis disponibles de tipos con `propulsion_type` distinto de `human`.
- `min_desde_reporte`: minutos entre el último reporte de la estación y `hora_dato`. Un número grande quiere decir que la estación no está reportando.
- `vacias`: con estado y 0 bicis. `llenas`: con estado y 0 anclajes libres. `sin_servicio`: sin el bit de presta.
- Peso medido el 24 nov 2026: unos 30 KB, unos 7 KB con gzip.

## `bicis/serie-48h.ndjson.gz`

Una línea JSON por corrida, las últimas 48 h:

```json
{"t": "2026-11-24T23:29:40Z", "ids": ["1", "2", "…"], "v": [[12, 3, 7, 0, 0, 7, 4], "…"]}
```

`ids[i]` va con `v[i]`, y cada `v[i]` sigue `campos` de `ahora.json`. Si una corrida faltó, falta la línea: los huecos quedan como huecos, no se rellenan.

## Día cerrado de bicis (Release `bicis-AAAA-MM`)

`bicis-AAAA-MM-DD.ndjson.gz`. Primera línea `{"tipo": "cabecera", "esquema": "bogotwin/bicis/1/dia", "fecha_bogota", "campos", "estado_bits", "estaciones": [{id, nombre, lat, lon, capacidad}], "fuente", …}`; luego una línea `{"tipo": "fila", "t", "ids", "v"}` por corrida de ese día de Bogotá.

## `pronostico/ahora.json`, esquema `bogotwin/pronostico/1`

```json
{
  "esquema": "bogotwin/pronostico/1",
  "generado": "2026-11-24T23:30:00Z",
  "emitido": "2026-11-24T23:30:00Z",
  "cadencia_min": 60,
  "etiqueta": "estimado",
  "punto": {"nombre": "Aeropuerto El Dorado", "icao": "SKBO", "lat": 4.706, "lon": -74.151, "altura_m": 2547},
  "celda_modelo": {"lat": 4.7, "lon": -74.15, "altura_m": 2547},
  "modelo": "best_match de Open-Meteo …",
  "nota": "…",
  "horas": [{"t": "2026-11-24T23:00:00Z", "temp_c": 13.1, "prob_lluvia_pct": 35, "lluvia_mm": 0.1}],
  "fuente": {"nombre": "Open-Meteo, API de pronóstico", "url": "…", "licencia": {"nombre": "CC BY 4.0", "url": "…", "atribucion": "Weather data by Open-Meteo.com", "nota": "…"}},
  "acierto": "pronostico/acierto.json",
  "corrida": {}
}
```

- `horas`: 49 entradas desde la hora en punto en curso. `temp_c` a 2 m en °C, `prob_lluvia_pct` en %, `lluvia_mm` en la hora.
- Es el pronóstico del punto del aeropuerto, en el occidente, el mismo donde mide el METAR. No es el de cada barrio.

## `pronostico/emitidos-5d.ndjson.gz` y día cerrado (Release `pronostico-AAAA-MM`)

Una línea por emisión: `{"emitido", "h0", "temp": [55], "prob": [55], "mm": [55], "celda"}`. `h0` es la hora del primer valor; el valor `i` es de `h0 + i` horas. El día cerrado `pronostico-AAAA-MM-DD.ndjson.gz` lleva una cabecera `bogotwin/pronostico/1/dia` y luego `{"tipo": "emision", …}` por cada emisión de ese día (se esperan 24).

## `pronostico/acierto.json`, esquema `bogotwin/acierto/1`

Califica cada día de Bogotá cerrado, un día después, contra el METAR de El Dorado. Las reglas completas van dentro del archivo, en `reglas`.

```json
{
  "esquema": "bogotwin/acierto/1",
  "generado": "…",
  "cadencia_min": 1440,
  "etiqueta": "inferido",
  "resumen": {
    "ultimos_7_dias": {
      "desde": "2026-11-18", "hasta": "2026-11-24", "dias_calificados": 7,
      "a24h": {
        "temperatura": {"horas": 160, "error_medio_c": 1.2, "sesgo_c": -0.3},
        "lluvia": {"bloques": 28, "si_si": 6, "si_no": 4, "no_si": 3, "no_no": 15, "aciertos": 21, "porcentaje": 75.0}
      },
      "a48h": {}
    },
    "ultimos_30_dias": {}
  },
  "dias": [{"fecha_bogota": "2026-11-23", "reportes_metar": 30, "temperatura": {"a24h": {}, "a48h": {}}, "lluvia": {"a24h": {}, "a48h": {}}}],
  "reglas": {},
  "fuentes": {},
  "licencia_del_archivo": "CC BY 4.0. …"
}
```

- `a24h`: la emisión hecha entre 24 y 29 h antes de cada hora; `a48h`: entre 48 y 54 h antes.
- Temperatura: `error_medio_c` es el promedio del error absoluto; `sesgo_c` el promedio con signo (positivo: el pronóstico se pasó de caliente). El METAR redondea a grados enteros.
- Lluvia: bloques de 6 h. Dijo «sí» si alguna hora tiene 50 % o más; llovió si algún METAR o SPECI del bloque reporta precipitación en la estación (sin contar VC). Se necesitan 4 reportes en el bloque.
- `porcentaje` en `null` quiere decir que todavía no hay bloques calificados. Los primeros días calificados aparecen dos días después de que empieza la cosecha.

## `aire/pronostico.json`, esquema `bogotwin/aire/1`

```json
{
  "esquema": "bogotwin/aire/1",
  "generado": "…",
  "cadencia_min": 60,
  "etiqueta": "estimado",
  "variable": {"clave": "pm25", "nombre": "PM2,5", "unidad": "µg/m³"},
  "punto": {"nombre": "Bogotá, centro aproximado", "lat": 4.65, "lon": -74.1, "celda_modelo": {}, "tamano_celda_km": 45},
  "nota": "…",
  "horas_campos": ["t", "pm25"],
  "horas": [["2026-11-24T23:00:00Z", 18.4]],
  "dias": [{"fecha_bogota": "2026-11-24", "media": 17.9, "maxima": 24.1, "horas": 1}],
  "fuente": {"nombre": "CAMS global (Copernicus), servido por Open-Meteo", "url": "…", "licencia": {"nombre": "CC BY 4.0", "atribucion": "Generated using Copernicus Atmosphere Monitoring Service information, served by Open-Meteo.com", "nota": "…"}}
}
```

- Un solo valor por hora para toda la ciudad, 120 horas desde la hora en curso. La celda de CAMS mide unos 45 km: no sirve por barrio ni por localidad y no se pinta por barrio.
- No hay marcador de acierto: lo medido de la RMCAB no declara licencia. No es un índice oficial ni una alerta.

## `manifiestos/<release>.json`

Una entrada por día cerrado: `{nombre, fecha_bogota, bytes, sha256, filas o emisiones, fuente, licencia, script, corrida, url}`. `sha256sum` del activo descargado debe dar la misma huella.

## Licencias

Las de cada fuente, con crédito: GBFS de bicicletas CC0 1.0; Open-Meteo y CAMS CC BY 4.0 (la API gratis de Open-Meteo es para uso no comercial); METAR dominio público (NOAA/NWS); lluvia del SAB del IDIGER CC BY‑NC 4.0. Lo NC y lo SA nunca se combinan en un mismo cálculo. Los cálculos nuestros (acierto) van con CC BY 4.0 citando sus fuentes.

Código bajo AGPL‑3.0.
