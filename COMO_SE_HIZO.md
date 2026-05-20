# Cómo se construyó — Colegios de Bolivia (OSM)

## Resumen del sistema

Aplicación web de una sola página (`index.html`) que consulta la **Overpass API** de OpenStreetMap para mostrar en un mapa interactivo todos los colegios georeferenciados de Bolivia, con filtro por departamento y auto-actualización cada 5 minutos.

---

## Tecnologías y licencias

| Componente | Versión | Licencia |
|---|---|---|
| [Leaflet.js](https://leafletjs.com/) | 1.9.4 | BSD-2-Clause |
| [Leaflet.markercluster](https://github.com/Leaflet/Leaflet.markercluster) | 1.5.3 | MIT |
| OpenStreetMap tiles | — | ODbL 1.0 + CC-BY-SA |
| Overpass API | — | Uso libre (fair use) |

**Atribución requerida por ODbL:**
> © OpenStreetMap contributors, datos bajo licencia ODbL.
> La atribución aparece automáticamente en el mapa (esquina inferior derecha).

---

## Arquitectura

```
index.html (todo en un archivo)
│
├── HTML — estructura de UI (header, mapa, barra de estado, sidebar)
├── CSS  — diseño dark mode sin dependencias externas
└── JS   — lógica de datos + renderizado del mapa
     │
     ├── CONFIG           → URLs de Overpass, intervalo 5 min
     ├── DEPT_BBOX        → bounding boxes de los 9 departamentos
     ├── buildQuery()     → genera la consulta Overpass QL
     ├── fetchOverpass()  → fetch POST con fallback a espejo
     ├── parseElements()  → convierte nodos/vías/relaciones a {lat,lon,tags}
     ├── inferDept()      → asigna departamento por coordenadas (fallback)
     ├── renderMarkers()  → clusteriza marcadores en el mapa
     ├── applyFilter()    → filtra por departamento localmente
     ├── loadSchools()    → orquesta carga completa
     └── scheduleNextRefresh() → temporizador de 5 min con countdown
```

---

## Cómo funciona la consulta a OSM

### Protocolo: Overpass API (HTTP POST)

```
URL principal: https://overpass-api.de/api/interpreter
URL espejo:    https://overpass.kumi.systems/api/interpreter
Método:        POST
Content-Type:  application/x-www-form-urlencoded
Body:          data=<query en Overpass QL>
```

Si el servidor principal no responde, el sistema intenta automáticamente el espejo sin intervención del usuario.

### Consulta Overpass QL (ejemplo para Bolivia completa)

```overpassql
[out:json][timeout:60];
(
  node["amenity"="school"](-23.0,-70.5,-9.5,-57.0);
  node["amenity"="college"](-23.0,-70.5,-9.5,-57.0);
  node["building"="school"](-23.0,-70.5,-9.5,-57.0);
  way["amenity"="school"](-23.0,-70.5,-9.5,-57.0);
  way["amenity"="college"](-23.0,-70.5,-9.5,-57.0);
  relation["amenity"="school"](-23.0,-70.5,-9.5,-57.0);
);
out center qt;
```

**Explicación de cada parte:**

- `[out:json]` — respuesta en formato JSON
- `[timeout:60]` — máximo 60 segundos de espera en el servidor
- `node/way/relation[...]` — busca en los tres tipos de geometría OSM
- `"amenity"="school"` — etiqueta principal para colegios/escuelas
- `"amenity"="college"` — institutos y colegios secundarios
- `"building"="school"` — edificios etiquetados como colegio
- `(bbox)` — bounding box: sur, oeste, norte, este
- `out center qt` — devuelve el centroide de vías/relaciones + ordena por cuadrante (más rápido)

### Al filtrar por departamento

Se usa el bounding box específico del departamento en la consulta, reduciendo el tamaño de la respuesta y el tiempo de carga.

---

## Datos mostrados por colegio

Cada marcador popup muestra los tags OSM disponibles:

| Tag OSM | Descripción |
|---|---|
| `name` / `name:es` | Nombre del colegio |
| `amenity` | Tipo (school, college) |
| `addr:state` | Departamento |
| `operator` | Entidad gestora |
| `operator:type` | Público / privado |
| `levels` | Niveles educativos |
| `capacity` | Número de alumnos |
| `phone` | Teléfono |
| `website` | Sitio web |

Cada popup incluye un enlace directo a `openstreetmap.org/node/<id>` para verificar o editar el dato.

---

## Auto-actualización cada 5 minutos

```javascript
CONFIG.refreshInterval = 5 * 60 * 1000  // ms

// Al terminar cada carga:
refreshTimer = setTimeout(() => loadSchools(false), CONFIG.refreshInterval);

// Countdown visible en la barra de estado:
countdownTimer = setInterval(() => { /* actualiza "Próxima actualización: X:XX" */ }, 1000);
```

La actualización es **incremental por diseño**: al hacer la misma consulta a OSM, si alguien agregó un colegio nuevo en esos 5 minutos, aparecerá en el mapa en la siguiente carga. El cluster se reconstruye completo con los nuevos datos.

El usuario puede:
- Pausar/reanudar con el botón "⏸ Pausar auto"
- Forzar actualización inmediata con "↻ Actualizar ahora"

---

## Filtro por departamento

El filtro tiene dos niveles:

1. **Filtro local** (instantáneo): cuando ya hay datos cargados, filtra el array `allSchools` por coordenadas (función `inferDept`) o por tags `addr:state` / `is_in:department`.
2. **Re-consulta a OSM** (en paralelo): lanza también una consulta con el bounding box del departamento para tener datos frescos y más completos del área.

---

## Bounding boxes de departamentos

```
La Paz:      [-17.5, -70.0, -13.5, -67.5]
Cochabamba:  [-18.3, -66.8, -16.0, -64.5]
Santa Cruz:  [-21.5, -64.5, -13.5, -57.0]
Oruro:       [-21.5, -68.5, -17.0, -66.5]
Potosí:      [-23.0, -68.0, -18.5, -64.5]
Chuquisaca:  [-22.0, -65.5, -18.0, -63.0]
Tarija:      [-23.0, -65.2, -20.5, -62.5]
Beni:        [-16.0, -68.5,  -9.5, -60.0]
Pando:       [-13.0, -70.5,  -9.5, -65.5]
```

Formato: `[sur, oeste, norte, este]` (latitud/longitud decimal WGS84)

---

## Licencia y atribución OSM (obligatorio)

OpenStreetMap distribuye sus datos bajo la licencia **Open Database License (ODbL) 1.0**.  
Los tiles (imágenes del mapa) están bajo **CC-BY-SA 2.0**.

**Obligaciones legales cumplidas en este proyecto:**

1. ✅ Atribución visible en el mapa: `© OpenStreetMap contributors`
2. ✅ Enlace a la licencia ODbL en el control de atribución de Leaflet
3. ✅ Cada popup incluye enlace al nodo/vía original en OSM
4. ✅ Los datos no se redistribuyen ni almacenan localmente de forma permanente
5. ✅ Se usa la Overpass API pública con respeto al fair use (timeout razonable, sin bucles abusivos)

Más información: https://www.openstreetmap.org/copyright

---

## Cómo abrir el proyecto

1. Abrir la carpeta `bolivia-colegios-osm` en el Escritorio
2. Hacer doble clic en `index.html`
3. Se abre en el navegador — requiere conexión a internet para cargar datos OSM

> No requiere servidor, npm, ni instalación alguna. Es HTML puro.

---

## Posibles mejoras futuras

- Agregar geocodificación inversa con Nominatim para departamentos más precisos
- Exportar los datos a CSV / GeoJSON
- Agregar capas adicionales (hospitales, postas, etc.)
- Cachear respuestas en `localStorage` para funcionar offline temporalmente
- Usar `Service Worker` para notificaciones de nuevos colegios detectados
