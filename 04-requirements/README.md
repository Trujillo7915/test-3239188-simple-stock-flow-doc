# 04 - Requisitos del Sistema (Simple Stock Flow)

> **Documento SDD · Fase 2 (Reconstrucción Inversa)**  
> **Insumo:** `spec/data-model.md`  
> **Estado:** Historias de Usuario (HU) y Requisitos No Funcionales (RNF) derivados del Modelo de Datos  

---

## 1. Introducción y Enfoque

En la metodología SDD inversa, los requisitos del sistema se descubren a partir de las restricciones del esquema físico, las invariantes de dominio (`§2`), las políticas de integridad referencial (`§5`) y los patrones de acceso reales (`§6.1`). El modelo de datos de *Simple Stock Flow* demuestra que el sistema está diseñado como un punto de venta e inventario interno de alta integridad, transaccionalidad atómica y retención inmutable.

---

## 2. Historias de Usuario (HU)

### Épica 1: Gestión de Catálogo e Inventario

#### `HU-CAT-01`: Búsqueda y listado paginado de productos activos
* **Como:** Operador del sistema (`admin` o `seller`).
* **Quiero:** Buscar productos en el catálogo mediante texto parcial o filtrando por categoría, obteniendo una lista paginada de productos activos.
* **Para:** Localizar rápidamente artículos durante la venta o revisión de catálogo sin sobrecargar la red ni el motor de base de datos.
* **Criterios de Aceptación (Gherkin):**
  - **CA-01.1 (Exclusión de bajas lógicas):** Solo deben retornar productos cuyo atributo sombra `deleted_at` sea `NULL` (`§2.2`, `T-09`, `ADR-003`).
  - **CA-01.2 (Búsqueda optimizada):** La consulta debe resolver los filtros combinados de categoría y nombre sobre productos activos mediante índice parcial (`§6.1 Q1`, `§6.2`, `T-13`).
  - **CA-01.3 (Paginación y conteo):** La respuesta debe incluir los ítems de la página solicitada y el conteo total de coincidencias (`§6.1 Q1`).

#### `HU-CAT-02`: Creación y actualización de producto
* **Como:** Operador con rol `admin`.
* **Quiero:** Registrar nuevos productos y modificar su nombre, precio, categoría e imagen asociada.
* **Para:** Mantener actualizado el catálogo disponible para la venta.
* **Criterios de Aceptación:**
  - **CA-02.1 (Atributos estrictos):** El producto solo admite nombre, precio, stock, categoría e imagen opcional; no se admiten descripciones, SKUs ni códigos adicionales (`DP-03`, `§1`).
  - **CA-02.2 (Categoría válida obligatoria):** La categoría asignada debe existir en la tabla `category` (`FK-1`, `RESTRICT`, `§5`).
  - **CA-02.3 (Reglas de precio y formato):** El precio debe ser estrictamente mayor a cero (`price > 0`, `§2.2`, `T-20`) y redondearse a 2 decimales bancarios (`MidpointRounding.AwayFromZero`, `numeric(18,2)`, `§2.2`).
  - **CA-02.4 (Manejo de imagen opaca):** Si no se proporciona imagen, se almacena `NULL` (nunca cadena vacía). La imagen se referencia mediante su clave opaca `image_key` (`D-08`, `§2.2`, `§3`).

#### `HU-CAT-03`: Baja lógica de producto
* **Como:** Operador con rol `admin`.
* **Quiero:** Dar de baja un producto del catálogo que ya no se comercializará.
* **Para:** Evitar que se siga vendiendo sin destruir la integridad histórica de ventas pasadas.
* **Criterios de Aceptación:**
  - **CA-03.1 (No borrado físico):** El sistema nunca ejecuta un `DELETE` físico sobre la tabla `product`. En su lugar, estampa la fecha actual en la propiedad sombra `deleted_at` (`§2.2`, `§7.1`, `ADR-003`).
  - **CA-03.2 (Barrera de integridad):** Si se intentara un borrado físico directo en la base de datos, el motor debe abortar la operación mediante la clave foránea restrictiva hacia líneas de venta (`FK-3`, `FK_sale_item_product_product_id`, `ON DELETE RESTRICT`, `§4`, `§5`, `T-20`).
  - **CA-03.3 (Eliminación de binario):** Al dar de baja un producto con imagen, primero se anula `image_key` en base de datos y posteriormente se solicita la eliminación del binario en el almacenamiento (`§7.1`, `D-08`).

