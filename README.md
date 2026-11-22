# Sensor UV · servidor y panel web

Servidor del **Medidor UV para colegios de Peñalolén** (TEI201, UAI, 2026).
Recibe las lecturas del ESP32, las guarda en PostgreSQL y muestra un panel web con el estado actual,
alertas, estadísticas del día y gráficos.

```
ESP32 + GUVA-S12SD ──HTTPS POST cada 10 s──▶ Express (Railway) ──▶ PostgreSQL (Aiven)
                                                   │
                                                   └──▶ Panel web (public/index.html)
```

## Archivos

| Archivo | Qué hace |
|---|---|
| `server.js` | API en Express: guarda lecturas y calcula la información del panel |
| `public/index.html` | Panel web con Chart.js, se actualiza solo cada 5 s |
| `esp32/uv_semaforo_nube/uv_semaforo_nube.ino` | Firmware del ESP32: semáforo + Telegram + envío a la nube |
| `package.json` | Dependencias de Node (express, pg, dotenv) |
| `.env.example` | Plantilla de variables, **sin claves reales** |
| `.gitignore` | Evita subir `node_modules/`, `.env` y `ca.pem` |

## Tabla `lecturas_uv`

| Columna | Tipo | Ejemplo |
|---|---|---|
| `id` | SERIAL | 1532 |
| `dispositivo` | VARCHAR(50) | `uv-colegio-1` |
| `indice_uv` | REAL | 6.4 |
| `milivoltios` | INTEGER | 730 |
| `nivel` | VARCHAR(12) | `ALTO` |
| `creado_en` | TIMESTAMPTZ | 2026-11-22 13:05:10-03 |

La tabla se crea sola al arrancar el servidor.

## API

| Método y ruta | Uso |
|---|---|
| `GET /api/salud` | Comprueba que el servidor esté conectado a la base |
| `POST /api/lecturas` | El ESP32 guarda una lectura. Requiere el header `x-api-key` |
| `GET /api/lecturas?limite=50` | Últimas lecturas (tabla del panel) |
| `GET /api/ultima` | Lectura más reciente y segundos desde que llegó |
| `GET /api/historial?horas=24` | Promedio y máximo agrupados para el gráfico |
| `GET /api/resumen` | Información del día: máximo y hora, promedio, minutos en cada nivel, alertas |
| `GET /api/exportar.csv?horas=24` | Descarga de datos para el informe de testing |

Ejemplo del JSON que manda el ESP32:

```json
{"dispositivo":"uv-colegio-1","indice_uv":6.4,"milivoltios":730,"nivel":"ALTO"}
```

## Información que genera el panel (para tomar decisiones)

- **Nivel OMS actual** con su color y la recomendación para el colegio (recreo, Educación Física).
- **Alerta** visible cuando el índice llega a ALTO (6) o más.
- **Máximo del día** y a qué hora ocurrió.
- **Tiempo con protección necesaria** (UV ≥ 3) y **minutos en cada nivel**.
- **Número de alertas** del día (veces que el nivel subió a ALTO o más).
- **Máximo diario de los últimos 7 días** para comparar días.
- Indicador **en línea / sin datos** (si no llega nada en 60 s).

## Ponerlo en marcha

1. Crear la base PostgreSQL **Free** en Aiven y copiar la Service URI y el certificado CA.
2. Crear el proyecto en Railway desde este repositorio.
3. En Railway, pestaña **Variables**: `DATABASE_URL`, `DB_CA_CERT` y `API_KEY`. Presionar **Deploy**.
4. En **Settings → Networking → Generate Domain** usar el puerto **8080**.
5. Abrir `https://SU-APP.up.railway.app/api/salud` → debe decir `{"ok":true,"db":"conectada"}`.
6. En el `.ino` completar `WIFI_NOMBRE`, `WIFI_CLAVE`, `SERVER_URL` (terminada en `/api/lecturas`)
   y `API_KEY` (la misma de Railway). Instalar **ArduinoJson** v7 en Arduino IDE y subir el código.
7. En el Monitor Serie (115200) debe aparecer `[Nube] POST ... -> 201`.

Probar en el computador (opcional): copiar `.env.example` a `.env`, completar los valores, y ejecutar
`npm install` y `npm start`. El panel queda en http://localhost:3000.

> Las claves reales (`.env`, contraseña de Aiven, `API_KEY`, clave del WiFi, token de Telegram)
> **nunca** se suben a GitHub.

Basado en la *Guía TEI: ESP32 + Aiven + Railway* del curso. Ver `FUENTES.md` del repositorio del proyecto.
