# Mi Cronómetro PSP

Aplicación web para seguimiento de tiempo dedicado a actividades y proyectos, con arquitectura cliente-servidor.

## 🚀 Características

- ⏱️ **Timer en tiempo real** con seguimiento de sesiones activas
- 📊 **Pestañas por actividad** + pestaña "Frecuentes" con tareas más usadas
- 🎯 **Tipos de tarea multi-actividad** (ej: "Codificar" puede usarse en múltiples proyectos)
- 💾 **Persistencia con SQLite** - tus datos se guardan automáticamente
- 📱 **Responsive** - funciona en móvil y desktop
- 🔄 **Multi-dispositivo** - accede desde cualquier dispositivo en tu red local

## 📁 Estructura del proyecto

```
mi-cronometro-psp/
├── backend/                    # API REST en PHP
│   ├── api/
│   │   ├── actividades.php    # CRUD actividades
│   │   ├── tipos-tarea.php    # CRUD tipos de tarea
│   │   └── sesiones.php       # Gestión de sesiones
│   ├── db/
│   │   ├── database.php       # Conexión SQLite
│   │   └── schema.sql         # Estructura de BD
│   ├── config.php             # Configuración
│   └── index.php              # Router principal
│
├── frontend/                   # Interfaz web
│   ├── index.html
│   ├── css/styles.css
│   └── js/
│       ├── api-client.js      # Cliente API
│       └── app.js             # Lógica de la app
│
├── data/                       # Base de datos
│   └── cronometro.db          # SQLite (se crea automáticamente)
│
└── scripts/                    # Scripts de utilidad
    └── deploy-nas.sh          # Despliegue en NAS
```

## 🛠️ Instalación Local (Windows)

### Requisitos previos
- PHP 7.4+ con extensión SQLite3
- Servidor web (XAMPP, Laragon, o PHP built-in server)

### Opción 1: PHP Built-in Server (más rápido para pruebas)

```bash
# Desde el directorio del proyecto
cd C:\ProyectosClaude\mi-cronometro-psp

# Iniciar backend
cd backend
php -S localhost:8000

# En otra terminal, iniciar frontend
cd ../frontend
php -S localhost:8080
```

Luego abre en tu navegador: `http://localhost:8080`

### Opción 2: XAMPP/Laragon

1. Copia el proyecto a tu carpeta `htdocs/` o `www/`
2. Configura un virtual host o accede directamente:
   - Backend: `http://localhost/mi-cronometro-psp/backend`
   - Frontend: `http://localhost/mi-cronometro-psp/frontend`

3. Edita `frontend/js/api-client.js` y ajusta la URL del backend si es necesario:
```javascript
const API_CONFIG = {
    baseURL: 'http://localhost/mi-cronometro-psp/backend'
};
```

## 🏠 Instalación en NAS (WD My Cloud EX2 Ultra)

### Requisitos
- WD My Cloud EX2 Ultra (Linux Debian-based, Apache nativo)
- PHP-FPM habilitado (PHP 7.4+)
- Acceso SSH activado

### Pasos de instalación

1. **Conectar por SSH a tu NAS:**
```bash
ssh usuario@192.168.1.71
```

2. **Crear directorio para la aplicación:**
```bash
cd /volume1/web
mkdir mi-cronometro-psp
```

3. **Subir archivos al NAS:**

Desde tu PC Windows, usa WinSCP, FileZilla o scp:
```bash
scp -r C:\ProyectosClaude\mi-cronometro-psp/* usuario@192.168.1.71:/volume1/web/mi-cronometro-psp/
```

4. **Configurar permisos:**
```bash
ssh usuario@192.168.1.71
cd /volume1/web/mi-cronometro-psp
chmod -R 755 .
chmod -R 777 data/  # Importante: SQLite necesita escribir aquí
```

5. **Configurar Web Station:**
   - Abre Web Station en DSM
   - Crea un nuevo "Web Portal" o "Virtual Host"
   - Apunta a `/volume1/web/mi-cronometro-psp/frontend`
   - Habilita PHP 7.4+

6. **Ajustar URL de la API:**

