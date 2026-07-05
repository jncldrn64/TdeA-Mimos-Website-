# ROADMAP.md: Sistema Helados Mimo's

> Cómo se usa: una fase por sesión, en orden. Cada decisión que se tome durante una fase
> se anota en `DECISIONS.md` con fecha; no se pierde en un chat. Una fase no se cierra
> sin correr `test-requisitos-funcionales.sh` (o `.ps1`) contra un SQL Server real y
> decir el resultado con números.

Este roadmap se depuró contra el código el 2026-07-05. Las prioridades del documento de
contexto anterior ya estaban cerradas en su mayoría: los templates existen (12 en uso
tras borrar un duplicado), RF-02 tiene su suite (`test_rf02_pagos`,
`test-requisitos-funcionales.sh:931`), el catálogo es público y la tabla de pedidos
guarda dirección y teléfono. Lo que sigue es lo que de verdad falta.

---

## FASE 1: Mover la reducción de stock del checkout al pago

El TODO más viejo del código (`ServicioCarritoCompras.java:214-216`). Hoy el stock baja
al hacer checkout, con el pedido todavía en `PAGO_PENDIENTE`. Un pedido que nunca se paga
deja stock descontado sin venta.

1. Cablear `ItemPedido` (la entidad existe y ningún archivo la usa): puerto, adaptador y
   persistencia de los items del carrito dentro del pedido durante el checkout.
2. Implementar de verdad `reducirStockDesdePedido()` (`ServicioPagos.java:224-231`,
   hoy un stub) y llamarla solo tras confirmar el pago.
3. Sacar la reducción de `procesarCheckout()` y arreglar el log que dice "Stock
   reducido" sin haber reducido nada.

**Criterio de éxito:** las suites de carrito y pagos pasan, y un pedido creado y nunca
pagado deja el stock intacto (caso nuevo en el script de tests).

---

## FASE 2: Costo de envío y descuentos reales

`costoEnvio` está hardcodeado en 5000.0 en dos controladores (`ControladorCarrito.java:53`
y `ControladorCarritoREST.java:123`), y el cálculo vive en la capa equivocada. La entidad
`Pedido` ya tiene los campos `costoEnvio` y `descuento`.

1. Mover el cálculo a `ServicioCarritoCompras` (o un servicio propio), una sola fuente.
2. Definir la regla de negocio del envío (fija, por monto mínimo, por zona) y anotarla
   en `DECISIONS.md` antes de programarla.
3. Descuentos: hoy no hay servicio de cupones. Decidir si entra o se descarta, y dejarlo
   escrito.

---

## FASE 3: PDF de la factura

`ServicioFacturacion` genera el registro con IVA 19% y número `FACT-YYYYMMDD-XXXXX`, y
`facturacion/detalle-factura.html` lo muestra. Falta el PDF descargable, que era parte
del flujo original. Requiere una dependencia nueva (consultar antes de agregarla, regla
de `CLAUDE.md`).

---

## FASE 4: Limpieza de huérfanos

**Estado: cerrada el 2026-07-05**, a pedido del dueño del repo en la misma sesión. Se
borraron 14 archivos: `keys.rs`, `nbactions.xml`, los 10 `.lock` vacíos de
`src/main/java` y el duplicado `templates/formulario-factura.html`.
`fix-version-null.sql` no se borró: se movió a `scripts/` para que no viaje adentro del
JAR, porque sigue siendo el parche para bases creadas antes del `@Version` de
`Producto`. Detalle en CHANGELOG v1.0 y en la entrada del 2026-07-05 de `DECISIONS.md`
sobre qué se versiona y qué no.

---

## BACKLOG (sin fecha)

- Búsqueda y filtros en el catálogo (`GET /api/productos/buscar?q=...`).
- Sesiones distribuidas si algún día hay más de una instancia: `@SessionScope` no escala
  horizontal sin sticky sessions o Redis.
- Sincronizar `<version>` de `pom.xml` con el CHANGELOG en el próximo PR de código que
  toque el artefacto (hoy: `0.0.1-SNAPSHOT` contra v1.0).