#### `HU-CAT-04`: Consulta de categorías del catálogo
* **Como:** Operador del sistema.
* **Quiero:** Consultar el listado de categorías disponibles.
* **Para:** Seleccionar la categoría adecuada al filtrar o registrar productos.
* **Criterios de Aceptación:**
  - **CA-04.1 (Repositorio de solo lectura):** El sistema expone las 5 categorías sembradas: General, Herramientas, Electricidad, Fontanería y Pinturas (`§2.1`, `§9.1`, `D-10`).
  - **CA-04.2 (Sin mantenimiento):** No existe ningún puerto ni endpoint que permita crear, editar o eliminar categorías (`§2.1`, `§4.1`).

---

### Épica 2: Registro y Gestión de Ventas

#### `HU-VTA-01`: Registro atómico de venta con descuento de inventario
* **Como:** Operador autenticado (`admin` o `seller`).
* **Quiero:** Registrar una venta compuesta por uno o varios productos, indicando las cantidades vendidas.
* **Para:** Formalizar la transacción comercial y actualizar las existencias de inventario de forma inmediata y consistente.
* **Criterios de Aceptación:**
  - **CA-01.1 (Atomicidad indivisible):** El registro de la venta (`sale`), sus renglones (`sale_item`) y el descuento de existencias (`Product.Withdraw`) deben completarse en una sola transacción (`§2.3`).
  - **CA-01.2 (Control de stock no negativo):** Si la cantidad vendida excede el stock disponible de cualquier producto, la operación completa es rechazada sin registrar venta ni descontar inventario parcial (`ck_product_stock_non_negative`, `§2.2`, `ADR-002`).
  - **CA-01.3 (Al menos una línea):** La venta debe contener como mínimo una línea para poder confirmarse (`Sale.EnsureConfirmable`, `§2.3`).
  - **CA-01.4 (No repetición de productos):** No se permite incluir el mismo producto más de una vez en la misma venta (`IX_sale_item_sale_id_product_id`, `§2.3`, `T-20`).
  - **CA-01.5 (Congelamiento de hechos comerciales):** La línea de venta debe clonar y almacenar de forma fija: `product_name` actual, `unit_price` vigente y `category_name` (`§1`, `§2.4`, `D-06`, `T-11`, `ADR-004`).
  - **CA-01.6 (Concurrencia optimista):** Ante intentos concurrentes de venta sobre el mismo producto, el sistema valida la versión vía `xmin`; si hay choque, rechaza con conflicto de concurrencia (`§3`, `D-04`, `T-10`).
  - **CA-01.7 (Atribución y fecha):** La venta guarda la marca de tiempo `sold_at` en UTC (`§3`) y el nombre del operador `sold_by` (y `sold_by_user_id` pendiente `T-12`, `FK-4`).

#### `HU-VTA-02`: Consulta de comprobante de venta y listado por rango
* **Como:** Operador autenticado.
* **Quiero:** Consultar el detalle de una venta específica o listar ventas realizadas en un período de tiempo.
* **Para:** Verificar compras pasadas o auditar el flujo comercial de una jornada.
* **Criterios de Aceptación:**
  - **CA-02.1 (Fidelidad histórica):** La consulta de una venta recupera los precios y nombres congelados en `sale_item`, garantizando que cambios posteriores en el catálogo no alteren el comprobante (`§1`, `§2.4`).
  - **CA-02.2 (Orden cronológico):** El listado por fechas se ordena descendentemente por `sold_at` y se sirve eficientemente mediante `IX_sale_sold_at` (`§6.1 Q7`, `§6.2`).
  - **CA-02.3 (Inmutabilidad estricta):** El sistema no provee ninguna función de anulación, modificación o borrado de ventas (`§1`, `§2.3`, `§7.1`).

