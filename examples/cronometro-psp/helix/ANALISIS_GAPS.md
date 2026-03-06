# ANALISIS_GAPS.md — Lo que el DSL no cubre (todavía)

**Generado por**: ejercicio de especificación completa del cronómetro PSP  
**Fecha**: 6 de marzo de 2026  
**Método**: escribir todos los `.helix` del sistema real y anotar cada punto de fricción

---

## Resumen

Al intentar especificar el cronómetro completo encontramos **8 gaps** en el
DSL actual. No todos tienen el mismo peso. Los ordenamos de mayor a menor
impacto en la expresividad del lenguaje.

---

## GAP-1: Parámetros de entrada en contextos (CRÍTICO)

**Dónde aparece**: ModalComentario, ModalEditarTarea, ModalEditarActividad,
ModalSeleccionActividad.

**El problema**: cuando ModoEdicion hace `on tap -> abrirEditarTarea`, necesita
pasar el `tipoId` de la tarjeta que se pulsó. Sin esto, el modal no sabe
*qué* está editando. En `app.js` esto se resuelve con `AppState.tareaIdPendiente`
— exactamente el estado implícito que helix pretende eliminar.

**Impacto en app.js actual**: la variable `AppState.tareaIdPendiente` es el
ejemplo canónico. Un estado global mutable para transferir datos entre un
contexto y el siguiente.

**Propuesta de sintaxis**:
```
context ModalEditarTarea:
    input:
        tipoTareaId: Id            -- obligatorio
        actividadId: Id?           -- opcional (con ?)
```

Los parámetros de entrada viajan en la transición:
```
transitions:
    on abrirEditarTarea(tipoId) -> ModalEditarTarea with tipoTareaId: tipoId
```

**Por qué es crítico**: sin esto, el compilador no puede verificar que todos
los contextos que abren un modal le pasan los datos que necesita. La
verificación formal queda incompleta.

---

## GAP-2: Contextos overlay (CRÍTICO)

**Dónde aparece**: todos los modales y el menú de configuración.

**El problema**: los modales no reemplazan el contexto base — se superponen
a él. Cuando `ModalComentario` se cierra, el sistema vuelve a `ModoNormal`
(o a `ModoEdicion`, dependiendo de quién lo abrió). El DSL actual no tiene
concepto de "pila de contextos".

La diferencia con las transiciones normales es fundamental:
- Transición normal: `ModoNormal → ModoEdicion`. ModoNormal desaparece.
- Transición a overlay: `ModoNormal → (ModoNormal + ModalComentario)`.
  ModoNormal sigue activo debajo.

**Impacto en app.js actual**: todos los modales usan `classList.add('active')`
y `classList.remove('active')` — el modal se superpone, no reemplaza. El
flag `modoEdicion` persiste mientras el modal está abierto.

**Propuesta de sintaxis**:
```
system CronometroPSP:
    overlays:              -- nueva cláusula
        MenuConfiguracion
        ModalComentario
        ...
```

Y en las transiciones, el cierre usa un token especial:
```
transitions:
    on cancelar -> [cerrar_overlay]    -- regresa al contexto base que había debajo
```

**Corolario**: cuando un overlay transiciona a otro overlay (MenuConfiguracion
→ ModalCrearActividad), ¿se apilan o el primero se cierra? En app.js el menú
se cierra. El DSL necesita distinguir `[cerrar_y_abrir: NuevoOverlay]` de
`[apilar: NuevoOverlay]`.

---

## GAP-3: Contextos concurrentes (IMPORTANTE)

**Dónde aparece**: SesionActiva coexiste con ModoNormal o ModoEdicion.

**El problema**: `SesionActiva` no es ni un contexto base ni un overlay. Es
un estado ortogonal que puede estar activo o no independientemente del modo.
Cuando hay sesión activa, el timer muestra datos; cuando no, muestra `--:--`.
Esto ocurre tanto en ModoNormal como en ModoEdicion.

