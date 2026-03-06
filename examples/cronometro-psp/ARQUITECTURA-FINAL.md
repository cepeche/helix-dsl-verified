# Arquitectura Final - Mi Cronómetro PSP

## Resumen Ejecutivo

Se ha implementado una **aplicación web cliente-servidor** para seguimiento de tiempo, con backend PHP + SQLite y frontend JavaScript vanilla.

## Stack Tecnológico Final

### Backend
- **Lenguaje**: PHP 7.4+
- **Base de datos**: SQLite 3
- **Arquitectura**: API REST
- **Patrón**: MVC simplificado

### Frontend
- **HTML5** + **CSS3** (sin frameworks)
- **JavaScript ES6+** (vanilla, sin dependencias)
- **Patrón**: Estado global + renderizado reactivo

### Comunicación
- **Protocolo**: HTTP/REST
- **Formato**: JSON
- **CORS**: Habilitado para desarrollo local

## Estructura de Archivos

```
mi-cronometro-psp/
├── backend/
│   ├── index.php                # Router principal
│   ├── config.php               # Configuración + helpers
│   ├── db/
│   │   ├── database.php         # Singleton PDO
│   │   └── schema.sql           # DDL + datos iniciales
│   └── api/
│       ├── actividades.php      # CRUD actividades
│       ├── tipos-tarea.php      # CRUD tipos de tarea
│       └── sesiones.php         # Gestión de sesiones
│
├── frontend/
│   ├── index.html               # HTML estructural
│   ├── css/
│   │   └── styles.css           # Estilos responsive
│   └── js/
│       ├── api-client.js        # Cliente HTTP
│       └── app.js               # Lógica de negocio
│
├── data/
│   └── .gitkeep                 # cronometro.db se genera aquí
│
└── scripts/
    └── deploy-nas.sh            # Script de despliegue
```

## Modelo de Datos

