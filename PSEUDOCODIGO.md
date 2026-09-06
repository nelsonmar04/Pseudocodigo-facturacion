# Pseudocódigo — Sistema de Facturación (Arquitectura en 4 Capas)

Cada bloque corresponde a una de las
capas de la arquitectura (Presentación → Aplicación → Negocio → Repositorio),
y muestra cómo cada una delega en la siguiente sin conocer sus detalles
internos.

## Índice
- [1. Capa de Aplicación — Orquestación](#1-capa-de-aplicación--orquestación)
- [2. Capa de Negocio (Dominio) — Reglas de facturación](#2-capa-de-negocio-dominio--reglas-de-facturación)
- [3. Capa de Repositorio — Acceso a datos SQL](#3-capa-de-repositorio--acceso-a-datos-sql)
- [Nota sobre el desacoplamiento](#nota-sobre-el-desacoplamiento)

---

## 1. Capa de Aplicación — Orquestación

Esta capa se encarga de coordinar todo el proceso para crear una factura y organiza el trabajo entre las diferentes capas del sistema. Para esto delega las reglas y validaciones a la Capa de Dominio mientras que todo lo relacionado con guardar la información y actualizar los datos se envia a la Capa de Repositorio. Tambien se asegura de que la creación de la factura y la actualización del inventario se realicen dentro de la misma transacción para evitar que una operación quede realizada y la otra no en caso de que ocurra algun error


```
// ==========================================================
// CAPA DE APLICACIÓN — Caso de uso: crearFactura
// ==========================================================
FUNCION crearFactura(clienteId, itemsList):
    INICIAR_TRANSACCION()
    TRY:
        cliente = RepositorioCliente.buscarPorId(clienteId)
        SI cliente ES NULO:
            LANZAR ErrorNegocio("Cliente no encontrado")

        // Delega la validación de reglas de negocio al Dominio
        factura = ServicioFacturacion.crear(cliente, itemsList)

        // Persistencia de la factura (Repositorio)
        facturaId = RepositorioFactura.guardar(factura)

        // Actualización de inventario (misma transacción)
        PARA CADA linea EN factura.lineas:
            RepositorioProducto.actualizarStock(
                linea.productoCodigo,
                -linea.cantidad
            )

        CONFIRMAR_TRANSACCION()
        RETORNAR Resultado.exito(facturaId)

    CATCH ErrorNegocio AS e:
        REVERTIR_TRANSACCION()
        RETORNAR Resultado.error(e.mensaje)
    CATCH ErrorBaseDeDatos AS e:
        REVERTIR_TRANSACCION()
        RETORNAR Resultado.error("No fue posible registrar la factura")
FIN FUNCION
```

---

## 2. Capa de Negocio (Dominio) — Reglas de facturación

Esta capa contiene las reglas principales que permiten verificar si una factura se puede generar correctamente teniendo en cuenta aspectos como el estado del cliente la disponibilidad de productos en el inventario y el calculo del subtotal el IVA y el total de la factura. Tambien se encarga de controlar la anulación de las facturas evitando que una factura que ya fue anulada pueda volver a modificarse o anularse


```
// ==========================================================
// CAPA DE NEGOCIO (Dominio) — Reglas de facturación
// ==========================================================
FUNCION ServicioFacturacion.crear(cliente, itemsList):
    SI cliente.estado EN ["Inactivo", "BloqueadoPorMora"]:
        LANZAR ErrorNegocio("El cliente no puede ser facturado")

    lineas = LISTA_VACIA
    subtotal = 0
    totalImpuestos = 0

    PARA CADA item EN itemsList:
        producto = RepositorioProducto.buscarPorCodigo(item.codigo)
        SI producto ES NULO:
            LANZAR ErrorNegocio("Producto no existe: " + item.codigo)
        SI item.cantidad > producto.stockDisponible:
            LANZAR ErrorNegocio("Stock insuficiente para " + producto.descripcion)

        valorLinea = producto.precioUnitario * item.cantidad
        ivaLinea = valorLinea * producto.porcentajeIVA

        lineas.agregar(NuevaLineaFactura(
            producto.codigo, item.cantidad,
            producto.precioUnitario, ivaLinea
        ))
        subtotal += valorLinea
        totalImpuestos += ivaLinea

    factura = NuevaFactura(
        cliente.id, lineas, subtotal,
        totalImpuestos, subtotal + totalImpuestos,
        estado = "Emitida"
    )
    RETORNAR factura
FIN FUNCION

FUNCION anularFactura(factura):
    SI factura.estado == "Anulada":
        LANZAR ErrorNegocio("La factura ya se encuentra anulada")
    factura.estado = "Anulada"
    RETORNAR factura
FIN FUNCION
```

---

## 3. Capa de Repositorio — Acceso a datos SQL

Esta es la única capa del sistema que se encarga de crear y ejecutar las sentencias SQL para trabajar directamente con la base de datos. Su funcionamiento se realiza por medio de una interfaz que permite que las demas capas utilicen sus funciones sin tener que conocer que motor de base de datos se esta utilizando ya sea MySQL PostgreSQL Oracle o SQLite. De esta manera se puede cambiar la base de datos en el futuro sin tener que modificar el resto del sistema


```
// ==========================================================
// CAPA DE REPOSITORIO — Acceso a datos (pseudocódigo modificado)
// Independiente del motor de base de datos (MySQL / PostgreSQL / ...)
// ==========================================================
INTERFAZ RepositorioFactura:
    guardar(factura) : facturaId
    buscarPorId(facturaId) : Factura
    buscarFacturasPorClienteYFecha(clienteId, fechaInicio, fechaFin) : Lista<Factura>

CLASE RepositorioFacturaSQL IMPLEMENTA RepositorioFactura:

    FUNCION guardar(factura):
        sql = "INSERT INTO facturas " +
              "(cliente_id, subtotal, impuestos, total, estado, fecha) " +
              "VALUES (?, ?, ?, ?, ?, NOW())"
        facturaId = conexion.ejecutar(sql, [
            factura.clienteId, factura.subtotal,
            factura.impuestos, factura.total, factura.estado
        ])

        PARA CADA linea EN factura.lineas:
            sqlLinea = "INSERT INTO lineas_factura " +
                       "(factura_id, producto_codigo, cantidad, " +
                       "precio_unitario, iva) VALUES (?, ?, ?, ?, ?)"
            conexion.ejecutar(sqlLinea, [
                facturaId, linea.productoCodigo, linea.cantidad,
                linea.precioUnitario, linea.iva
            ])
        RETORNAR facturaId

    FUNCION buscarFacturasPorClienteYFecha(clienteId, fechaInicio, fechaFin):
        sql = "SELECT * FROM facturas " +
              "WHERE cliente_id = ? " +
              "AND fecha BETWEEN ? AND ? " +
              "ORDER BY fecha DESC"
        filas = conexion.consultar(sql, [clienteId, fechaInicio, fechaFin])
        RETORNAR mapearAFacturas(filas)

    FUNCION buscarPorId(facturaId):
        sql = "SELECT * FROM facturas WHERE id = ?"
        fila = conexion.consultarUno(sql, [facturaId])
        RETORNAR mapearAFactura(fila)

CLASE RepositorioProductoSQL:

    FUNCION actualizarStock(codigo, delta):
        sql = "UPDATE productos " +
              "SET stock_disponible = stock_disponible + ? " +
              "WHERE codigo = ?"
        conexion.ejecutar(sql, [delta, codigo])

    FUNCION buscarPorCodigo(codigo):
        sql = "SELECT * FROM productos WHERE codigo = ?"
        fila = conexion.consultarUno(sql, [codigo])
        RETORNAR mapearAProducto(fila)
```

---

## Nota sobre el desacoplamiento

Ninguna de las capas superiores al Repositorio tiene que trabajar directamente con SQL. Esto permite que si en algun momento la empresa decide cambiar de MySQL a PostgreSQL solamente sea necesario modificar la implementación de **RepositorioFacturaSQL** y **RepositorioProductoSQL** junto con el controlador de conexión y posiblemente algunos detalles propios de la sintaxis de la nueva base de datos. Las capas de Presentación Aplicación y Negocio no tendrian que ser modificadas ya que trabajan por medio de las interfaces **RepositorioFactura** y **RepositorioProducto** y no dependen directamente de una implementación especifica. De esta forma el sistema se vuelve más flexible y facil de mantener cuando se necesitan realizar cambios en la tecnologia utilizada


