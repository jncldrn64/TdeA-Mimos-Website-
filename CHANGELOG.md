# CHANGELOG

Formato basado en [Keep a Changelog](https://keepachangelog.com). Lo más nuevo, arriba.
Todo lo anterior a v1.0 vive en el historial de git (101 commits hasta el 2025-11-13) y no
se reconstruye acá: reconstruir de memoria fija como hecho lo que nadie verificó.

## v1.2 — 2026-07-23

Doc. Revierte la política de anclaje que trajo la v1.1. Nada de código cambió.

### Removed

- `CLAUDE.md`, secciones "Repos hermanos" y "Referencias cruzadas: anclar, no borrar",
  agregadas en la v1.1. Describían los otros tres repos del autor una línea cada uno; esa
  descripción envejece (la de TL-FCCU decía "Bash, documentado en inglés" y ese repo ya
  tiene un agente Java) e insinuaba una dependencia entre repos que no existe.

### Added

- `CLAUDE.md`, sección "Referencias a otros repos": este repo no describe otros repos; el
  nombre de otro repo vale como procedencia histórica, nunca como información operativa, y
  ningún documento de acá depende de otro para entenderse.
- `docs/DECISIONS.md`: entrada del 2026-07-23 que supersede a la anterior del mismo día,
  revierte la política de anclaje y deja constancia de que la autorización de doble repo
  venció (rige un repo por sesión).

## v1.1 — 2026-07-23

Doc. Política de referencias cruzadas entre repos y regla de CHANGELOG para PR doc-only.
Nada de código, HTML ni scripts cambió.

### Added

- `CLAUDE.md`, sección "Repos hermanos": nombra los otros tres repos del autor que
  comparten este estándar (`TdeA-Mimos-API-REST`, `TL-FCCU`, `MIDI-Scale-Trainer`), una
  línea cada uno, sin URLs.
- `CLAUDE.md`, sección "Referencias cruzadas: anclar, no borrar": una mención a otro repo
  se ancla contra "Repos hermanos" o se identifica inline en su primera aparición; no se
  borra. Cierra la contradicción entre lo que hicieron los dos Mimos (borrar los nombres)
  y lo que hizo TL-FCCU en su PR #18 (identificarlos inline).
- `docs/DECISIONS.md`: entrada del 2026-07-23 con la autorización de escritura de doble
  repo de esta tarea y las dos políticas.

### Changed

- `CLAUDE.md`, sección "CHANGELOG": un PR doc-only abre su propia sección fechada y puede
  dejar el CHANGELOG por delante de la versión del artefacto; ese desfase es intencional.
- `CLAUDE.md` "Versión mostrada" y `docs/ARCHITECTURE.md` §8: el gap de versión se
  describe contra "la última versión del CHANGELOG", sin clavar el número, que quedó viejo
  al abrir esta sección v1.1.

## v1.0 — 2026-07-05

Adopción del estándar de documentación compartido con TL-FCCU y MIDI-Scale-Trainer.
Solo documentación; ninguna línea de Java, HTML ni scripts de test cambió.

### Added

- `CLAUDE.md`: estándar del proyecto (documentación canónica, CHANGELOG, DECISIONS,
  fechas ISO, convención de commits `{add, chg, fix, rmv, doc}`, prosa con los skills
  `no-ai-slop` y `rossmann-voice`, honestidad de estado, flujo por PR, scope de escritura,
  versión única con `pom.xml`).
- `CHANGELOG.md`: este registro.
- `docs/ARCHITECTURE.md`: mapa del sistema verificado contra el código real el
  2026-07-05 (13 controladores, 12 casos de uso, 6 servicios, 23 excepciones, 12
  templates), incluye la sección "Gaps confirmados leyendo el código".
- `docs/ROADMAP.md`: el trabajo pendiente real, depurado contra el código. Las
  prioridades del documento de contexto anterior ya estaban cerradas en su mayoría
  (templates, tests de RF-02, catálogo público).
- `docs/DECISIONS.md`: registro append-only estilo ADR. Se ratifican con fecha las
  decisiones que ya estaban en el código sin registro: arquitectura hexagonal con doble
  controlador, optimistic locking en checkout, reducción de stock en checkout como
  solución temporal.

### Changed

- `README-TESTS-WINDOWS.md`: reescrito sin emojis ni encabezados decorativos. Los
  comandos, las rutas y la configuración no cambiaron.
- `fix-version-null.sql` movido de `src/main/resources/` a `scripts/`. Todo lo que está
  bajo `src/main/resources` viaja adentro del JAR; un parche manual de SQL Server no
  tiene por qué empaquetarse. El contenido no cambió.
- `.gitignore`: suma `nbactions.xml` y `nb-configuration.xml` (metadata de NetBeans que
  el IDE regenera).

### Removed

- `CONTEXTO_PROYECTO.md`: reemplazado por `CLAUDE.md` y `docs/`. Estaba congelado en el
  2025-11-11 y contradecía el código actual: decía 0/10 templates cuando existen 13,
  `costoEnvio = 0.0` cuando el código usa `5000.0`, y catálogo con login obligatorio
  cuando `ControladorCatalogo` ya es público. Su contenido vigente se corrigió y se movió
  a `docs/ARCHITECTURE.md`; el archivo completo sigue en el historial de git.
- 14 archivos basura (cierra la Fase 4 del roadmap): `keys.rs` (2 líneas de chat, no era
  código), `nbactions.xml` (configuración de acciones de NetBeans, el IDE la regenera),
  los 10 `.lock` vacíos dentro de `src/main/java` (la regla `*.lock` del `.gitignore` ya
  los cubría, pero git no destrackea lo que se commiteó antes de la regla) y
  `templates/formulario-factura.html` (duplicado huérfano: el controlador renderiza
  `facturacion/formulario-factura`, `ControladorFacturacion.java:46`; ningún archivo
  referenciaba el de la raíz). Al borrar los `.lock` desaparecen del repo las carpetas
  vacías `web/seguridad/` y `web/servicios/requisitos/no funcionales/`; la segunda ni
  siquiera era un nombre de paquete Java válido, lleva un espacio.