El modelo actual de `composition: prioridad` asume que los contextos son
alternativos. SesionActiva no es una alternativa a ModoNormal — es una
dimensión diferente.

**Propuesta de sintaxis**:
```
system CronometroPSP:
    concurrent:            -- nueva cláusula (activos en paralelo con el base)
        SesionActiva       -- se activa/desactiva independientemente
```

Y en la desactivación:
```
context SesionActiva:
    transitions:
        on sesionFinalizada -> [desactivar]   -- sale del concurrent sin cambiar el base
```

---

## GAP-4: Roles condicionales según contexto concurrente (IMPORTANTE)

**Dónde aparece**: `checkbox_sustituir` en ModalComentario, que solo existe
cuando SesionActiva está activo.

**El problema**: cuando ModalComentario se abre sin sesión activa, el checkbox
"Sustituye a la tarea en curso" no existe en el DOM. Cuando se abre con sesión
activa, existe. Este es un rol que depende de si un contexto concurrente está activo.

En app.js: `sustituirGroup.style.display = AppState.sesionActiva ? 'block' : 'none'`.
Un condicional más que helix debería eliminar.

**Propuesta de sintaxis**:
```
-- En SesionActiva.helix, declarar un rol que se inyecta en ModalComentario:
context SesionActiva:
    role checkbox_sustituir: Checkbox en ModalComentario
        on cambio -> marcarSustituir(self.marcado)
```

O alternativamente, en ModalComentario:
```
context ModalComentario:
    role checkbox_sustituir: Checkbox si SesionActiva
        on cambio -> marcarSustituir(self.marcado)
```

**Observación**: esto es exactamente lo que hace DCI con los roles opcionales.
Un objeto (el modal) puede tener un rol adicional asignado por un contexto
concurrente. La sintaxis helix debería poder expresar esto.

---

## GAP-5: Transiciones condicionales por validación (MODERADO)

**Dónde aparece**: ModalCrearTarea (nombre no vacío + al menos una actividad),
ModalEditarTarea (nombre no vacío), ModalReset Fase3 (campo == "BORRAR").

**El problema**: el botón "Guardar" solo dispara la transición si los datos
del formulario son válidos. Si no, debería mostrar un error o deshabilitar el botón.
El DSL actual no expresa esto — la validación queda oculta en el external.

**Propuesta de sintaxis**:
```
role boton_guardar: Boton
    on tap si campo_nombre.valido -> guardarEdicion
    on tap si not campo_nombre.valido -> mostrarErrorNombre
```

O usando guards en las transiciones:
```
transitions:
    on guardarEdicion si campo_nombre.valido -> [cerrar_overlay]
    on guardarEdicion si not campo_nombre.valido -> mostrarErrorValidacion
```

**Nota**: esto introduce lógica booleana en el DSL. Hay que diseñar el
vocabulario de condiciones cuidadosamente para que siga siendo verificable
formalmente. Las condiciones deben referirse solo a propiedades de los datos
del contexto, nunca a estado externo.

---

## GAP-6: Roles dinámicos (multiplicidad) (MODERADO)

**Dónde aparece**: botones de actividad en ModalSeleccionActividad,
checkboxes en ModalCrearTarea y ModalReset.

**El problema**: el número de elementos no es fijo. Hay tantos botones de
actividad como actividades existan en el sistema. El DSL actual declara roles
como entidades únicas, no como colecciones.

**Propuesta de sintaxis**:
```
role boton_actividad[]: Actividad     -- [] indica multiplicidad
    on tap -> elegirActividad(self.id)
```

La verificación de completitud sigue funcionando: el compilador verifica que
el rol `boton_actividad[]` (como clase) maneje todos los eventos en todos los
contextos donde aparece, independientemente de cuántas instancias haya.

---

## GAP-7: Efectos de ciclo de vida (MODERADO)

**Dónde aparece**: ModalAcercaDe (carga datos al abrir), ModalHistorial
(carga datos al cambiar de sub-contexto).