---

### Épica 3: Reportes Comerciales

#### `HU-REP-01`: Reporte consolidado de ventas por rango de fechas
* **Como:** Operador con rol `admin`.
* **Quiero:** Generar un reporte consolidado de ventas dentro de una ventana temporal especificada.
* **Para:** Analizar el desempeño de ventas por producto e importe total sin degradar el rendimiento operacional.
* **Criterios de Aceptación:**
  - **CA-01.1 (Validación de ventana):** La fecha fin del rango no puede ser anterior a la fecha de inicio (`§1`).
  - **CA-01.2 (Agregación directa en motor):** El reporte no carga objetos de dominio; se ejecuta directamente en PostgreSQL agrupando por producto e importe total descendente (`§1`, `§6.1 Q9`, `D-06`, `ADR-004`).
  - **CA-01.3 (Agrupación por valor congelado):** Si un producto cambió de categoría o nombre en el tiempo, el reporte agrupa estrictamente por los valores congelados (`category_name`, `product_name`) sin reescribir históricos (`§11.1`, `H-1`).
  - **CA-01.4 (Sin desglose por vendedor):** El reporte consolida volúmenes por artículo; no expone ni segmenta métricas por operador para proteger datos personales (`DP-02`, `§6.3`, `§7`).

---

### Épica 4: Identidad y Seguridad

#### `HU-SEC-01`: Autenticación de operadores internos
* **Como:** Operador del sistema (`admin` o `seller`).
* **Quiero:** Iniciar sesión con mi nombre de usuario y contraseña.
* **Para:** Acceder a las funciones autorizadas según mi rol.
* **Criterios de Aceptación:**
  - **CA-01.1 (Normalización de usuario):** El `username` se evalúa siempre recortado y en minúsculas (`User.NormalizeUsername`, `§2.5`).
  - **CA-01.2 (Custodia de secretos):** La contraseña nunca se almacena ni viaja en claro hacia el dominio; la autenticación valida el hash criptográfico (`password_hash`, `D-09`, `§1`, `§2.5`).
  - **CA-01.3 (Roles estrictos):** El usuario posee exactamente uno de dos roles permitidos: `admin` o `seller` (`§2.5`, `§3`).

---

## 3. Requisitos No Funcionales (RNF)

| Código | Requisito No Funcional | Categoría | Especificación y Trazabilidad al Modelo |
|---|---|---|---|
| **RNF-01** | **Atomicidad e Integridad Transaccional (ACID)** | Confiabilidad | Toda operación de venta que involucre múltiples líneas e inventario debe ejecutarse bajo una transacción atómica; si una línea falla, no se descuenta stock (`ADR-002`, `§2.3`). |
| **RNF-02** | **Concurrencia Optimista sin Bloqueo Pesimista** | Rendimiento / Concurrencia | El agregado `Product` debe detectar colisiones de escritura simultánea en stock mediante la columna de sistema `xmin` de PostgreSQL (`D-04`, `T-10`, `§3`). |
| **RNF-03** | **Inmutabilidad y Preservación Histórica** | Integridad de Datos | Los registros de `sale` y `sale_item` tienen retención indefinida y carecen de operaciones de actualización o borrado físico (`§7.1`, `ADR-004`). |
| **RNF-04** | **Arquitectura Monomoneda por Construcción** | Negocio / Arquitectura | El sistema opera bajo una única moneda estándar. Ninguna tabla del esquema `sales` almacena códigos de divisa (`D-05`, `§1`, `§3`). |
| **RNF-05** | **Precisión Numérica y Financiera** | Precisión | Todos los valores monetarios se gestionan con escala fija a dos decimales (`numeric(18,2)`) y redondeo `MidpointRounding.AwayFromZero` (`§2.2`, `§3`). |
| **RNF-06** | **Privacidad y Protección de Datos Atributo por Atributo** | Seguridad | `password_hash` nunca se expone en logs, trazas ni respuestas API (`§7`). `user.username` y `sale.sold_by` se clasifican como datos personales de acceso restringido (`§7`). No se recopilan datos de clientes compradores (`§1`, `§7`). |
| **RNF-07** | **Optimización Indexada de Lectura (Index-Only Scan)** | Rendimiento | La consulta de reporte de ventas Q9 se optimiza con el índice compuesto `(sale_id, product_id) INCLUDE (quantity, unit_price)` para evitar accesos a tabla (`§6.1`, `§6.2`, `T-13`). |
| **RNF-08** | **Estandarización Temporal UTC** | Interoperabilidad | Todas las marcas de tiempo se almacenan como `timestamptz` con servidor sincronizado en UTC (`§3`, `§10.1`). |
| **RNF-09** | **Integridad Referencial en Motor** | Robustez | El motor de base de datos aplica restricciones foráneas estrictas (`ON DELETE RESTRICT` en FK-1, FK-3 y FK-4; `ON DELETE CASCADE` en FK-2) (`§5`). |

