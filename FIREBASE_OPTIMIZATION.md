# Firebase Optimization — CESUM Asistencia

## Plan objetivo: Spark (gratuito)
| Límite | Diario |
|--------|--------|
| Lecturas | 50 000 |
| Escrituras | 20 000 |
| Eliminaciones | 20 000 |

---

## Estrategias implementadas

### 1. Persistencia offline (IndexedDB)
`initializeFirestore` con `persistentLocalCache` + `persistentMultipleTabManager`.

- **Efecto:** Cada documento cacheado en IndexedDB se sirve localmente en visitas siguientes (0 lecturas de servidor).
- **Aplica a:** Toda la app automáticamente.

### 2. Caché en memoria con TTL
`Map`-based cache (`cacheGet / cacheSet / cacheDel`) con expiración configurable.

| Colección | TTL | Justificación |
|-----------|-----|---------------|
| `asistencia` | 5 min | Cambia varias veces al día |
| `reportes` | 5 min | Baja frecuencia de cambio |
| `novedades` | 2 min | Puede cambiar con frecuencia |
| `noticias` | 10 min | Muy estable |
| `citas` | 2 min | Puede cambiar durante la jornada |

Todas las funciones de lectura comparten el mismo caché. El primer llamado paga el costo; los siguientes dentro del TTL devuelven `0 lecturas de servidor`.

### 3. Funciones de lectura compartidas (cache-first)
| Función | Colección | Sin caché | Con caché (warm) |
|---------|-----------|-----------|------------------|
| `fetchAsistenciaTodos()` | `asistencia` | ~200 lect | 0 |
| `fetchReportesTodos()` | `reportes` | N docs | 0 |
| `fetchNovedadesTodos()` | `novedades` | N docs | 0 |
| `fetchNoticiasAll()` | `noticias` | N docs | 0 |
| `fetchCitasAll()` | `citas` | N docs | 0 |

### 4. Filtrado local en `verRecordEstudiante`
Antes: 4 queries Firestore individuales por estudiante.
Después: 4 lecturas de caché + `.filter()` en memoria.

### 5. Carga paralela en `cargarDatosIniciales`
`Promise.all([fetchCitasAll(), refrescarNovedadesActivas(), cargarNoticiasActivas()])`
Reduce la latencia de inicio de sesión de ~3× serie a ~1× paralelo.

### 6. `writeBatch` en importación CSV
Antes: N llamadas `addDoc` individuales.
Después: 1 `batch.commit()` por cada 400 documentos.
**Efecto:** Atómica, menor latencia, sin límite de reintentos por documento.

### 7. Fuerza-refresco explícito
Parámetro `force = true` en funciones compartidas.
- Botones "Refrescar" lo usan → datos frescos del servidor.
- Cargas automáticas usan `force = false` → caché.

### 8. Contadores de lecturas/escrituras (DEBUG_MODE)
```javascript
const DEBUG_DB = false; // cambia a true para depurar
```
Cuando está activo imprime en consola: `📖 +N lect (total: N)`.

---

## Comparación antes / después

### Dashboard (inicio de sesión)

| Operación | Antes | Después |
|-----------|-------|---------|
| `asistencia` (todos los días) | ~200 lect × 2 = 400 | 1 fetch (caché 5 min) |
| `novedades` activas | getDocs filtrado | caché warm = 0 |
| `noticias` activas | getDocs | caché warm = 0 |
| **Total aprox.** | **~410 lect** | **~1-5 lect** |

### Estadísticas

| Operación | Antes | Después |
|-----------|-------|---------|
| `asistencia` | ~200 lect | 0 (caché) |
| Loop por estudiante | N lect | 0 (caché) |
| **Total aprox.** | **~200+ lect** | **0 lect** |

### Ver Record de Estudiante

| Operación | Antes | Después |
|-----------|-------|---------|
| `asistencia` query | ~200 lect | 0 (caché) |
| `reportes` query | N lect | 0 (caché) |
| `novedades` query | N lect | 0 (caché) |
| `citas` query | N lect | 0 (caché) |
| **Total aprox.** | **~200+ lect** | **0 lect** |

### Estimación con 50 docentes simultáneos

| Escenario | Lecturas/día (antes) | Lecturas/día (después) |
|-----------|---------------------|----------------------|
| 50 logins + dashboard | ~20 500 | ~250 |
| 50 × stats | ~10 000 | 0 |
| 50 × ver records (5 c/u) | ~50 000+ | 0 |
| Escrituras normales | ~200 | ~200 |
| **TOTAL** | **>>50 000 ⚠️** | **~450 ✅** |

---

## Límites por funcionalidad

| Funcionalidad | Lecturas por acción | Escrituras por acción |
|---------------|--------------------|-----------------------|
| Login | 1 (perfil usuario) | 0 |
| Dashboard | 0-5 (primer login del día) | 0 |
| Marcar asistencia | 0 (caché) + 1 save | 1 |
| Ver estadísticas | 0 (caché) | 0 |
| Ver record estudiante | 0 (caché) | 0 |
| Crear reporte | 0 | 1 |
| Crear novedad | 0 | 1 |
| Agendar cita | 0 | 1 |
| Publicar noticia | 0 | 1 |
| Importar CSV (100 est.) | 0 | 1 batch |
| Refrescar datos (forzado) | ~5-10 | 0 |

---

## Seguridad (`firestore.rules`)

- Solo usuarios autenticados con perfil `activo != false` pueden leer datos.
- Docentes pueden crear registros; solo admins o el creador pueden editar/eliminar.
- Solo admins pueden modificar usuarios, estudiantes y noticias.
- Todo lo que no tiene regla explícita está bloqueado por defecto.

---

## Índices compuestos (`firestore.indexes.json`)

Índices creados para las queries más frecuentes:

- `citas`: `estado + fecha`, `estudiante + fecha`
- `reportes`: `estudiante + fecha`
- `novedades`: `estudiante + inicio`, `activa + inicio`
- `noticias`: `inicio + fin`
- `usuarios`: `rol + activo`
- `actividades`: `grado + fecha`

---

## Monitoreo y alertas

### Activar debug en consola
```javascript
// En index.html, línea DEBUG_DB:
const DEBUG_DB = true;
```
Muestra cada lectura/escritura con totales acumulados en la sesión.

### Alertas recomendadas en Firebase Console
1. **Firebase Console → Usage → Firestore**
   Configurar alerta en 40 000 lecturas/día (80% del límite).

2. **Cloud Monitoring** (si se migra a Blaze):
   Alerta en `firestore.googleapis.com/document/read_count > 40000/day`.

### Señales de caché funcionando
- Segunda carga del dashboard: 0 lecturas nuevas en consola.
- Ver record de estudiante por segunda vez: respuesta instantánea, 0 lecturas.
- Icono de red en DevTools: sin requests a Firestore al navegar entre vistas.

---

## Archivos modificados

| Archivo | Cambio |
|---------|--------|
| `index.html` | Caché, batches, funciones compartidas, `_getDocs`/`_getDoc` wrappers |
| `firestore.rules` | Reglas de seguridad por rol |
| `firestore.indexes.json` | Índices compuestos para queries frecuentes |

---

*Optimización aplicada: 2026-03-17*
