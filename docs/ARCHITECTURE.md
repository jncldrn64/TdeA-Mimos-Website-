# ARCHITECTURE.md: Sistema Helados Mimo's

> **Regla de este documento:** todo lo que está acá se verificó contra el código real el
> 2026-07-05, con archivo y línea. Lo que no se pudo verificar se marca como tal. El
> documento de contexto anterior quedó congelado en el 2025-11-11 y llegó a contradecir
> el código en tres puntos comprobables (ver §8); no se repite ese patrón.

## 0. Qué es y en qué está hecho

Tienda web para Helados Mimo's: catálogo, carrito, checkout, pasarela de pagos ficticia,
facturación con IVA colombiano y seguimiento de pedidos. Spring Boot 3.5.7 sobre Java 17
(`pom.xml:30`), persistencia JPA/Hibernate contra MS SQL Server, vistas Thymeleaf con
Bootstrap. Licencia GPL-3.0.

Los números del árbol, contados el 2026-07-05 tras la limpieza de la Fase 4: 80 archivos
Java bajo `src/main/java`, 12 templates HTML bajo `src/main/resources/templates`, 101
commits en `main` hasta el 2025-11-13. Todo el código está en español: clases, métodos,
variables.

Fuera de los `.java` se versiona lo que la build necesita para reproducirse: `pom.xml`,
el Maven Wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/maven-wrapper.properties`, que fija
Maven 3.9.11 sin binarios porque `distributionType=only-script`), `.gitattributes` (fin
de línea LF para `mvnw`, CRLF para `.cmd`), `application.properties`, los dos scripts de
test, `scripts/fix-version-null.sql` y la documentación. La metadata de IDE
(`nbactions.xml`, `.idea/`, `nbproject/`) y los `target/`/`build/` no: el editor y Maven
los regeneran, y el `.gitignore` los cubre.

## 1. Arquitectura hexagonal: la regla que ordena todo

Cada request atraviesa las capas en un solo sentido:

```
Controlador → Caso de Uso → Servicio → Adaptador → Puerto → Entidad
```

Un controlador nunca inyecta un repositorio ni llama a un servicio directo; pasa por su
caso de uso. Un caso de uso nunca toca un adaptador; pasa por el servicio. Romper ese
sentido es el anti-patrón que ya se corrigió una vez en `ControladorProductoREST` (usaba
el repositorio directo) y no vuelve.

Las piezas, contadas en el código:

| Capa | Ubicación | Cantidad |
|---|---|---|
| Entidades | `web/entidades/` | 9 clases + 3 enums |
| Puertos | `web/puertos/` | 6 |
| Adaptadores | `web/adaptadores/` | 6 |
| Servicios (uno por RF, más registro) | `web/servicios/requisitos/funcionales/` | 6 |
| Casos de uso | `web/casosdeuso/` | 12 |
| Controladores | `web/controladores/` | 13 (7 `@Controller`, 6 `@RestController`) |
| Excepciones propias | `web/excepciones/` | 23 |

El manejo de errores es centralizado: `ManejadorGlobalExcepciones` (`@ControllerAdvice`,
en `web/excepciones/manejadores/`) traduce cada excepción propia a su código HTTP. Si se
crea una excepción nueva, el handler se agrega ahí, no como try-catch en el controlador.

## 2. Doble controlador por funcionalidad

Cada RF expone dos entradas que comparten los mismos casos de uso: un `@Controller` que
renderiza Thymeleaf (server-side) y un `@RestController` que responde JSON para AJAX.
`ControladorCarrito` y `ControladorCarritoREST` llaman ambos a `CasoDeUsoAccesoCarrito`;
la lógica vive una sola vez.

El criterio de reparto: página completa (login, catálogo, carrito, pasarela) por
server-side; operación puntual sin recarga (modificar cantidad, checkout, procesar pago)
por REST. Los formularios tradicionales funcionan sin JavaScript; el JavaScript mejora la
experiencia pero no es requisito.

## 3. Los 5 requisitos funcionales y dónde viven

| RF | Qué hace | Servicio | Controladores |
|---|---|---|---|
| RF-01 Inventario | Alta, edición, stock y activación de productos | `ServicioInventario` | `ControladorProductoREST` |
| RF-02 Pagos | Pasarela ficticia: tarjetas de prueba y contra entrega | `ServicioPagos` | `ControladorPagos` + REST |
| RF-03 Login/Registro | Registro en 2 pasos, sesión HTTP | `ServicioAutenticacion`, `ServicioRegistro` | `ControladorAutenticacion` + REST |
| RF-04 Facturación | Factura con IVA 19%, número `FACT-YYYYMMDD-XXXXX` | `ServicioFacturacion` | `ControladorFacturacion` + REST |
| RF-05 Carrito | Carrito por sesión, checkout atómico | `ServicioCarritoCompras` | `ControladorCarrito` + REST |

Fuera de los 5 RF hay tres controladores más: `ControladorCatalogo` (catálogo público,
sin login: `ControladorCatalogo.java:26-40` solo agrega datos de usuario si hay sesión),
`ControladorDatosEnvio` (captura dirección y teléfono antes del pago) y
`ControladorPedidos` (`GET /pedidos`, seguimiento para usuarios autenticados, template
`seguimiento-pedidos.html`).

La pasarela valida contra dos tarjetas hardcodeadas para testing:
`4111111111111111` (Visa) y `5500000000000004` (Mastercard), constantes en
`ServicioPagos.java:43-44`. Efectivo y datáfono generan un código de 6 dígitos que sale
por el log de Spring Boot.

## 4. Flujo de compra completo (verificado en controladores y enums)

```
/catalogo (público)
  → /carrito (agrega, edita, elimina; requiere login)
  → /datos-envio (dirección y teléfono)
  → checkout → Pedido en estado PAGO_PENDIENTE
  → /pasarela/{idPedido} → pago válido → PAGO_CONFIRMADO
  → factura opcional (/factura/formulario/{idPedido})
  → /pedidos (seguimiento: EN_CAMINO, ENTREGADO, CANCELADO)
