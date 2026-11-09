# The5411_UBI-BINS

## Project Overview

**Conversor de ubicaciones a bins** (`ubi-bins`). Es una app web de una sola página, servida por un servidor Node.js mínimo (pensado para Railway), con dos herramientas:

1. **Conversor de ubicaciones a bins:** convierte ubicaciones de inventario (por ejemplo `2-L-11-2`) en registros de bins listos para exportar como CSV, con warehouse, zona, pasillo, bay, shelf, tipo de ubicación y Pick Sequence.
2. **Generador de etiquetas:** a partir de un Excel o CSV genera un PDF con etiquetas de 1.5" × 1.5" (una por página) con código de barras, listo para imprimir en un rollo de stickers.

Todo el procesamiento ocurre en el navegador; el servidor sólo entrega archivos estáticos.

## Features

**Conversor de ubicaciones**

- Entrada de ubicaciones en un textarea, una por línea. Formatos aceptados: `Zona-Pasillo-Bay-Shelf` (4 partes) o `Pasillo-Bay-Shelf` (3 partes, se asume zona `1`). Un `*` inicial se ignora para el parseo.
- Selección de cantidad de bins por ubicación: **6, 12 o 18** (default 18). Cada bin se nombra `<ubicación>-<n>`.
- Selección de **LocationType**: `PICK` (default) o `BINS`.
- Cada ubicación se genera siempre para los **3 warehouses virtuales**: `General / Wholesale`, `Ecommerce` y `Majors`.
- **Pick Sequence** buscado primero en `locations.csv` y, si no está, en una tabla interna embebida (`LOOKUP`). Si no hay dato para la ubicación, se busca la ubicación padre (p. ej. `2-A-1-0` para `2-A-1-0-1`). Herencia entre warehouses: Majors → Ecommerce → General / Wholesale.
- Avisos de ubicaciones sin Pick Sequence y de formatos no reconocidos.
- Descarga de un CSV `Locations_<primera ubicación>_3WH.csv` con columnas `INTERNAL_LOCATION_ID, Warehouse, Zone, Aisle, Bay, Shelf, LocationName, SimpleLocationName, LocationType, ExcludeVerificationWhenPicking, PickSequence`.
- Las etiquetas de los bins generados quedan precargadas en el generador (una por ubicación, sin repetir por warehouse).

**Generador de etiquetas**

- Carga de `.xlsx`, `.xls` o `.csv` por drag & drop o selector, o botón "Usar el CSV generado".
- **Modo producto** (detección automática por columnas): reconoce `title`, `style number`, `description`, `color`, `size`, `barcode`/`upc`/`ean`, `rrp`/`price`, `quantity`/`qty`/`cantidad`, `box`, `location`/`ubicación`. Genera tantas etiquetas por fila como indique la cantidad.
- **Modo simple:** si no hay columnas de producto, toma los códigos de la columna H (deduplicados).
- Detección del formato de código de barras: EAN13, UPC, EAN8 o CODE128 (fallback).
- Vista previa de columnas detectadas, filas y cantidad de etiquetas; PDF de 1.5" × 1.5" descargable.

## Tech Stack

- **Servidor:** Node.js ≥ 18, módulo nativo `http` (sin dependencias npm).
- **Frontend:** un único HTML con CSS/JS vanilla.
- **Librerías embebidas en el HTML (base64, sin CDN):** SheetJS (xlsx), jsPDF 4.2.1 y JsBarcode 3.12.3.
- **Datos:** `locations.csv` (fuente de Pick Sequence).
- **Testing:** script `selftest.js` que usa Chrome/Chromium en modo headless.
- **Hosting previsto:** Railway (según la descripción de `package.json`).

## Architecture

- `server.js` es un servidor HTTP estático que escucha en `0.0.0.0:$PORT` (default `3000`):
  - `/` sirve `Conversor_ubicaciones_bins.html`.
  - `/locations.csv` se resuelve primero junto al servidor y, si no existe, en el directorio padre del repo.
  - Cualquier otro archivo del directorio del repo se sirve con su MIME type; se bloquean rutas fuera del directorio (403).
- `Conversor_ubicaciones_bins.html` contiene toda la lógica: al cargar hace `fetch('locations.csv')` para armar la tabla de Pick Sequence por warehouse; si falla, usa la tabla interna. La generación del CSV y del PDF ocurre en el navegador (Blob + descarga).

## Prerequisites

- Node.js 18 o superior.
- Para `npm run check`: Google Chrome, Chromium, Edge o Brave instalado (o su ruta en `CHROME_PATH`).

## Installation

No hay dependencias npm que instalar (`package.json` no declara dependencias). Basta con clonar el repo.

## Configuration

| Variable | Descripción |
|---|---|
| `PORT` | Puerto del servidor HTTP (default `3000`). |
| `CHROME_PATH` | (Sólo `selftest.js`) ruta al ejecutable de Chrome/Chromium si no está en las rutas conocidas. |

Configuración de datos:

- `locations.csv`: export de ubicaciones con al menos las columnas `Warehouse`, `LocationName`, `SimpleLocationName` y `PickSequence`. Actualizarlo para refrescar los Pick Sequence.
- Los warehouses (`WAREHOUSES`) y la tabla interna `LOOKUP` están definidos dentro del HTML.

## Usage

```bash
npm start
```

Abrir `http://localhost:3000` y:

1. Verificar el mensaje de origen de Pick Sequence (cantidad de ubicaciones leídas de `locations.csv` por warehouse).
2. Elegir cantidad de bins (6/12/18) y tipo (PICK/BINS).
3. Pegar las ubicaciones (una por línea) y presionar **"Generar y descargar CSV"**; se descarga sólo el CSV.
4. En **Generador de etiquetas**, presionar **"Generar PDF"** para las etiquetas de los bins, o subir otro Excel/CSV.

También se puede abrir `Conversor_ubicaciones_bins.html` directamente, aunque la carga de `locations.csv` requiere servirlo por HTTP (si falla, se usa la tabla interna).

## Project Structure

```
.
├── Conversor_ubicaciones_bins.html  # App completa (UI, lógica, librerías embebidas y tabla LOOKUP)
├── locations.csv                    # Fuente de PickSequence por warehouse/ubicación
├── server.js                        # Servidor HTTP estático (Node, sin dependencias)
├── selftest.js                      # Prueba end-to-end con navegador headless
├── selftest-etiquetas.pdf           # PDF de salida generado por una corrida de selftest
└── package.json                     # Scripts start/check, engines node >= 18
```

## API Documentation

No expone una API: el servidor sólo sirve archivos estáticos (`GET /` y `GET /<archivo>`). No consume APIs externas.

## Testing

No hay tests unitarios. Hay una prueba de punta a punta:

```bash
npm run check                 # usa la ubicación por defecto 2-L-11-0
node selftest.js 3-A-4-1      # ubicación personalizada
```

`selftest.js` levanta un servidor temporal, inyecta un arnés en memoria en el HTML (el archivo no se modifica), abre la app en Chrome headless, genera el CSV y el PDF de etiquetas, imprime un log del resultado y guarda el PDF en `selftest-etiquetas.pdf`. Termina con código 0 si se generó el PDF, 1 si falló o hubo timeout (120 s).

## Deployment

La descripción de `package.json` indica que se sirve como app web en Railway; no hay `Procfile`, `railway.json` ni Dockerfile en el repo. El arranque es `npm start` (`node server.js`) y el servidor toma el puerto de `PORT`. No hay paso de build.