**El problema**: algunos contextos necesitan ejecutar efectos al entrar
(`al_entrar`) o al salir (`al_salir`), no en respuesta a un evento de usuario.
Son disparados por el cambio de contexto en sí.

En app.js: `mostrarModalAcercaDe()` llama a `api.healthCheck()` y
`api.getAcumulado()` — no hay evento de usuario que los dispare; es el
hecho de abrir el modal lo que los lanza.

**Propuesta de sintaxis**:
```
effects:
    [al_entrar] -> external verificar_conexion()
    [al_salir]  -> limpiarEstadoModal()
```

Los corchetes distinguen los hooks de ciclo de vida de los efectos de acción.

---

## GAP-8: Flujo de error en externals (MENOR)

**Dónde aparece**: todas las llamadas a la API (cualquier modal que guarda datos).

**El problema**: las funciones `external` pueden fallar (servidor caído, error
de red, validación backend). El DSL actual no modela el flujo de error. En
app.js hay `catch` en cada función async, pero los errores son todos
`mostrarError(mensaje)` — el mismo manejador para todo.

Un sistema bien especificado debería poder decir:
```
external module cronometro_api:
    crear_tipo_tarea(...) -> TipoTarea | Error<ApiError>
```

Y los contextos deberían poder manejar el error:
```
transitions:
    on guardarEdicion      -> [cerrar_overlay]
    on error_guardar_edicion -> mostrarErrorAPI
```

**Por qué es menor**: la lógica de manejo de errores es en su mayoría
uniforme (`mostrarError`). Modelarla completamente en helix sería verboso
sin añadir mucha verificabilidad. Lo más pragmático puede ser que los
externals declaren que pueden fallar, y helix genere el código de
manejo de error estándar.

---

## Resumen de propuestas

| GAP | Propuesta | Impacto en verificabilidad |
|-----|-----------|---------------------------|
| 1 | `input:` en contextos overlay | ALTO: elimina estado implícito |
| 2 | `overlays:` en system + `[cerrar_overlay]` | ALTO: modela correctamente los modales |
| 3 | `concurrent:` en system + `[desactivar]` | ALTO: modela SesionActiva correctamente |
| 4 | `role X: T si <Contexto>` | MEDIO: elimina condicionales de visibilidad |
| 5 | Guards en transiciones (`si <condición>`) | MEDIO: valida flujos de formulario |
| 6 | `role X[]: T` (multiplicidad) | MEDIO: modela listas dinámicas |
| 7 | `[al_entrar]` / `[al_salir]` en effects | MEDIO: modela side effects de inicialización |
| 8 | Tipos de error en externals | BAJO: mejora completitud pero es verboso |

---

## Lo que funciona bien

Para ser honestos, estos gaps no invalidan el diseño central. El núcleo del
DSL — contextos, roles, eventos, transiciones — funciona muy bien para:

- La lógica ModoNormal / ModoEdicion (el problema original): **perfecto**.
- Los sub-contextos de ModalReset (3 fases): **funciona sin cambios**.
- Los sub-contextos de ModalHistorial (7d / 30d): **funciona con GAP-7**.
- La separación data / context: **sólida, sin fricciones**.
- Los módulos external: **funcionan bien como contratos**.

Los gaps más críticos (1, 2, 3) son todos variaciones del mismo tema:
el DSL necesita más tipos de relación entre contextos que solo "exclusivos
con prioridad". Los tres tipos identificados (base exclusivo, overlay apilable,
concurrente) son una extensión natural del modelo DCI, no una ruptura de él.

---

## Próximo paso sugerido

Antes de añadir sintaxis nueva, comprobar si los tres tipos de contexto
(base, overlay, concurrent) pueden expresarse con el modelo actual de
composición, usando convenciones de nomenclatura en lugar de palabras clave
nuevas. Si el verificador puede inferir el tipo desde las transiciones
(`[cerrar_overlay]` implica overlay; `[desactivar]` implica concurrent),
quizás no necesitamos keywords adicionales en la declaración del system.

La navaja de Occam antes de añadir palabras reservadas.