```

`EstadoPedido` tiene exactamente 5 valores: `PAGO_PENDIENTE`, `PAGO_CONFIRMADO`,
`EN_CAMINO`, `ENTREGADO`, `CANCELADO` (`web/entidades/enums/EstadoPedido.java`). El
documento viejo lo llamaba `PENDIENTE_PAGO`; ese nombre no existe en el código.

## 5. Concurrencia: optimistic locking en el checkout

El stock no se reserva al agregar al carrito. La validación y la reducción ocurren juntas
en `procesarCheckout()` (`ServicioCarritoCompras.java:204-240`), dentro de una
transacción, sobre una entidad `Producto` con `@Version`. Dos checkouts simultáneos sobre
el mismo stock: el primero commitea y sube la versión, el segundo recibe
`OptimisticLockException`, la transacción se reversa y el usuario ve un error claro de
conflicto. `GET /api/carrito` devuelve advertencias preventivas cuando el stock cambió
por debajo de lo que el carrito pide.

`ServicioCarritoCompras` es `@SessionScope`: una instancia por sesión HTTP. Cambiarlo a
`@Service` a secas mezcla los carritos de todos los usuarios; no se toca.

## 6. Testing

Dos scripts equivalentes en la raíz: `test-requisitos-funcionales.sh` (Linux, curl) y
`test-requisitos-funcionales.ps1` (Windows, `Invoke-RestMethod`; su guía de entorno es
`README-TESTS-WINDOWS.md`). El script bash define 9 suites, una función por bloque
(`test-requisitos-funcionales.sh:354-1337`): homepage, RF-03, RF-01, cuatro de RF-05
(carrito, warnings de stock, checkout, conflictos), RF-02 y RF-04.

Los tests pegan a los mismos endpoints REST que usa el frontend; no hay endpoints de
testing ni backdoors. Necesitan la aplicación corriendo contra un SQL Server real.
**En la sesión del 2026-07-05 no se corrieron** (no hay SQL Server en este entorno); los
porcentajes que circulan de corridas anteriores (94%, 95%) valen para la fecha en que se
corrieron, no para hoy.

## 7. Convenciones de código, con el porqué

1. **Español en todo.** `ServicioCarritoCompras`, no `ShoppingCartService`.
2. **Máximo 2 niveles de indentación.** Early return, métodos privados u `Optional`
   antes que anidar un tercer `if`.
3. **Máximo 5 líneas de comentario por clase.** Hubo clases con entre 20 y 44 líneas de
   comentarios que narraban lo que el nombre ya decía; se recortaron y la regla quedó.
4. **`throws` antes que try-catch.** Cada excepción es propia y específica
   (`StockInsuficienteException`, no `RuntimeException`), y la respuesta HTTP la arma
   `ManejadorGlobalExcepciones`. Un try-catch genérico en un controlador es un bug de
   estilo aunque compile.
5. **`@Transactional` en cada método que escribe.** El checkout depende de eso para el
   rollback del optimistic locking (§5).
6. **SLF4J para logs.** `logger.warn("Producto no encontrado: {}", ex.getMessage())`,
   nunca `System.out`.

## 8. Gaps confirmados leyendo el código (2026-07-05)

Honestidad de estado: esto es lo que está mal o a medias hoy, con archivo y línea. No se
reporta ninguno como resuelto sin una corrida real que lo pruebe.

- **El stock se reduce en el checkout, no al confirmar el pago.** El TODO está escrito en
  `ServicioCarritoCompras.java:214-216` y la reducción en las líneas 227-235.
  `ServicioPagos.reducirStockDesdePedido()` (`ServicioPagos.java:224-231`) existe pero es
  un stub: no reduce nada y aun así loguea "Stock reducido para pedido {}". Ese log
  afirma algo que no pasó; corregirlo va junto con la Fase 1 del roadmap.
- **`ItemPedido.java` es una entidad huérfana.** Ningún otro archivo la referencia. Es el
  prerequisito de mover el stock al pago (los items del carrito no se persisten en el
  pedido) y está creada pero sin cablear.
- **`costoEnvio` hardcodeado en 5000.0, dos veces.** `ControladorCarrito.java:53` y
  `ControladorCarritoREST.java:123`. Además vive en la capa de controlador, que según §1
  no debería calcular nada.
- **`pom.xml` dice `0.0.1-SNAPSHOT`** mientras el CHANGELOG asigna v1.0. El bump cambia
  el artefacto, así que queda fuera de un PR doc-only; se cierra en el próximo PR de
  código (regla "Versión mostrada" de `CLAUDE.md`).
- **Tests sin correr en este entorno.** Ver §6.
