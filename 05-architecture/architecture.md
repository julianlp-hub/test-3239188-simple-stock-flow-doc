# Arquitectura — Simple Stock Flow

> **Fuente única:** `spec/data-model.md`. Las citas §x.y remiten a ese documento.
> Lo que no sale del modelo está marcado como **Supuesto**.

## 1. Estilo arquitectónico

**Arquitectura hexagonal (puertos y adaptadores).** El dominio no conoce la base de datos ni los servicios externos; habla con ellos a través de puertos.

Evidencia en el modelo: el «adaptador de persistencia» traduce entre clases y tablas (§0), el «puerto de hash» produce el hash de contraseña (§2.5, §9.2) y un «puerto de lectura» calcula el reporte (§1, §6.1).

## 2. Piezas del sistema

| Pieza | Tipo | Responsabilidad | Fuente |
|---|---|---|---|
| `Category` | Entidad de referencia | Cinco categorías fijas, solo lectura, sin ciclo de vida | §2.1, §9.1 |
| `Product` | Raíz de agregado (catálogo) | Nombre, precio, stock, categoría, imagen; baja lógica | §2.2 |
| `Sale` | Raíz de agregado (ventas) | Venta inmutable: quién, cuándo y qué | §2.3 |
| `SaleItem` | Entidad interna de `Sale` | Línea con producto, cantidad y precio congelado | §2.4 |
| `User` | Raíz de agregado (identidad) | Usuario, rol (`admin` / `seller`) y hash de clave | §2.5 |
| `Money`, `Quantity` | Objetos de valor | Sin identidad ni tabla; viven en la fila de su dueño | §2 |
| Puerto de hash | Puerto de salida | El dominio nunca ve la clave en claro | §2.5, §9.2 |
| Puerto de lectura de reporte | Puerto de salida | Agregación por producto calculada en el motor | §1, §6.1 (Q9) |
| Adaptador de persistencia | Adaptador | Mapea clases a tablas; colecciones C# en plural, tablas en singular | §0 |
| Almacenamiento de imágenes | Sistema externo | Guarda el binario; el modelo solo guarda una clave opaca | §1, §7.1 |
| Arranque de la aplicación | Proceso | Crea el administrador inicial con credenciales de entorno | §9.2 |

## 3. Dónde vive cada regla

El modelo clasifica cada regla en tres marcas (§ «Cómo se lee este documento», §4):

- **motor** (la garantiza Postgres): `stock >= 0` (`ck_product_stock_non_negative`), unicidad de `category.name` y `user.username`, claves foráneas FK-1, FK-2 y FK-3, único `(sale_id, product_id)`.
- **solo dominio** (la garantiza C#): `price > 0`, `quantity > 0`, rol válido, nombre de usuario en minúsculas, venta con al menos una línea, retirar más stock del disponible.
- **pendiente**: `sale.sold_by_user_id` y FK-4 (T-12), índices de búsqueda (T-13), los `CHECK` que bajan al motor (T-20).

**Implicación:** lo que solo vive en el dominio protege a la aplicación pero no a los datos; un `INSERT` manual lo salta (§ «Cómo se lee este documento»).

## 4. Decisiones estructurales implicadas por el modelo

| Decisión | Razón | Fuente |
|---|---|---|
| El esquema lo poseen las migraciones | Todo el DDL pasa por un solo camino | §3.2 |
| Concurrencia optimista con `xmin` | Evita sobreventa al descontar stock | §3 (nota `xmin`), §2.2 |
| Baja lógica de productos (`deleted_at`) | Las líneas de venta y el reporte dependen de la fila | §2.2, §7.1 |
| Nombre, precio y categoría congelados en `SaleItem` | Reprecificar o renombrar no reescribe el histórico | §1, §2.4 |
| Total y subtotal se calculan, no se almacenan | Evita dos fuentes de verdad | §1 |
| Reporte no persistido | Se calcula en el motor sobre un rango de fechas | §1, §6.2 |
| Reporte agrupa por la categoría congelada | Un reporte cerrado no debe cambiar | §11.1 |
| Sistema monomoneda, sin columnas de moneda | Decisión cerrada | §3 |
| Sin columnas de auditoría | No hay requisito | §8 |

## 5. Supuestos

- **Supuesto:** el sistema se expone mediante una API; el modelo la menciona pero su contrato queda fuera de alcance (§12).
- **Supuesto:** la implementación usa C# con un ORM (el modelo cita `Sale.Items`, `DbSet<Product>` y a EF), sobre PostgreSQL 16 (§0, §3.2).

## 6. Verificación de coherencia

*(Pendiente: se completa al terminar el paso de cierre, comparando con requisitos, producto, dominio y contexto.)*
