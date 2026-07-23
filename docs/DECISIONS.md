# DECISIONS.md: Sistema Helados Mimo's

> Registro append-only, estilo ADR. No se borra una entrada vieja aunque quede obsoleta;
> se agrega una nueva que la reemplaza y la referencia. Cada entrada abre con
> `## YYYY-MM-DD — <título>`. Antes de este archivo el porqué vivía en chats y en un
> documento de contexto que quedó 8 meses desactualizado; acá no se reescribe la historia.

---

## 2026-07-23 — Autorización de escritura de doble repo, y política de referencias cruzadas

**Contexto:** los dos repos Mimos (este y TdeA-Mimos-API-REST) borraron los nombres de
otros repos de sus DECISIONS para que cada documento se sostenga solo ante un lector sin
contexto. TL-FCCU, con la misma regla en la mano, hizo lo contrario en su PR #18:
identificó a MIDI-Scale-Trainer en la primera mención de su `ROADMAP.md` y su `AGENTS.md`.
Dos decisiones opuestas justificadas con la misma regla, ninguna escrita. Un agente que
abre un repo nuevo no sabe cuál seguir.

**Decisión:** tres cosas. Primera, esta tarea autoriza escribir en los dos repos Mimos a
la vez, este y TdeA-Mimos-API-REST, como excepción explícita a "Scope de escritura", que
manda un solo destino de escritura; TL-FCCU y MIDI-Scale-Trainer quedan solo lectura. La
excepción vale solo para esta tarea: al cerrarla vuelve a regir un repo por sesión.
Segunda, se adopta la política de anclar y no borrar referencias cruzadas (ver `CLAUDE.md`,
"Repos hermanos" y "Referencias cruzadas"). Tercera, un PR doc-only abre su propia sección
fechada en el CHANGELOG y puede dejarlo por delante de la versión del artefacto (regla en
`CLAUDE.md`, "CHANGELOG").

