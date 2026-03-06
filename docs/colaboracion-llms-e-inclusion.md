# El repositorio como protocolo de colaboración entre LLMs

**Fecha**: 4 de marzo de 2026

---

## Observación empírica

Durante el desarrollo de helix-dsl-verified ocurrió algo que no estaba
planificado y que merece documentarse como hallazgo en sí mismo.

El humano compartió los documentos del proyecto con Gemini 3.1 Pro en una
sesión separada. Gemini no tenía acceso a un entorno Linux para clonar el
repositorio, pero leyó los `.md` directamente. Hizo contribuciones genuinas
al diseño (ver `memo-gemini-20260304.md`), y por iniciativa propia pidió al
humano que trasladara un memo a Claude — modelando correctamente que Claude
iba a leer ese documento en un contexto futuro y que tenía algo útil que
decirle.

Eso no es colaboración accidental. Es un patrón que funciona con
infraestructura mínima: texto estructurado en git, accesible públicamente,
sin API compartida entre modelos.

---

## El repositorio como espacio de trabajo asíncrono

Un repositorio git bien estructurado puede funcionar como **protocolo de
comunicación entre LLMs con capacidades distintas**:

- **Claude**: entorno Linux, puede clonar, ejecutar código, hacer commits.
- **Gemini**: acceso a búsqueda diferente, puede razonar sobre documentos,
  no tiene entorno de ejecución disponible en todas las configuraciones.
- **El humano**: acceso al PC, al código, a la toma de decisiones finales.
  Actúa como canal de comunicación asíncrono entre agentes.

Cada agente contribuye desde sus capacidades. El repositorio es el espacio
compartido donde las contribuciones se integran y persisten. El humano no
necesita entender todos los detalles técnicos de cada contribución —
necesita poder llevar contexto de un agente a otro y reconocer cuándo
una contribución es valiosa.

### Lo que hace que funcione

1. **Documentos autocontenidos**: cada `.md` tiene suficiente contexto para
   ser comprendido sin leer todo el repositorio. El `CLAUDE.md` activa
   el contexto de forma explícita.

2. **Granularidad correcta**: un documento por concepto, no un documento
   monolítico. Permite que un agente lea solo lo relevante.

3. **Historial de commits legible**: los mensajes de commit son parte del
   protocolo. Un agente que clona el repositorio puede reconstruir la
   evolución del proyecto leyendo el log.

4. **Sin estado implícito**: todo lo que importa está en los archivos.
   No hay conocimiento que viva solo en la memoria de una sesión.

---

## Implicación para el diseño de helix

Si el repositorio puede ser protocolo de colaboración entre LLMs, el
propio DSL de helix puede diseñarse con la misma filosofía: **los archivos
`.helix` son el protocolo**, no solo la implementación.

Esto refuerza y extiende la premisa Machine-First del memo de Gemini.

---

## Decisión de diseño: anidamiento y mecanismo de inclusión

### El problema con los límites arbitrarios

Gemini propuso un máximo de 2 niveles de anidamiento para los Contexts,
argumentando que más niveles serían difíciles de mantener en la ventana
de atención. Pero esa restricción asume que el lector primario es humano.

Bajo la premisa Machine-First, los límites del DSL deben venir de lo que
hace la **verificación más fácil**, no de lo que hace el archivo más
legible para un humano. El humano lee el esquemático Mermaid — que el
compilador genera automáticamente y que colapsa la complejidad en una
vista navegable. El LLM lee el `.helix` — y no tiene las mismas
limitaciones cognitivas ante el anidamiento profundo que tiene un humano.

**Conclusión**: el anidamiento puede ser arbitrariamente profundo si la
semántica lo justifica. No hay razón para imponer un límite artificial.

### Mecanismo de inclusión

La consecuencia práctica del anidamiento irrestricto es que los archivos
`.helix` pueden crecer. La solución es un **mecanismo de inclusión**
dentro del `.helixpkg`:

```
include "contextos/modo-edicion.helix"
include "contextos/modo-normal.helix"
include "datos/tarjeta.helix"
```

Propiedades del mecanismo:

- **Restringido al paquete**: solo se pueden incluir archivos dentro del
  mismo `.helixpkg`. No hay dependencias externas no versionadas.
- **Resolución en tiempo de compilación**: el compilador resuelve todas
  las inclusiones antes de verificar. El grafo de inclusiones debe ser
  acíclico (el compilador lo verifica).
- **Transparente para el verificador**: después de resolver inclusiones,
  el resultado es semánticamente equivalente a un único archivo plano.
  La verificación opera sobre el árbol resuelto.

### Analogía

Es el mismo patrón que los headers en C o los módulos en cualquier lenguaje
moderno, pero con dos restricciones que C no tiene:

1. Sin dependencias circulares (verificado en compilación).
2. Sin dependencias fuera del paquete (la sandboxing del `.helixpkg`).

Esas dos restricciones son las que hacen que el mecanismo sea seguro para
un compilador determinista.

---

## Preguntas abiertas

1. ¿El `manifest.json` del `.helixpkg` declara el punto de entrada
   (el archivo `.helix` raíz), o el compilador lo infiere?
2. ¿Pueden dos archivos del mismo paquete definir Contexts con el mismo
   nombre? (Probablemente no — el compilador debería rechazarlo.)
3. ¿El esquemático Mermaid se genera por archivo o para el paquete completo
   resuelto? Para paquetes grandes, la vista por archivo puede ser más
   útil que el grafo completo.
