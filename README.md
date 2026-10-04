# Práctica — Sistema para Ferretería (Desarrollo)

## 1) Narración de requisitos (Descripción del problema y solución)

Al analizar las necesidades de una ferretería local, identifiqué un problema crítico: el negocio lleva el registro de su inventario y ventas en cuadernos, lo que provoca un descontrol total. Se desconoce el stock real de los artículos, se pierden los datos de contacto de quienes suministraron la mercadería cuando hay fallas, y el cálculo manual al momento de cobrar retrasa la atención al cliente. 

Para resolver esto, la base de datos gestionará a los **Proveedores** (para tener sus datos exactos y saber a quién realizar reclamos o pedidos); estructurará el inventario agrupando los **Productos** en **Categorías** (registrando el código de barras, precio y stock real para un control exacto); y registrará cada **Venta** al mostrador, almacenando el detalle de los artículos vendidos para generar un ticket rápido y descontar automáticamente el stock, asegurando que los datos del sistema siempre coincidan con la cantidad física en los estantes.

## 2) Suposiciones (decisiones para aclarar ambigüedades)

* Dado que las ventas son rápidas y al contado ("nada de fiar"), no es necesario llevar un registro complejo de clientes con cuentas corrientes. Se registrará el nombre o NIT del cliente directamente en la Venta de forma opcional para el ticket.
* Un producto pertenece a una sola Categoría (ej. Un martillo pertenece a "Herramientas").
* Para simplificar el control y los reclamos directos, asumiremos que un producto específico es suministrado por un único Proveedor principal (Relación 1:N entre Proveedor y Producto). 
* Una Venta (Ticket) puede incluir varios productos diferentes. Esto requiere una tabla intermedia (Detalle_Venta) para registrar la cantidad y el precio al momento exacto de la venta.
* El código de barras será tratado como un campo único (`UNIQUE`), pero usaremos un `id_producto` autoincremental como Clave Primaria para mayor eficiencia en la base de datos.

## 3) Identificación de Entidades, Atributos, Tipos y PK

```text
┌────────────────────────────────────────────┐
│ PROVEEDOR                                  │
├────────────────────────────────────────────┤
│ + id_proveedor: INTEGER PK (AUTOINCREMENT) │
│ + razon_social: VARCHAR(100) NOT NULL      │
│ + nombre_contacto: VARCHAR(100)            │
│ + telefono: VARCHAR(20) NOT NULL           │
│ + email: VARCHAR(100)                      │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ CATEGORIA                                  │
├────────────────────────────────────────────┤
│ + id_categoria: INTEGER PK (AUTOINCREMENT) │
│ + nombre: VARCHAR(50) UNIQUE NOT NULL      │
│ + descripcion: VARCHAR(255)                │
└────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRODUCTO                                                        │
├─────────────────────────────────────────────────────────────────┤
│ + id_producto: INTEGER PK (AUTOINCREMENT)                       │
│ + codigo_barras: VARCHAR(50) UNIQUE NOT NULL                    │
│ + nombre: VARCHAR(100) NOT NULL                                 │
│ + id_categoria: INTEGER FK → CATEGORIA(id_categoria) NOT NULL   │
│ + id_proveedor: INTEGER FK → PROVEEDOR(id_proveedor) NOT NULL   │
│ + precio_unitario: DECIMAL(8,2) CHECK (precio_unitario > 0)     │
│ + stock_actual: INTEGER DEFAULT 0 CHECK (stock_actual >= 0)     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ VENTA                                                           │
├─────────────────────────────────────────────────────────────────┤
│ + id_venta: INTEGER PK (AUTOINCREMENT)                          │
│ + fecha_hora: DATETIME DEFAULT CURRENT_TIMESTAMP                │
│ + nombre_cliente: VARCHAR(100) DEFAULT 'Consumidor Final'       │
│ + total_venta: DECIMAL(10,2) DEFAULT 0                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ DETALLE_VENTA                                                   │
├─────────────────────────────────────────────────────────────────┤
│ + id_venta: INTEGER FK → VENTA(id_venta) PK parcial             │
│ + id_producto: INTEGER FK → PRODUCTO(id_producto) PK parcial    │
│ + cantidad: INTEGER CHECK (cantidad > 0) NOT NULL               │
│ + precio_unitario: DECIMAL(8,2) NOT NULL                        │
│ + subtotal: DECIMAL(10,2) NOT NULL                              │
└─────────────────────────────────────────────────────────────────┘
```

## 4) Relaciones y cardinalidades (con justificación)

* **CATEGORIA (1) — (N) PRODUCTO**
  * *Justificación:* Una categoría agrupa a muchos productos, pero un producto específico pertenece a una sola categoría para mantener el inventario lógicamente ordenado.
* **PROVEEDOR (1) — (N) PRODUCTO**
  * *Justificación:* Un proveedor puede suministrar múltiples productos. Cada producto está asociado a un proveedor principal para saber a quién dirigir los reclamos o pedidos de reposición.
* **VENTA (1) — (N) DETALLE_VENTA**
  * *Justificación:* Un ticket de venta puede contener varios productos (líneas de detalle). Cada línea de detalle pertenece a una única venta registrada.
* **PRODUCTO (1) — (N) DETALLE_VENTA**
  * *Justificación:* Un mismo producto del catálogo de la ferretería puede ser vendido en múltiples transacciones distintas a lo largo del tiempo.

## 5) Reglas de negocio y restricciones importantes

* **Control de Stock (Inventario en tiempo real):** Al insertar un nuevo registro en `DETALLE_VENTA`, el sistema debe descontar automáticamente la `cantidad` vendida del `stock_actual` en la tabla `PRODUCTO`. Esto se implementa mediante un `TRIGGER` (Disparador) en la base de datos.
* **Restricción de Stock Negativo:** El atributo `stock_actual` en la tabla `PRODUCTO` tiene un `CHECK (stock_actual >= 0)`. El sistema rechazará a nivel de base de datos cualquier intento de venta de un producto si la cantidad solicitada supera el stock disponible.
* **Congelamiento de Precios Históricos:** En la tabla `DETALLE_VENTA` se copia el `precio_unitario` del producto en el momento de la venta. Esto garantiza que, si el precio de un producto cambia en el futuro, el historial financiero de ventas pasadas se mantenga intacto.
* **Ausencia de Cuentas por Cobrar:** Al ser todas las ventas al contado, no existen estados de "pendiente" o "pagado a medias"; toda fila creada en la tabla `VENTA` se considera una transacción finalizada y abonada al 100%.
