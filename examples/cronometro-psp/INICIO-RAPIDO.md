# 🚀 Inicio Rápido - Mi Cronómetro PSP

## Para probar YA en Windows

### Opción 1: XAMPP (Recomendado)

1. **Instala XAMPP** (si no lo tienes): https://www.apachefriends.org/

2. **Copia el proyecto** a la carpeta de XAMPP:
```cmd
xcopy /E /I C:\ProyectosClaude\mi-cronometro-psp C:\xampp\htdocs\mi-cronometro-psp
```

3. **Inicia Apache** desde el Panel de Control de XAMPP

4. **Abre en tu navegador**:
```
http://localhost/mi-cronometro-psp/frontend
```

¡Listo! Ya puedes empezar a trackear tiempo.

### Opción 2: PHP Built-in Server

1. **Descarga PHP** (si no lo tienes): https://windows.php.net/download/

2. **Añade PHP a tu PATH** (opcional)

3. **Abre DOS terminal en el proyecto:**
```cmd
cd C:\ProyectosClaude\mi-cronometro-psp\backend
php -S localhost:8000
```

4. **En OTRA terminal:**
```cmd
cd C:\ProyectosClaude\mi-cronometro-psp\frontend
php -S localhost:8080
```

5. **Abre en tu navegador**:
```
http://localhost:8080
```

---

## Para usar en tu móvil (misma red WiFi)

### Paso 1: Encuentra la IP de tu PC

En Windows:
```cmd
ipconfig
```
Busca tu IP local (ejemplo: `192.168.1.100`)

### Paso 2: Configura la URL en el código

Edita `frontend/js/api-client.js`:
```javascript
const API_CONFIG = {
    baseURL: 'http://192.168.1.100:8000'  // Tu IP aquí
};
```

### Paso 3: Inicia los servidores

```cmd
cd backend
php -S 0.0.0.0:8000

# En otra terminal
cd frontend
php -S 0.0.0.0:8080
```

### Paso 4: Accede desde tu móvil

Abre en el navegador de tu móvil:
```
http://192.168.1.100:8080
```

---

## Para desplegar en tu NAS (WD My Cloud EX2 Ultra)

### Pre-requisitos
- WD My Cloud EX2 Ultra con Apache nativo
- PHP-FPM habilitado (PHP 7.4+)
- SSH activado

### Pasos

1. **Edita el script de deploy** con tus datos:
```bash
nano scripts/deploy-nas.sh
```
Cambia:
- `NAS_USER="tu_usuario"`
- `NAS_HOST="192.168.1.71"` (tu IP del NAS)

2. **Ejecuta el script:**
```bash
chmod +x scripts/deploy-nas.sh
./scripts/deploy-nas.sh
```

3. **Configura Web Station:**
   - Abre Web Station en DSM
   - Crea un Virtual Host
   - Apunta a `/volume1/web/mi-cronometro-psp/frontend`
   - Habilita PHP 7.4+

4. **Configura la URL del backend:**

Edita `frontend/js/api-client.js` en el NAS:
```javascript
const API_CONFIG = {
    baseURL: 'http://192.168.1.71/mi-cronometro-psp/backend'
};
```

5. **Accede desde cualquier dispositivo:**
```
http://192.168.1.71/mi-cronometro-psp/frontend
```

---

## Verificar que funciona

### 1. Backend health check

Abre en tu navegador:
```
http://localhost:8000/api/health
```

Deberías ver:
```json
{
  "status": "ok",
  "timestamp": 1676476800,
  "version": "1.0.0"
}
```

### 2. Ver actividades iniciales

```
http://localhost:8000/api/actividades
```

Deberías ver las 5 actividades predefinidas.

### 3. Frontend carga correctamente

Abre las DevTools del navegador (F12) y verifica:
- ✅ No hay errores en la consola
- ✅ En "Network" ves las llamadas a la API
- ✅ Las tarjetas de tareas se muestran

---

## Solución de Problemas Comunes

### "No se puede conectar al backend"

**Causa**: La URL del backend está mal configurada.

**Solución**: Edita `frontend/js/api-client.js` y ajusta `baseURL`.

### "Permission denied" al crear base de datos

**Causa**: El directorio `data/` no tiene permisos de escritura.

**Solución**:
```bash
chmod 777 data/
```

### El timer no funciona

**Causa**: JavaScript está deshabilitado o hay errores en consola.

**Solución**: Abre DevTools (F12) y revisa la pestaña Console.

### Las tareas no se guardan

**Causa**: El backend no está ejecutándose.

**Solución**: Verifica que el servidor PHP esté corriendo y responda en el puerto 8000.

---

## Próximos Pasos

Una vez que la app funcione:

1. **Prueba crear una nueva tarea** (botón +)
2. **Inicia una sesión** pulsando una tarjeta
3. **Cambia de tarea** pulsando otra tarjeta
4. **Verifica** que el tiempo se acumula correctamente
5. **Recarga la página** (F5) y comprueba que los datos persisten

---

## ¿Necesitas ayuda?

1. Lee el `README.md` completo
2. Revisa `ARQUITECTURA-FINAL.md` para entender cómo funciona
3. Consulta la consola del navegador (F12) para errores
4. Verifica que PHP tenga la extensión SQLite habilitada:
   ```bash
   php -m | grep sqlite
   ```

---

**¡Disfruta trackeando tu tiempo sin fricciones! ⏱️**
