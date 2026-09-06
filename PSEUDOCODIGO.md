# Pseudocódigo — Sistema de Facturación (Arquitectura en 4 Capas)

Este documento describe, capa por capa, la lógica del caso de uso principal
del sistema: **crear una factura**. Cada bloque corresponde a una de las
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

Coordina el flujo completo del caso de uso `crearFactura`, delega las reglas
de negocio a la Capa de Dominio y la persistencia a la Capa de Repositorio,
y garantiza que la factura y la actualización del inventario ocurran dentro
de la misma transacción.

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

Contiene las reglas que determinan si una factura es válida: estado del
cliente, disponibilidad de stock, cálculo del subtotal/IVA/total, y el
control de anulación de facturas.

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

Es la única capa que construye y ejecuta sentencias SQL. Expone su
funcionalidad a través de una interfaz, de modo que el resto del sistema
nunca depende del motor de base de datos concreto (MySQL, PostgreSQL,
Oracle, SQLite, etc.).

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

Ninguna capa por encima del Repositorio hace referencia directa a SQL.
Si mañana la empresa migra de MySQL a PostgreSQL, solo se modifica la
implementación de `RepositorioFacturaSQL` y `RepositorioProductoSQL`
(el driver de conexión y, si acaso, alguna sintaxis específica del motor).
Las capas de Presentación, Aplicación y Negocio permanecen intactas porque
dependen únicamente de las interfaces (`RepositorioFactura`,
`RepositorioProducto`), nunca de su implementación concreta.

> La implementación real en Java + Spring Boot de este pseudocódigo está
> en [`/src/main/java/com/tienda/facturacion`](./src/main/java/com/tienda/facturacion),
> organizada en los mismos cuatro paquetes: `presentation`, `application`,
> `domain` y `repository`.
