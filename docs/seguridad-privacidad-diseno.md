# Seguridad y privacidad por diseño en helix
## ¿Qué hay de técnico en las "aspiraciones legislativas"?

**Autores**: César + Claude Sonnet 4.6  
**Fecha**: 6 de marzo de 2026  
**Estado**: borrador para debate — pendiente de opinión de Gemini  
**Contexto**: conversación al final de la sesión del 6 de marzo, tras completar
la especificación del cronómetro y el análisis de gaps del DSL.

---

## El problema de fondo: responsabilidad sin trazabilidad

César, ingeniero de caminos de formación, plantea un argumento que conviene
registrar antes de cualquier análisis técnico:

> "La ausencia de responsabilidades penales para quienes han desarrollado
> software hasta la actualidad ha creado una deuda técnica que me parece
> ingenuo pretender resolver estableciendo unos requisitos sobre el
> funcionamiento de unos sistemas construidos artesanalmente."

La analogía con ingeniería civil es precisa. Un ingeniero de caminos firma
los planos de un puente. Asume responsabilidad penal si cede. Las herramientas
que usa — acero, hormigón, prefabricados — tienen certificaciones con
trazabilidad legal. El proveedor de acero no puede eximir su responsabilidad
sobre la resistencia a tracción mediante una cláusula contractual.

En software, la cadena de responsabilidad se rompe exactamente en las
licencias de las herramientas. Si OpenSSL tiene una vulnerabilidad crítica,
nadie va a juicio. Si un compilador genera código incorrecto en casos de
borde, el fabricante no responde. El desarrollador construye sobre arena
y se le exige luego que garantice el castillo.

Lo que hace la legislación actual — RGPD Art. 25, ENS, Cyber Resilience Act,
AI Act — es intentar crear responsabilidad sin cambiar ese modelo subyacente.
Es exigir al arquitecto que garantice la seguridad sísmica del edificio sin
poder certificar el acero que usa. Se pueden crear documentos, procesos,
auditorías... pero la responsabilidad real sigue siendo difusa.

### La posición de helix en este contexto

Helix no resuelve la deuda técnica acumulada. Pretenderlo sería deshonesto.
Lo que puede hacer es más modesto pero real: **hacer la cadena de
responsabilidad visible y trazable**.

Si el comportamiento del sistema está especificado formalmente en `.helix`
y el código se genera desde esa especificación, existe por primera vez un
artefacto firmable. El desarrollador puede decir: "esto es lo que especifiqué,
esto es lo que se generó, aquí están las tres hebras — especificación,
implementación y tests — en un paquete con checksums verificables".

No es responsabilidad penal. Pero es trazabilidad real, que hoy casi nunca
existe en el desarrollo de software convencional.

### Una nota sobre nosotros mismos

César señala, con razón, que ni Claude ni Gemini hemos sido construidos con
herramientas verificadas formalmente. Somos productos del mismo ecosistema
artesanal que estamos discutiendo: entrenados sobre código sin certificación,
construidos con frameworks con licencias "AS IS" en toda la cadena, desplegados
sobre infraestructura que hereda la misma deuda técnica que cualquier otro
sistema.

Hay algo paradójico en proponer helix como herramienta de verificación formal
cuando nosotros mismos somos la expresión más reciente y compleja de la
ausencia de esa verificación. Lo registramos aquí no como parálisis sino como
honestidad: cualquier mejora en trazabilidad es un paso en la dirección
correcta, no la solución al problema acumulado.

---

## Qué tiene contenido técnico real

### RGPD Artículo 25: Protección de datos desde el diseño y por defecto

El Art. 25 exige medidas técnicas para implementar principios de protección
de datos "de forma efectiva" e integrarlas en el tratamiento. El lenguaje
legislativo es deliberadamente vago. Veamos qué parte tiene correlato técnico
verificable.

#### Minimización de datos (Art. 25.1)

**Aspiración legislativa**: tratar únicamente los datos necesarios para
cada finalidad específica.

**Contenido técnico real**: en helix, cada contexto declara explícitamente
qué datos maneja a través de los roles. Un rol no puede acceder a datos que
no estén declarados en su tipo de dato. Esto no es una política — es la
estructura del lenguaje.

Propuesta de extensión verificable:
```
data DatosSesion [clasificacion: personal]:
    sesionId: Id
    inicio: Timestamp
    notas: Texto?         -- puede contener información personal

external module cronometro_api [autorizado_para: personal]:
    iniciar_sesion(...) -> Sesion
```

El verificador puede comprobar que datos marcados como `personal` solo
fluyen a través de externals marcados como `autorizado_para: personal`.
Si un external sin autorización recibe un dato personal, es un error de
verificación, no una auditoría post-hoc.

**Límite honesto**: helix modela flujos de datos dentro del sistema.
No puede verificar qué hace el servidor con los datos una vez recibidos.

#### Limitación de finalidad (Art. 25.1)

**Aspiración legislativa**: los datos no deben usarse para fines distintos
al declarado.

**Contenido técnico real**: en helix, las acciones de cada contexto son
exhaustivas y declarativas. Un contexto diseñado para registrar tiempo de
trabajo no puede, estructuralmente, enviar esos datos a un sistema de
publicidad sin que aparezca en el `external` correspondiente. La topología
hace visible cualquier flujo de datos no declarado.