Edita `frontend/js/api-client.js`:
```javascript
const API_CONFIG = {
    baseURL: 'http://192.168.1.71/mi-cronometro-psp/backend'
};
```

7. **Acceder a la aplicación:**
   - Desde tu red local: `http://192.168.1.71/mi-cronometro-psp/frontend`
   - Desde móvil: `http://192.168.1.71/mi-cronometro-psp/frontend`

## 🧪 Probar el backend

Verifica que el backend funcione correctamente:

```bash
# Health check
curl http://localhost:8000/api/health

# Listar actividades
curl http://localhost:8000/api/actividades

# Listar tipos de tarea
curl http://localhost:8000/api/tipos-tarea

# Ver sesión activa
curl http://localhost:8000/api/sesiones?action=activa
```

## 📡 API Endpoints

### Actividades
- `GET /api/actividades` - Listar todas las actividades
- `POST /api/actividades` - Crear actividad
  ```json
  { "nombre": "Proyecto Z", "color": "#667eea" }
  ```

### Tipos de Tarea
- `GET /api/tipos-tarea` - Listar todos los tipos de tarea
- `POST /api/tipos-tarea` - Crear tipo de tarea
  ```json
  {
    "nombre": "Diseñar",
    "icono": "🎨",
    "actividades_permitidas": ["px", "py"]
  }
  ```

### Sesiones
- `GET /api/sesiones?fecha=hoy` - Sesiones del día
- `GET /api/sesiones?action=activa` - Sesión activa actual
- `GET /api/sesiones?action=estadisticas&fecha=hoy` - Estadísticas del día
- `POST /api/sesiones?action=iniciar` - Iniciar sesión
  ```json
  { "tarea_id": "codificar_px" }
  ```
- `POST /api/sesiones?action=detener` - Detener sesión activa

## 🎨 Uso de la aplicación

### Iniciar una sesión
1. Pulsa sobre la tarjeta de la tarea que quieres realizar
2. Si la tarea puede usarse en múltiples actividades, selecciona una
3. El timer empezará automáticamente

### Cambiar de tarea
- Simplemente pulsa otra tarea
- La sesión anterior se cerrará y se guardará automáticamente
- La nueva sesión iniciará

### Crear nueva tarea
1. Pulsa el botón `+` flotante
2. Ingresa nombre, selecciona icono y actividades
3. La tarea aparecerá disponible inmediatamente

### Crear nueva actividad
1. Pulsa el botón ⚙️ (settings)
2. Selecciona "Nueva actividad"
3. Ingresa nombre y selecciona color
4. Aparecerá una nueva pestaña

## 💾 Backup de datos

La base de datos SQLite está en `data/cronometro.db`. Para hacer backup:

```bash
# Copiar archivo de base de datos
cp data/cronometro.db data/cronometro-backup-$(date +%Y%m%d).db
```

En NAS, puedes configurar una tarea programada en DSM para hacer backups automáticos.

## 🔧 Troubleshooting

### El frontend no carga los datos
1. Abre las DevTools del navegador (F12)
2. Ve a la pestaña "Console"
3. Si ves errores de CORS o conexión:
   - Verifica que el backend esté ejecutándose
   - Revisa la URL en `frontend/js/api-client.js`

### Error "Permission denied" al crear base de datos
```bash
# Dar permisos al directorio data/
chmod 777 data/
```

### El timer no se actualiza
- Refresca la página (F5)
- Verifica que JavaScript esté habilitado en tu navegador

### No se guardan las sesiones
- Revisa que el archivo `data/cronometro.db` exista
- Verifica permisos de escritura en la carpeta `data/`

## 🚀 Próximas mejoras

- [ ] Exportar datos a CSV/JSON
- [ ] Vista de historial de sesiones
- [ ] Gráficos de tiempo por actividad
- [ ] Estimaciones vs tiempo real
- [ ] Notificaciones de inactividad
- [ ] Modo offline con sincronización

## 📝 Licencia

MIT License - Úsalo libremente para tu tracking personal.

---

**Desarrollado con ❤️ para ingenieros que quieren trackear su tiempo sin fricciones**
