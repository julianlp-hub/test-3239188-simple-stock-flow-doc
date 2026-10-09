# Requisitos — Simple Stock Flow

> **Fuente única:** `spec/data-model.md`. Las citas §x.y y Qn remiten a ese documento.
> Lo que no sale del modelo está marcado como **Supuesto**.

## 1. Actores

| Actor | Descripción | Fuente |
|---|---|---|
| Administrador (`admin`) | Operador interno con privilegios altos | §1, §2.5 |
| Vendedor (`seller`) | Operador interno que registra ventas | §1, §2.5 |

No existe actor «cliente» ni «comprador» (§1).
**Supuesto:** qué puede hacer exactamente cada rol (el modelo solo dice que son dos y que el `admin` da de alta vendedores, §11 H-3).

## 2. Historias de usuario

| ID | Historia | Reglas que la sustentan | Fuente |
|---|---|---|---|
| HU-01 | Como operador, quiero iniciar sesión con mi usuario y contraseña para usar el sistema. | Usuario único, en minúsculas; la contraseña solo se verifica mediante el hash | §2.5, §6.1 (Q10) |
| HU-02 | Como administrador, quiero dar de alta vendedores para que puedan registrar ventas. | El rol `admin` no se otorga en ejecución: lo provisiona el despliegue | §11 (H-3, DP-04) |
| HU-03 | Como operador, quiero buscar productos por texto y categoría para encontrarlos rápido. | Solo productos activos, ordenados por nombre, con paginación | §6.1 (Q1) |
| HU-04 | Como operador, quiero ver las categorías disponibles para clasificar productos. | Son cinco, fijas y de solo lectura | §2.1, §9.1 (Q4) |
| HU-05 | Como operador, quiero crear y editar productos (nombre, precio, stock, categoría) para mantener el catálogo. | `price > 0`; nombre obligatorio y recortado; categoría obligatoria y existente | §2.2 |
| HU-06 | Como operador, quiero dar de baja un producto sin perder su historial. | Baja lógica con `deleted_at`; nunca borrado físico | §2.2, §7.1 |
| HU-07 | Como operador, quiero asociar una imagen a un producto. | Se guarda una clave opaca; sin imagen es `NULL`, nunca cadena vacía | §1, §2.2 |
| HU-08 | Como vendedor, quiero registrar una venta con varias líneas (producto y cantidad). | Al menos una línea; un producto no se repite en la misma venta; `quantity > 0` | §2.3, §2.4 |
| HU-09 | Como vendedor, quiero que el stock se descuente al agregar una línea y que no me deje vender más de lo disponible. | Descontar stock y añadir la línea es una sola operación; `stock >= 0` | §2.2, §2.3 |
| HU-10 | Como operador, quiero consultar las ventas de un rango de fechas. | Orden por fecha descendente, con paginación | §6.1 (Q7) |
| HU-11 | Como administrador, quiero un reporte de ventas por producto en un rango de fechas. | Agrupa por producto y por categoría congelada; el fin del rango no puede ser anterior al inicio | §1, §6.1 (Q9), §11.1 |

## 3. Requisitos no funcionales

| ID | Categoría | Requisito | Fuente |
|---|---|---|---|
| RNF-01 | Integridad | El stock nunca es negativo; lo garantiza el motor como última barrera | §2.2 (`ck_product_stock_non_negative`) |
| RNF-02 | Concurrencia | Dos ventas simultáneas sobre el mismo producto no deben sobrevender; se usa `xmin` como testigo de concurrencia | §3 (`xmin`), §6.1 (Q3) |
| RNF-03 | Inmutabilidad | Una venta registrada no se edita ni se borra; no existe operación que lo permita | §2.3, §7.1 |
| RNF-04 | Trazabilidad histórica | La línea de venta guarda copia congelada de nombre, precio y categoría, para que el reporte de un período cerrado no cambie | §1, §2.4, §11.1 |
| RNF-05 | Exactitud monetaria | Importes en `numeric(18,2)` y redondeo a 2 decimales (`AwayFromZero`); sistema monomoneda | §2.2, §3 |
| RNF-06 | Cálculo | Total y subtotal se calculan, no se almacenan | §1 |
| RNF-07 | Tiempo | Todas las marcas de tiempo son `timestamptz`; el servidor corre en UTC | §3 |
| RNF-08 | Privacidad | El hash de contraseña nunca aparece en logs, respuestas ni errores, y nunca se indexa; el reporte no se desglosa por vendedor | §7, §7.1 (DP-02) |
| RNF-09 | Rendimiento | La búsqueda (Q1), el listado de ventas (Q7) y el reporte (Q9) usan índices dedicados; el reporte se calcula en el motor | §6.1, §6.2 |
| RNF-10 | Mantenibilidad | Todo el DDL se define mediante migraciones; el motor prevalece sobre el documento si hay contradicción | § «Rige bajo», §3.2 |
| RNF-11 | Integridad referencial | Políticas `ON DELETE` definidas: FK-1, FK-3 y FK-4 en `RESTRICT`; FK-2 en `CASCADE` | §5 |
| RNF-12 | Retención | Ventas y líneas se conservan indefinidamente; el único dato que se borra físicamente es el binario de imagen | §7.1 |

## 4. Fuera de alcance
No hay clientes, pagos, multimoneda, auditoría `created_at`/`updated_at`, ni mantenimiento de categorías (§1, §2.1, §3, §8).

## 5. Pendientes del modelo que afectan requisitos
- `sale.sold_by_user_id` y FK-4: pendientes (T-12), §5.
- Índices de búsqueda y reporte: pendientes (T-13), §6.2.
- `CHECK` de `price > 0`, `quantity > 0`, rol y nombre no vacío: hoy solo en dominio (T-20), §4.
- **Posible contradicción:** `spec.md` CA-06.1 dice «una fila por producto», pero §11.1 produce más de una si hubo recategorización. Está pendiente de decisión del propietario.