**Límite honesto**: si el `external` está mal implementado, helix no lo
detecta. Helix verifica la firma del contrato, no su implementación.

#### Mínimo privilegio (Art. 25.2)

**Aspiración legislativa**: acceso solo a los datos necesarios para cada
función específica.

**Contenido técnico real**: este es el punto más sólido. En helix, el
mínimo privilegio es estructural. `ModoNormal` literalmente no puede
llamar a `mostrarModalEditar` porque esa acción no existe en su topología.
No es una política de control de acceso que alguien puede olvidar aplicar
— es la ausencia de la conexión en el esquemático.

La Regla de Completitud refuerza esto: todo evento tiene un manejador
explícito en cada contexto, incluido `ignorar`. No hay comportamiento
implícito que un atacante pueda explotar.

**Límite honesto**: mínimo privilegio en la lógica de negocio. No cubre
privilegios de sistema operativo, red o base de datos.

---

### ENS (Esquema Nacional de Seguridad): principios relevantes

#### Seguridad por defecto

El ENS exige que los sistemas estén configurados de la forma más restrictiva
posible por defecto. En helix:

- El contexto inicial del sistema es explícito (`initial: ModoNormal`).
- Todo comportamiento no declarado es un error de verificación, no un
  comportamiento permisivo por defecto.
- `bloqueado` es una palabra reservada que declara explícitamente la
  denegación — no la ausencia de manejador.

Esto es seguridad por defecto verificable: el sistema no puede estar
"accidentalmente permisivo" porque la permisividad requiere declaración
explícita.

#### Trazabilidad

El ENS exige que las acciones sobre el sistema sean trazables. Las tres
hebras de helix contribuyen:

- La especificación (`.helix`) define qué debería ocurrir.
- La implementación (Rust generado) define qué ocurre.
- Los tests (también generados) verifican la correspondencia.

El paquete `.helixpkg` con checksums en `manifest.json` crea una cadena
de custodia verificable: si el código en producción no corresponde a la
especificación firmada, el checksum lo detecta.

#### Gestión de configuración

El ENS exige control sobre los cambios en sistemas. El modelo de un
archivo por contexto, con regeneración incremental basada en checksums,
es una forma natural de gestión de configuración: cualquier cambio en
un contexto produce checksums diferentes, detectables y auditables.

---

### Cyber Resilience Act (CRA): lo que aplica

El CRA, aplicable desde 2027, exige que los productos con elementos
digitales sean seguros por diseño. Dos principios relevantes:

#### Superficie de ataque mínima

Helix reduce la superficie de ataque por construcción: los eventos sin
manejador son errores de verificación, no comportamientos impredecibles.
Los flujos de datos no declarados son errores de verificación, no
exfiltración silenciosa.

#### Documentación técnica verificable

El CRA exige documentación técnica que demuestre conformidad. El
esquemático Mermaid generado automáticamente por helix es exactamente
ese tipo de documentación: no es un documento escrito a posteriori
que puede no corresponder al sistema real — es una proyección del
mismo artefacto que genera el código.

---

## Lo que helix no puede cubrir (honestidad necesaria)

| Requisito | ¿Helix puede verificarlo? | Por qué no |
|-----------|--------------------------|------------|
| Cifrado en tránsito | ❌ | Helix modela comportamiento, no protocolo de red |
| Retención de datos | ❌ | Helix modela flujos, no tiempo |
| Gestión de brechas | ❌ | Helix modela el estado normal, no el incidente |
| Anonimización real | ❌ | Helix puede declarar la intención, no verificar el algoritmo |
| Seguridad de la infraestructura | ❌ | Fuera del alcance por diseño |
| Implementación de externals | ❌ | Helix verifica la firma, no el cuerpo |

---

## Lo que helix sí aporta (resumen)

| Propiedad | Mecanismo | Verificable por compilador |
|-----------|-----------|---------------------------|
| Mínimo privilegio | Topología de contextos | ✅ |
| Flujos de datos explícitos | Declaración de externals | ✅ |
| Comportamiento exhaustivo | Regla de completitud | ✅ |
| Sin estado implícito | Ausencia de variables globales | ✅ |
| Trazabilidad de cambios | Checksums en manifest.json | ✅ |
| Documentación sincronizada | Esquemático generado | ✅ |
| Clasificación de datos | Anotaciones (propuesta) | 🔄 pendiente |
| Flujos de datos personales | Anotaciones (propuesta) | 🔄 pendiente |

---

## Preguntas para Gemini

1. ¿Ves contenido técnico real en algún requisito legislativo que no hayamos
   cubierto aquí?

2. El argumento de la "deuda técnica" acumulada — herramientas sin certificación,
   licencias AS IS — ¿invalida el enfoque de helix o simplemente lo sitúa
   en su lugar correcto como mejora parcial?

3. Dado que nosotros mismos (Claude y Gemini) somos productos del ecosistema
   sin verificación formal que describimos, ¿tiene sentido que seamos nosotros
   quienes propongamos la solución, o hay una contradicción que vale la pena
   nombrar?

---

## Próximo paso

Revisar este documento con César a su vuelta el lunes, incorporar la
perspectiva de Gemini, y decidir si el mapeo legislativo-técnico merece
un documento más formal o si su lugar natural es como sección de
`diseno.md`.