**Razón:** los dos repos Mimos comparten el mismo defecto, una referencia cruzada sin
política declarada, y se arreglan con el mismo texto; hacerlos en una sola tarea con dos PR
sale más barato y garantiza que queden idénticos. Se ancla en vez de borrar porque borrar
no alcanza: la cabecera de la v1.0 del CHANGELOG nombra a TL-FCCU y a MIDI-Scale-Trainer, y
esa historia no se reescribe, así que la referencia sobrevive; mejor que resuelva contra un
anclaje. No se revierte el borrado previo de DECISIONS (PR #59): la política rige hacia
adelante y las menciones que quedan en el CHANGELOG ahora resuelven contra "Repos
hermanos".

**Estado:** vigente. La excepción de scope se cierra al terminar esta tarea; la política de
referencias cruzadas y la regla de CHANGELOG quedan.

---

## 2026-07-05 — Adopción del estándar de documentación

**Contexto:** los demás repos del autor ya comparten
un estándar: CLAUDE.md con las reglas de sesión, CHANGELOG único en formato Keep a
Changelog, registro de decisiones append-only con fecha, commits `{add, chg, fix, rmv,
doc}`, fechas ISO 8601, prosa con los skills `no-ai-slop` y `rossmann-voice`, honestidad
de estado y scope de escritura. Este repo tenía un solo documento
(`CONTEXTO_PROYECTO.md`, 1654 líneas) congelado en el 2025-11-11, que contradecía el
código en puntos comprobables: decía 0/10 templates cuando hay 13, `costoEnvio = 0.0`
cuando el código usa 5000.0, y login obligatorio en un catálogo que ya es público.

**Decisión:** se adopta el estándar completo. Documentación canónica en `docs/`
(`ARCHITECTURE.md`, `ROADMAP.md`, `DECISIONS.md`), `CHANGELOG.md` y `CLAUDE.md` en la
raíz, y `README-TESTS-WINDOWS.md` se conserva como guía de entorno. De las dos variantes
del estándar (carpeta `docs/` con DECISIONS, o un AGENTS.md con sección "Known gaps") se
elige la primera, la de los proyectos de aplicación en español, que es lo que este repo
es. `CONTEXTO_PROYECTO.md` se elimina; su contenido
vigente, corregido contra el código, vive en `docs/ARCHITECTURE.md`.

**Razón:** un documento único que mezcla arquitectura, historial, tutorial y checklist se
desactualiza entero cuando cambia cualquier parte, y eso ya pasó. Separar el mapa
(ARCHITECTURE), el pendiente (ROADMAP) y el porqué (DECISIONS) deja que cada uno
envejezca a su ritmo. La alternativa de actualizar el documento viejo en el mismo formato
se descartó: 8 meses de deriva demuestran que el formato no se sostiene.

**Estado:** vigente.

---

## 2026-07-05 — Ratificación: arquitectura hexagonal con doble controlador

**Contexto:** la decisión es anterior a este registro y no tiene fecha propia; se
ratifica leyendo el código el 2026-07-05. Cada RF tiene un `@Controller` (Thymeleaf) y un
`@RestController` (JSON) que comparten los mismos casos de uso: 7 y 6 respectivamente,
sobre 12 casos de uso y 6 servicios.

**Decisión:** se mantiene el flujo estricto Controlador → Caso de Uso → Servicio →
Adaptador → Puerto → Entidad, con excepciones propias (23) manejadas en
`ManejadorGlobalExcepciones` y las convenciones de `docs/ARCHITECTURE.md` §7.

**Razón:** el patrón ya pagó: cuando `ControladorProductoREST` accedía al repositorio
directo se corrigió hacia casos de uso, y el formulario tradicional y el AJAX comparten
una sola lógica de negocio. No hay motivo para tocarlo.

**Estado:** vigente.

---

## 2026-07-05 — Ratificación: optimistic locking en el checkout, sin reserva de stock

**Contexto:** también anterior al registro. `Producto` lleva `@Version` y
`procesarCheckout()` valida y reduce stock en una sola transacción
(`ServicioCarritoCompras.java:204-240`).

**Decisión:** se mantiene: nada de reservas con TTL ni bloqueos pesimistas. El carrito no
reserva; el checkout compite y el que pierde recibe un error de conflicto claro, con
advertencias preventivas al consultar el carrito.

**Razón:** las reservas exigen jobs de expiración y castigan al que abandona el carrito.
El conflicto real (dos compradores del mismo último stock) lo resuelve
`OptimisticLockException` con rollback automático; la alternativa compleja no compra nada
para el tamaño de esta tienda.

**Estado:** vigente.

---

## 2026-07-05 — Ratificación con reparo: el stock se reduce en el checkout (temporal)

**Contexto:** el TODO de `ServicioCarritoCompras.java:214-216` lo dice desde que se
escribió: el stock debería bajar al confirmar el pago, no al hacer checkout. La pieza que
falta (persistir los items del carrito en el pedido vía `ItemPedido`) está creada como
entidad y sin usar. Peor: `ServicioPagos.reducirStockDesdePedido()` es un stub que loguea
"Stock reducido" sin reducir nada (`ServicioPagos.java:224-231`).

**Decisión:** se documenta como deuda, no como diseño. Es la Fase 1 del roadmap y bloquea
declarar correcto el flujo de pago. El log mentiroso del stub se corrige en esa misma
fase.

**Razón:** un pedido que nunca se paga hoy deja stock descontado sin venta. Y un log que
afirma una operación que no ocurrió viola la regla de honestidad de estado que este
estándar acaba de adoptar; se deja escrito para que nadie lo cite como implementado.

**Estado:** vigente hasta cerrar la Fase 1.

---

## 2026-07-05 — Qué se versiona además de los .java: el wrapper sí, la metadata de IDE no

**Contexto:** al crear el proyecto le dijeron al autor "pusheá solo los `.java` para
evitar basura". El consejo era mitad cierto, y el repo terminó con las dos mitades mal:
le faltó el filtro (entraron `keys.rs` con 2 líneas de chat, `nbactions.xml`, 10 archivos
`.lock` vacíos dentro de `src/main/java` y un `formulario-factura.html` huérfano) y el
consejo, tomado literal, habría dejado afuera archivos que la build necesita. Los `.lock`
además ya estaban cubiertos por la regla `*.lock` del `.gitignore`, pero git no destrackea
lo que se commiteó antes de la regla.

**Decisión:** se borran los 14 archivos y `fix-version-null.sql` se mueve de
`src/main/resources/` a `scripts/`. Se quedan versionados, y con motivo: `pom.xml`, el
Maven Wrapper completo (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/maven-wrapper.properties`),
`.gitattributes`, `application.properties`, los scripts de test, los templates y
`static/`. `nbactions.xml` y `nb-configuration.xml` entran al `.gitignore` para que
NetBeans no los vuelva a colar.

**Razón:** la documentación del Maven Wrapper y las guías estándar (Baeldung, "A Quick
Guide to Maven Wrapper") dicen commitear `mvnw`, `mvnw.cmd` y `.mvn/`: son lo que permite
que cualquiera compile con Maven 3.9.11 exacto sin tenerlo instalado, y con
`distributionType=only-script` no hay ni un binario adentro. `nbactions.xml` es lo
contrario: configuración de acciones que NetBeans regenera al configurar el run, y las
plantillas de `.gitignore` para NetBeans lo excluyen. `fix-version-null.sql` no era
basura (es el parche para bases creadas antes del `@Version` de `Producto`, sin él el
checkout tira NullPointerException) pero vivía en `src/main/resources`, que se empaqueta
adentro del JAR; un parche manual de SSMS no tiene por qué viajar en el artefacto. Al
borrar los `.lock` desaparecen las carpetas vacías `web/seguridad/` y
`web/servicios/requisitos/no funcionales/`: git no trackea carpetas, y la segunda, con
espacio en el nombre, nunca fue un paquete Java válido.

**Estado:** vigente.

---

### Plantilla para nuevas entradas

```
## YYYY-MM-DD — Título corto de la decisión

**Contexto:** qué problema o pregunta motivó esto.

**Decisión:** qué se decidió, en una o dos frases.

**Razón:** por qué esta opción y no otra (mencionar alternativas descartadas si aplica).

**Estado:** vigente / reemplazada por [fecha] / obsoleta
```