---

## 4. Matriz de Trazabilidad: Requisitos vs. Modelo de Datos

| Requisito | Entidades involucradas | Restricciones / Índices del Motor | Secciones del Modelo |
|---|---|---|---|
| `HU-CAT-01` | `product`, `category` | `IX_product_category_id_name` (parcial `deleted_at IS NULL`) | `§2.2`, `§6.1 Q1`, `§6.2` |
| `HU-CAT-02` | `product`, `category` | `PK_product`, `FK-1` (`RESTRICT`), `ck_product_stock_non_negative` | `§2.2`, `§3`, `§5`, `DP-03`, `D-08` |
| `HU-CAT-03` | `product`, `sale_item` | `deleted_at`, `FK-3` (`RESTRICT`) | `§2.2`, `§5`, `§7.1`, `ADR-003`, `T-09` |
| `HU-CAT-04` | `category` | `PK_category`, `IX_category_name` | `§2.1`, `§3`, `§9.1`, `D-10` |
| `HU-VTA-01` | `sale`, `sale_item`, `product` | `PK_sale`, `PK_sale_item`, `FK-2` (`CASCADE`), `ck_product_stock_non_negative`, `xmin` | `§2.3`, `§2.4`, `§3`, `ADR-002`, `D-04` |
| `HU-VTA-02` | `sale`, `sale_item` | `IX_sale_sold_at`, `FK-2` | `§2.3`, `§6.1 Q6, Q7`, `§7.1` |
| `HU-REP-01` | `sale`, `sale_item` | `IX_sale_item_sale_id_product_id` (INCLUDE) | `§1`, `§6.1 Q9`, `D-06`, `ADR-004`, `§11.1` |
| `HU-SEC-01` | `user` | `PK_user`, `IX_user_username` | `§2.5`, `§3`, `§6.1 Q10`, `D-09` |

---

## 5. Supuestos de Requisitos Declarados

| Identificador | Supuesto de Requisito | Justificación |
|---|---|---|
| **SUP-REQ-01** | La interfaz de usuario muestra el total de la venta calculado al vuelo a partir de los subtotales de las líneas. | El modelo estipula explícitamente que el total de la venta no tiene columna en base de datos y se calcula en tiempo de ejecución (`§1`, artículo VII). |
| **SUP-REQ-02** | No se admite la venta de fracciones de unidades (las cantidades vendidas son siempre enteros positivos). | La columna `sale_item.quantity` y el atributo `product.stock` son de tipo `integer` (`§3`). |
| **SUP-REQ-03** | El alta inicial del usuario administrador se realiza al arrancar el contenedor o servicio mediante variables de entorno. | El modelo indica explícitamente que la base de datos no siembra el usuario admin para evitar exponer claves o duplicar lógica de hash (`§9.2`, `D-09`, `D-10`). |
