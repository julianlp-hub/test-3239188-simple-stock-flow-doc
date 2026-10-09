# Producto — Simple Stock Flow

> **Fuente única:** `spec/data-model.md`. Las citas §x.y remiten a ese documento.
> Lo que no sale del modelo está marcado como **Supuesto**.

## 1. Problema que resuelve

Un negocio que vende productos necesita saber **cuántas unidades tiene**, **qué vendió, quién lo vendió y a qué precio**, y poder **sumar ventas por producto en un período**. Sin un sistema, el stock y las ventas se desfasan: se vende lo que ya no hay, se pierde el precio al que se vendió y los reportes cambian cuando cambia el catálogo.

El modelo de datos muestra qué problemas están resueltos por diseño:

| Problema del negocio | Cómo lo resuelve el modelo | Fuente |
|---|---|---|
| Vender más de lo que hay | `stock >= 0` garantizado por el motor y retiro de stock atómico con la línea de venta | §2.2, §2.3 |
| Perder el precio de venta si el producto cambia | Nombre, precio y categoría quedan congelados en la línea de venta | §1, §2.4 |
| Reportes que cambian con el tiempo | El reporte agrupa por el valor congelado | §11.1 |
| Perder historial al «borrar» un producto | Baja lógica, nunca borrado físico | §2.2, §7.1 |
| No saber quién vendió | Cada venta registra al operador que la hizo | §2.3 |

**Supuesto:** el negocio es un comercio pequeño de venta presencial de artículos de ferretería y suministros. El modelo no lo dice de forma explícita, pero lo sugieren las cinco categorías sembradas (General, Herramientas, Electricidad, Fontanería, Pinturas, §9.1) y la ausencia de clientes y pagos (§1, §7).

## 2. Visión del producto

> *Para* pequeños negocios que venden artículos de un catálogo fijo, *Simple Stock Flow* es un sistema de inventario y ventas que mantiene el stock correcto y conserva un historial de ventas que no cambia, *a diferencia de* llevar el control a mano o en hojas sueltas.

Deja la frase de visión tal como está.

## 3. Usuarios

| Usuario | Qué necesita | Fuente |
|---|---|---|
| Vendedor (`seller`) | Registrar ventas y consultar productos | §2.5 |
| Administrador (`admin`) | Dar de alta vendedores, mantener el catálogo y ver reportes | §2.5, §11 (H-3) |

No hay compradores como usuarios del sistema (§1).

## 4. Propuesta de valor

1. **Stock confiable:** no baja de cero y se descuenta junto con la venta (§2.2, §2.3).
2. **Historial estable:** las ventas son inmutables y guardan copia de lo que se vendió (§2.3, §2.4).
3. **Reporte por producto en un rango de fechas,** calculado en el motor (§1, §6.1 Q9).
4. **Privacidad mínima:** solo se guarda al operador, no al cliente final (§7).

## 5. Qué es y qué no es el producto

**Es:** catálogo de productos con cinco categorías fijas, registro de ventas, usuarios internos con dos roles y reporte por producto.

**No es** (decisiones cerradas en el modelo):
- Un sistema de clientes o de pagos (§1, §7).
- Un sistema multimoneda (§3).
- Un sistema con auditoría de cambios del catálogo (§8).
- Un gestor de categorías (§2.1, §4.1).
- Un sistema con atributos extra de producto, como descripción o SKU (§1, DP-03).
- Un reporte por vendedor (§7.1, DP-02).

## 6. Criterios de éxito

**Supuesto:** el modelo no define métricas de éxito. Estas se derivan de sus reglas:

- Ninguna operación deja un producto con stock negativo.
- Un reporte de un rango cerrado devuelve siempre el mismo resultado.
- Ninguna venta queda sin al menos una línea.

- Un vendedor puede registrar una venta de varias líneas en una sola operación, y si alguna línea supera el stock disponible, la operación falla sin dejar el stock en negativo (§2.2, §2.3). **Supuesto:** el modelo no define este criterio como métrica; se deriva de sus reglas.

## 7. Riesgos y deudas conocidas del modelo

- Varias reglas solo viven en el dominio y no en el motor (`price > 0`, `quantity > 0`, rol válido); un `INSERT` manual las salta (§4, T-20).
- La autoría de la venta es texto, sin clave foránea a `user`, hasta T-12 (§5, FK-4).
- `spec.md` CA-06.1 («una fila por producto») choca con la decisión de §11.1 (puede haber más de una fila por producto si hubo recategorización).