### Tabla: `actividades`
Proyectos o categorías de trabajo.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | TEXT PK | Slug del nombre |
| nombre | TEXT | Nombre legible |
| color | TEXT | Color hex (#667eea) |
| created_at | INTEGER | Timestamp Unix |
| archived | INTEGER | 0=activa, 1=archivada |

### Tabla: `tipos_tarea`
Tipos de tarea que pueden usarse en múltiples actividades.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | TEXT PK | Slug del nombre |
| nombre | TEXT | "Codificar", "Reuniones" |
| icono | TEXT | Emoji (💻, 📝) |
| actividades_permitidas | TEXT | JSON array de IDs |
| usos_7d | INTEGER | Contador de uso |
| created_at | INTEGER | Timestamp Unix |

### Tabla: `tareas`
Tareas expandidas (tipo_tarea × actividad).

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | TEXT PK | "codificar_px" |
| tipo_tarea_id | TEXT FK | Referencia a tipos_tarea |
| actividad_id | TEXT FK | Referencia a actividades |

### Tabla: `sesiones`
Períodos de tiempo trabajados.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | TEXT PK | ID generado |
| tarea_id | TEXT FK | Referencia a tareas |
| inicio | INTEGER | Timestamp inicio |
| fin | INTEGER | Timestamp fin (NULL=activa) |
| duracion | INTEGER | Segundos (calculado) |
| notas | TEXT | Notas opcionales |

## API REST Endpoints

### Actividades

#### `GET /api/actividades`
Listar todas las actividades activas.

**Response:**
```json
[
  {
    "id": "px",
    "nombre": "Proyecto X",
    "color": "#667eea",
    "created_at": 1676476800,
    "archived": 0
  }
]
```

#### `POST /api/actividades`
Crear nueva actividad.

**Request:**
```json
{
  "nombre": "Proyecto Z",
  "color": "#43e97b"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Actividad creada",
  "data": { "id": "proyecto-z", ... }
}
```

### Tipos de Tarea

#### `GET /api/tipos-tarea`
Listar todos los tipos de tarea.

**Response:**
```json
[
  {
    "id": "codificar",
    "nombre": "Codificar",
    "icono": "💻",
    "actividades_permitidas": ["px", "py", "ps"],
    "usos_7d": 15,
    "created_at": 1676476800
  }
]
```

#### `POST /api/tipos-tarea`
Crear nuevo tipo de tarea.

**Request:**
```json
{
  "nombre": "Diseñar",
  "icono": "🎨",
  "actividades_permitidas": ["px", "py"]
}
```

### Sesiones

#### `GET /api/sesiones?fecha=hoy`
Obtener sesiones de una fecha.

**Params:**
- `fecha`: `hoy`, `ayer`, o `YYYY-MM-DD`

#### `GET /api/sesiones?action=activa`
Obtener sesión activa actual (o null).

#### `GET /api/sesiones?action=estadisticas&fecha=hoy`
Estadísticas de tiempo del día.

**Response:**
```json
{
  "tiempo_total": 28800,
  "sesion_activa_tiempo": 3600,
  "por_tipo_tarea": [...],
  "por_tarea": [...]
}
```

#### `POST /api/sesiones?action=iniciar`
Iniciar nueva sesión (detiene la anterior automáticamente).

**Request:**
```json
{
  "tarea_id": "codificar_px"
}
```

#### `POST /api/sesiones?action=detener`
Detener sesión activa.

## Flujo de Datos

### Inicialización de la app

```
Usuario → Frontend
├─ Cargar index.html
├─ Cargar styles.css
├─ Cargar api-client.js
└─ Cargar app.js
    ├─ inicializarApp()
    ├─ GET /api/actividades
    ├─ GET /api/tipos-tarea
    ├─ GET /api/sesiones?fecha=hoy
    ├─ GET /api/sesiones?action=activa
    └─ renderizar UI
```

### Iniciar sesión de trabajo

```
Usuario click tarea
    ↓
seleccionarTipoTarea(id)
    ↓
¿Múltiples actividades?
├─ No → iniciarTarea(tarea_id)
└─ Sí → mostrarModal()
             ↓
         Usuario elige actividad
             ↓
         iniciarTarea(tarea_id)
             ↓
         POST /api/sesiones?action=iniciar
             ↓
         Backend:
         ├─ Detener sesión anterior
         ├─ Crear nueva sesión
         └─ Responder con datos
             ↓
         Frontend actualiza estado
             ↓
         Renderizar UI
```

### Timer en tiempo real

```
setInterval(1000ms)
    ↓
actualizarTimer()
├─ Calcular tiempo transcurrido
├─ Formatear HH:MM:SS
└─ Actualizar DOM

actualizarTotalHoy()
├─ Sumar sesiones cerradas
├─ Sumar sesión activa
└─ Actualizar DOM
```

## Decisiones de Diseño

### ¿Por qué SQLite?
- ✅ Sin servidor de BD separado
- ✅ Archivo único, fácil de respaldar
- ✅ Rendimiento excelente para uso personal
- ✅ Portable entre sistemas

### ¿Por qué PHP vanilla?
- ✅ Disponible en WD My Cloud EX2 Ultra (PHP-FPM nativo)
- ✅ Sin dependencias externas
- ✅ Fácil de mantener
- ✅ Bajo consumo de recursos

### ¿Por qué JavaScript vanilla?
- ✅ Sin build step ni transpilación
- ✅ Carga instantánea
- ✅ Mantenimiento mínimo
- ✅ Compatible con cualquier navegador moderno

### ¿Por qué separar tipos_tarea de tareas?
Para permitir que un tipo de tarea (ej: "Codificar") se use en múltiples actividades (Proyecto X, Proyecto Y) sin duplicar datos.

## Seguridad

### Implementado
- ✅ Validación de parámetros requeridos
- ✅ Prepared statements (previene SQL injection)
- ✅ CORS configurado para red local
- ✅ Sanitización de IDs mediante slugify

### No implementado (no crítico para uso personal)
- ⚠️ Autenticación de usuarios
- ⚠️ Rate limiting
- ⚠️ HTTPS (usar reverse proxy si es necesario)

## Rendimiento

### Backend
- **Consultas optimizadas** con índices en SQLite
- **Sin N+1 queries**: JOINs eficientes
- **Transacciones** para operaciones atómicas

### Frontend
- **Render selectivo**: Solo actualiza grids modificados
- **Timer eficiente**: setInterval sin reflows innecesarios
- **Sin frameworks**: Carga en < 100ms

## Testing

### Backend
Probar endpoints con curl:
```bash
curl http://localhost:8000/api/health
curl http://localhost:8000/api/actividades
```

### Frontend
1. Abrir DevTools (F12)
2. Verificar red (tab Network)
3. Verificar consola (tab Console)

## Despliegue

### Local (desarrollo)
```bash
cd backend && php -S localhost:8000
cd frontend && php -S localhost:8080
```

### NAS (WD My Cloud EX2 Ultra)
```bash
./scripts/deploy-nas.sh
```

## Mantenimiento

### Backup
```bash
cp data/cronometro.db backups/cronometro-$(date +%Y%m%d).db
```

### Limpieza de sesiones antiguas
```sql
-- Opcional: archivar sesiones > 1 año
DELETE FROM sesiones WHERE inicio < strftime('%s', 'now', '-1 year');
VACUUM;
```

## Extensibilidad Futura

### Fácil de añadir
- Exportación CSV/JSON (frontend)
- Gráficos con Chart.js (frontend)
- Vista de historial (frontend + endpoint)

### Requiere cambios moderados
- Multi-usuario (auth + permisos)
- Sincronización multi-dispositivo (WebSocket o polling)
- Modo offline (Service Worker + IndexedDB)

### Arquitectura alternativa (futuro)
Si el proyecto crece:
- Migrar a PostgreSQL
- Usar framework PHP (Laravel/Symfony)
- Frontend con React/Vue
- Containerización con Docker

## Conclusión

**Arquitectura elegida**: Cliente-servidor clásico con REST API.

**Ventajas**:
- Simple de entender y mantener
- Funciona en cualquier servidor con PHP
- Multi-dispositivo out of the box
- Datos centralizados en SQLite

**Limitaciones aceptables**:
- No funciona offline (requiere conexión al NAS)
- Sin sincronización en tiempo real (polling manual con F5)
- Sin soporte para múltiples usuarios concurrentes

**Veredicto**: ✅ Perfecta para el caso de uso (tracking personal en red local).
