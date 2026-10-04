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
┌─────────────┐
│ PROVEEDOR   │
├─────────────┤
│ + id_proveedor: INTEGER PK (AUTOINCREMENT)
│ + razon_social: VARCHAR(100) NOT NULL
│ + nombre_contacto: VARCHAR(100)
│ + telefono: VARCHAR(20) NOT NULL
│ + email: VARCHAR(100)
└─────────────┘

┌─────────────┐
│ CATEGORIA   │
├─────────────┤
│ + id_categoria: INTEGER PK (AUTOINCREMENT)
│ + nombre: VARCHAR(50) UNIQUE NOT NULL
│ + descripcion: VARCHAR(255)
└─────────────┘

┌─────────────┐
│ PRODUCTO    │
├─────────────┤
│ + id_producto: INTEGER PK (AUTOINCREMENT)
│ + codigo_barras: VARCHAR(50) UNIQUE NOT NULL
│ + nombre: VARCHAR(100) NOT NULL
│ + id_categoria: INTEGER FK → CATEGORIA(id_categoria) NOT NULL
│ + id_proveedor: INTEGER FK → PROVEEDOR(id_proveedor) NOT NULL
│ + precio_unitario: DECIMAL(8,2) CHECK (precio_unitario > 0)
│ + stock_actual: INTEGER DEFAULT 0 CHECK (stock_actual >= 0)
└─────────────┘

┌─────────────┐
│ VENTA       │
├─────────────┤
│ + id_venta: INTEGER PK (AUTOINCREMENT)
│ + fecha_hora: DATETIME DEFAULT CURRENT_TIMESTAMP
│ + nombre_cliente: VARCHAR(100) DEFAULT 'Consumidor Final'
│ + total_venta: DECIMAL(10,2) DEFAULT 0
└─────────────┘

┌───────────────┐
│ DETALLE_VENTA │
├───────────────┤
│ + id_venta: INTEGER FK → VENTA(id_venta) PK parcial
│ + id_producto: INTEGER FK → PRODUCTO(id_producto) PK parcial
│ + cantidad: INTEGER CHECK (cantidad > 0) NOT NULL
│ + precio_unitario: DECIMAL(8,2) NOT NULL
│ + subtotal: DECIMAL(10,2) NOT NULL
└───────────────┘
