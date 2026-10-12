# Tienda Online — EcommerceApp2

### **PP3 — Desarrollo y Aplicación de Sistemas en la Nube · IFTS Nº16**

Docente: Damián Wajser

#

#### **Integrantes**

- Lucía Corral
- Pablo Demartini
- Carla Guisande
- Ignacio Hernandez
- Ignacio Vidal

#

#

## 1. De qué se trata el proyecto

**Tienda Online** es una aplicación web de comercio electrónico dirigida a pequeños comercios que venden productos de tecnología y artículos para el hogar. Permite que sus clientes consulten el catálogo, busquen por nombre, filtren por categoría, agreguen productos al carrito y confirmen una compra. Los usuarios pueden registrarse e iniciar sesión, mientras que el personal autorizado dispone de una dashboard administrativo para gestionar el stock de productos y alta y baja de productos y de usuarios.

El problema que busca resolver es la dispersión de la información de venta: cuando un comercio recibe consultas por mensajes y mantiene precios y existencias en listas separadas, el cliente necesita consultar qué está disponible y el personal debe revisar los datos para cada operación. La aplicación reúne el catálogo, los precios y el stock en una misma plataforma; durante el checkout, el servidor verifica la disponibilidad y calcula el total a partir de los precios guardados en la base de datos.

El público objetivo comprende dos tipos de usuarios: **clientes** que quieren consultar productos y preparar sus compras desde un navegador, y **administradores** que necesitan mantener la información del comercio.

Para los _clientes_, el beneficio es contar con un catálogo consultable y una confirmación inmediata del checkout.
Para los _administradores_, es gestionar los datos desde una interfaz común.

## 2. Arquitectura propuesta y despliegue previsto

El sistema dispone de un frontend para la tienda y el dashboard administrativo. Las solicitudes del frontend ingresan por un API Gateway, que se comunica con las APIs de Usuarios, Catálogo y Pedidos. Cada API administra su propia base de datos.

**El alojamiento indicado es previsto; este documento no certifica que los componentes ya estén desplegados. Los nombres de carpetas son propuestos y deben verificarse contra el repositorio de implementación.**

| Componente                       | Carpeta propuesta          | Tecnología             | Despliegue previsto |
| -------------------------------- | -------------------------- | ---------------------- | ------------------- |
| Frontend Web: tienda y dashboard | `frontend/`                | HTML, CSS y JavaScript | Vercel              |
| API Gateway                      | `api-gateway/`             | Node.js y Express      | Render              |
| API Usuarios                     | `ms-usuarios/`             | Node.js y Express      | Render              |
| API Catálogo                     | `ms-catalogo/`             | Node.js y Express      | Render              |
| API Pedidos                      | `ms-pedidos/`              | Node.js y Express      | Render              |
| DB Usuarios                      | Base exclusiva de Usuarios | MongoDB                | MongoDB Atlas       |
| DB Catálogo                      | Base exclusiva de Catálogo | PostgreSQL             | Render Postgres     |
| DB Pedidos                       | Base exclusiva de Pedidos  | MySQL                  | Railway             |

### Vista general del sistema

![Sistema completo](diagramas/diagArq.jpg?v=2)

[Ver imagen de arquitectura en tamaño completo](diagramas/diagArq.jpg)

## Responsabilidades y comunicación

**Estado del documento:** diseño propuesto. Las rutas, tablas y columnas que siguen requieren acuerdo del equipo y adaptación del código; no se presentan como una implementación verificada.

| Componente   | Responsabilidad                                                       | Base propia               |
| ------------ | --------------------------------------------------------------------- | ------------------------- |
| Frontend Web | Tienda y dashboard administrativo; consume únicamente el Gateway.     | .                         |
| API Gateway  | Valida JWT, enruta solicitudes y coordina la confirmación de pedidos. | .                         |
| API Usuarios | Usuarios, roles y autenticación.                                      | MongoDB en MongoDB Atlas. |
| API Catálogo | Productos, precios, existencias y movimientos de stock.               | PostgreSQL.               |
| API Pedidos  | Pedido, sus detalles y estado.                                        | MySQL.                    |

Cada API accede únicamente a su base. El Gateway no ejecuta SQL. Las solicitudes HTTP incluyen método, ruta y datos; las solicitudes a PostgreSQL y MySQL muestran SQL parametrizado; las consultas a MongoDB muestran operaciones del driver. Las respuestas de las bases muestran resultados, sin códigos HTTP.

La arquitectura general se documenta aparte, con componentes, alojamientos previstos y comunicaciones, sin endpoints. No se utiliza un diagrama de secuencia como sustituto de la arquitectura.

---

## 3. Diagramas de secuencia

Cada diagrama representa una operación independiente y reinicia su numeración. Las imágenes PNG muestran los diagramas renderizados. Cada una incluye un enlace para abrirla en tamaño completo. Los bloques PlantUML permiten mantener sus fuentes; GitHub no renderiza esos bloques automáticamente. No se reúnen autenticación, consulta de productos y compra dentro de una vista general.

### 1 — Crear sesión

Este diagrama muestra únicamente la autenticación de un cliente o administrador. El frontend envía las credenciales al Gateway y API Usuarios busca el usuario en MongoDB Atlas. La contraseña se verifica antes de crear la sesión y emitir el JWT. Se distinguen los casos de campos ausentes, credenciales inválidas y autenticación correcta.

**Contrato propuesto:** POST /api/sesiones crea una sesión autenticada: 201 en el caso nuevo; 400 si faltan campos y 401 si las credenciales no son válidas.

[Ver este diagrama en tamaño completo](diagramas/01-sesion.png)

![1 — Crear sesión](diagramas/01-sesion.png)

<details>
<summary>Ver código PlantUML de este diagrama</summary>

```plantuml
@startuml
skinparam defaultFontName SansSerif
skinparam shadowing false
skinparam maxMessageSize 300
autonumber
title Usuarios — Crear sesión (contrato propuesto)
actor "Cliente o administrador" as U
participant "Frontend" as F
participant "Gateway" as G
participant "API Usuarios" as A
database "MongoDB Atlas" as D
U -> F: Completar email y contraseña
F -> G: POST /api/sesiones\nContent-Type: application/json\n{email, password}
alt Campos obligatorios ausentes
 G --> F: 400 Bad Request\n{message: "Campos obligatorios ausentes"}
 F --> U: Mostrar error
else Campos completos
 G -> A: POST /api/sesiones\nContent-Type: application/json\n{email, password}
 A -> D: usuarios.findOne({email: emailRecibido},\n{projection: {_id: 1, email: 1, passwordHash: 1, rol: 1}})
 D --> A: Documento del usuario o null
 A -> A: Si existe usuario, comparar contraseña con bcrypt
 alt Usuario inexistente o contraseña incorrecta
  A --> G: 401 Unauthorized\n{message: "Credenciales inválidas"}
  G --> F: 401 Unauthorized\n{message: "Credenciales inválidas"}
  F --> U: Mostrar error
 else Credenciales válidas
  A -> A: Crear sesión autenticada y firmar JWT\n{sub: usuarioId, rol, exp}
  A --> G: 201 Created\n{token, usuario: {id, email, rol}}
  G --> F: 201 Created\n{token, usuario: {id, email, rol}}
  F --> U: Mostrar pantalla correspondiente al rol
 end
end
@enduml
```

</details>

<br>

---

<br>

### 2 — Consultar producto

Este diagrama muestra únicamente la consulta del detalle de un producto. El frontend solicita el producto por su identificador; Gateway deriva la solicitud a API Catálogo, que consulta PostgreSQL. La respuesta permite mostrar el detalle o informar que el producto no se encuentra.

**Contrato propuesto:** GET /api/productos/{id}: 200 si existe y está activo; 404 si no se encuentra.

[Ver este diagrama en tamaño completo](diagramas/02-producto.png)

![2 — Consultar producto](diagramas/02-producto.png)

<details>
<summary>Ver código PlantUML de este diagrama</summary>

```plantuml
@startuml
skinparam defaultFontName SansSerif
skinparam shadowing false
skinparam maxMessageSize 300
autonumber
title Catálogo — Consultar producto (contrato propuesto)
actor Cliente as U
participant "Frontend" as F
participant "Gateway" as G
participant "API Catálogo" as A
database "PostgreSQL" as D
U -> F: Seleccionar producto
F -> G: GET /api/productos/{id}
G -> A: GET /api/productos/{id}
A -> D: SELECT id, nombre, descripcion, precio, stock, categoria_id\nFROM productos WHERE id = :id AND activo = TRUE;
D --> A: Una fila o ninguna
alt Producto disponible en el catálogo
 A --> G: 200 OK\n{id, nombre, descripcion, precio, stock, categoriaId}
 G --> F: 200 OK\n{id, nombre, descripcion, precio, stock, categoriaId}
 F --> U: Mostrar detalle
else Producto inexistente o inactivo
 A --> G: 404 Not Found\n{message: "Producto no encontrado"}
 G --> F: 404 Not Found\n{message: "Producto no encontrado"}
 F --> U: Mostrar producto no encontrado
end
@enduml
```

</details>

<br>

---

<br>

### 3 — Crear pedido pendiente

Este diagrama muestra únicamente la creación de un pedido pendiente. El cliente envía el detalle del pedido y API Pedidos lo registra en MySQL junto con sus artículos. En esta operación todavía no se descuenta stock ni se declara la compra confirmada. El ejemplo está acotado a un producto y contempla la reutilización de un pedido existente para evitar duplicaciones.

**Contrato propuesto:** POST /api/pedidos: 201 si crea un pedido pendiente; 200 si reutiliza uno existente con la misma clave y solicitud; 400 si el detalle no es válido y 401 si falla la autenticación.

[Ver este diagrama en tamaño completo](diagramas/03-pedido.png)

![3 — Crear pedido pendiente](diagramas/03-pedido.png)

<details>
<summary>Ver código PlantUML de este diagrama</summary>

```plantuml
@startuml
skinparam defaultFontName SansSerif
skinparam shadowing false
skinparam maxMessageSize 300
autonumber
title Pedidos — Crear pedido pendiente (contrato propuesto)
actor Cliente as U
participant "Frontend" as F
participant "Gateway" as G
participant "API Pedidos" as A
database "MySQL" as D
note over F,D
Ejemplo acotado a un producto. Crear el pedido todavía no confirma la compra.
La cantidad pertenece al detalle del pedido; no es un endpoint.
end note
U -> F: Solicitar pedido
F -> G: POST /api/pedidos\nAuthorization: Bearer jwt\nIdempotency-Key: intentoId\n{items: [{productoId, cantidad}]}
G -> G: Verificar JWT y estructura del body
alt Token ausente, inválido o vencido
 G --> F: 401 Unauthorized\n{message: "Autenticación requerida"}
 F --> U: Solicitar inicio de sesión
else Solicitud incompleta o cantidad no positiva
 G --> F: 400 Bad Request\n{message: "Detalle de pedido inválido"}
 F --> U: Mostrar error
else Solicitud válida
 G -> A: POST /api/pedidos\nAuthorization: Bearer jwt\nIdempotency-Key: intentoId\n{items: [{productoId, cantidad}]}
 A -> A: Validar JWT; obtener usuarioId del token
 A -> D: SELECT id, estado FROM pedidos\nWHERE intento_id = :intentoId AND usuario_id = :usuarioId;
 D --> A: Pedido existente o ninguna fila
 alt Pedido existente y solicitud coincidente
  A --> G: 200 OK\n{pedidoId, estado}
  G --> F: 200 OK\n{pedidoId, estado}
  F --> U: Mostrar estado del pedido existente
 else Nuevo intento
 A -> D: START TRANSACTION;\nINSERT INTO pedidos (usuario_id, intento_id, estado)\nVALUES (:usuarioId, :intentoId, 'pendiente');
 D --> A: pedidoId generado
 A -> D: INSERT INTO detalles_pedido (pedido_id, producto_id, cantidad)\nVALUES (:pedidoId, :productoId, :cantidad);\nCOMMIT;
 D --> A: Transacción confirmada
 A --> G: 201 Created\n{pedidoId, estado: "pendiente"}
 G --> F: 201 Created\n{pedidoId, estado: "pendiente"}
 F --> U: Mostrar pedido pendiente de confirmación
 end
end
note over A,D
Los dos INSERT son una transacción local: si falla, no se guarda el pedido parcial.
Todavía no se modificó stock. Precio y total quedan pendientes.
end note
@enduml
```

</details>

<br>

---

<br>

### 4 — Confirmar o rechazar el pedido

Este diagrama muestra únicamente la confirmación de un pedido previamente creado. Gateway coordina el movimiento de stock en API Catálogo y la actualización del estado en API Pedidos. Si el producto no está disponible, se propone rechazar el pedido. Si una respuesta se pierde o la confirmación falla después de modificar stock, la operación requiere conciliación; no se declara una compra exitosa.

**Contrato propuesto:** POST /api/movimientos-stock: 201 para el nuevo movimiento, 200 para un resultado existente coincidente y 409 cuando no se puede atender el pedido. PATCH /api/pedidos/{pedidoId}: 200 cuando se actualiza o se recupera el estado confirmado.

[Ver este diagrama en tamaño completo](diagramas/04-confirmacion.png)

![4 — Confirmar o rechazar el pedido](diagramas/04-confirmacion.png)

<details>
<summary>Ver código PlantUML de este diagrama</summary>

```plantuml
@startuml
skinparam defaultFontName SansSerif
skinparam shadowing false
skinparam maxMessageSize 300
autonumber
title Pedidos — Confirmar pedido pendiente (contrato propuesto)
participant "Gateway" as G
participant "API Catálogo" as C
database "PostgreSQL" as DC
participant "API Pedidos" as P
database "MySQL" as DP
note over G,DP
Operación interna posterior a crear el pedido pendiente.
Caso: un producto, sin cupón ni pago. El frontend aún no vacía el carrito.
Gateway ya tiene pedidoId, usuarioId, productoId y cantidad del pedido.
end note
G -> C: POST /api/movimientos-stock\nAuthorization: Bearer jwt\nIdempotency-Key: pedidoId\n{pedidoId, productoId, cantidad, tipo: "venta"}
C -> C: Validar identidad y permisos de la solicitud
C -> DC: SELECT producto_id, cantidad, precio_unitario\nFROM movimientos_stock WHERE pedido_id = :pedidoId;
DC --> C: Movimiento existente o ninguna fila
alt Movimiento ya registrado y datos coincidentes
 C --> G: 200 OK\n{pedidoId, productoId, cantidad, precioUnitario}
else Primera solicitud
 C -> DC: BEGIN;\nUPDATE productos SET stock = stock - :cantidad\nWHERE id = :productoId AND activo = TRUE\nAND stock >= :cantidad\nRETURNING id, precio;
 DC --> C: Una fila actualizada o ninguna
 alt Sin fila actualizada
  C -> DC: COMMIT;
  DC --> C: Sin cambios
  C --> G: 409 Conflict\n{message: "Producto no disponible para el pedido"}
 else Stock actualizado
  C -> DC: INSERT INTO movimientos_stock\n(pedido_id, producto_id, cantidad, precio_unitario, tipo)\nVALUES (:pedidoId, :productoId, :cantidad, :precio, 'venta');\nCOMMIT;
  DC --> C: Movimiento y stock confirmados en la misma transacción
  C --> G: 201 Created\n{pedidoId, productoId, cantidad, precioUnitario}
 end
end
alt Movimiento confirmado o reutilizado
 G -> P: PATCH /api/pedidos/{pedidoId}\nAuthorization: Bearer jwt\n{estado: "confirmado", precioUnitario}
 P -> P: Validar llamada interna autorizada\nCalcular total = cantidad persistida * precioUnitario
 P -> DP: UPDATE pedidos p JOIN detalles_pedido d ON d.pedido_id = p.id\nSET p.estado = 'confirmado', p.total = d.cantidad * :precio,\nd.precio_unitario = :precio\nWHERE p.id = :pedidoId AND p.estado = 'pendiente';\nSELECT id, estado, total FROM pedidos WHERE id = :pedidoId;
 DP --> P: Resultado de actualización y estado persistido
 P --> G: 200 OK\n{pedidoId, estado: "confirmado", total}
else Stock no disponible
 G -> P: PATCH /api/pedidos/{pedidoId}\nAuthorization: Bearer jwt\n{estado: "rechazado"}
 P -> DP: UPDATE pedidos SET estado = 'rechazado'\nWHERE id = :pedidoId AND estado = 'pendiente';\nSELECT id, estado FROM pedidos WHERE id = :pedidoId;
 DP --> P: Resultado de actualización y estado persistido
 P --> G: 200 OK\n{pedidoId, estado: "rechazado"}
end
note over G,DP
Si una respuesta se pierde o Pedidos no confirma, no se declara éxito.
El pedido sigue pendiente de conciliación. Reintentar usa el mismo pedidoId.
No se representa una reversión conjunta de PostgreSQL y MySQL.
end note
@enduml
```

</details>

<br>

---

<br>

### Contratos propuestos

| Origen → destino              | Solicitud                       | Resultado principal                                          |
| ----------------------------- | ------------------------------- | ------------------------------------------------------------ |
| Frontend → Gateway → Usuarios | `POST /api/sesiones`            | `201 Created`, token y usuario autenticado.                  |
| Frontend → Gateway → Catálogo | `GET /api/productos/{id}`       | `200 OK`, detalle del producto.                              |
| Frontend → Gateway → Pedidos  | `POST /api/pedidos`             | `201 Created`, pedido pendiente.                             |
| Gateway → Catálogo            | `POST /api/movimientos-stock`   | `201 Created`, movimiento registrado con precio de Catálogo. |
| Gateway → Pedidos             | `PATCH /api/pedidos/{pedidoId}` | `200 OK`, estado persistido del pedido.                      |

Las rutas identifican recursos mediante sustantivos. Crear un movimiento de stock es distinto de usar una ruta con un verbo como `/descontar`. No se incluye una ruta de pagos porque estos diagramas no registran pagos.

### Alcance y consistencia

- La creación del pedido y su detalle se realiza dentro de una transacción local en MySQL. Si falla, no debe persistir un pedido parcial. En esta etapa no se modificó el stock.
- Catálogo registra el movimiento y modifica el stock dentro de una misma transacción local en PostgreSQL.
- La cantidad es un campo del detalle, no una ruta. Los ejemplos de pedidos incluyen un solo producto; el caso de varios productos requiere una transacción de inventario para el conjunto completo.
- La idempotencia requiere restricciones únicas, comparación del contenido y manejo de solicitudes simultáneas. Un SELECT previo por sí solo no evita duplicaciones. La misma clave con contenido diferente debe rechazarse con `409 Conflict`.
- Gateway debe consultar el estado durable del pedido antes de continuar o reintentar la confirmación. Un pedido confirmado o rechazado no vuelve a procesarse como un pedido nuevo.
- Si stock quedó aplicado y la confirmación del pedido no se completa, el pedido permanece pendiente de conciliación. El reintento debe recuperar el movimiento existente y continuar, sin volver a descontar unidades.
- Esa conciliación debe implementarse de manera durable en los servicios, incluida la recuperación después de reinicios. No queda resuelta únicamente por dibujar los mensajes HTTP.
- Los ejemplos muestran validaciones y errores de negocio. Los fallos de persistencia y red necesitan tratamiento adicional y nunca deben producir una confirmación falsa.
- Eliminar la compensación anterior del diagrama no elimina las transacciones locales: una transacción local fallida debe cancelarse. No existe una única transacción que abarque ambas bases.
- Pedidos obtiene la identidad del JWT. Solo llamadas internas autorizadas pueden establecer precios y cambiar el estado a confirmado o rechazado. El precio procede de Catálogo y el total se calcula en el servidor.
- El frontend muestra la compra confirmada y vacía el carrito únicamente al conocer el estado confirmado. La respuesta `201` de un pedido pendiente no alcanza. La limpieza de un carrito persistido en Usuarios requiere coordinación adicional.
- Los pagos, cupones y comprobantes tendrán sus propios diagramas cuando se definan sus contratos. No se afirma que hayan sido eliminados del alcance del proyecto.

### Correspondencia con la arquitectura

Usuarios utiliza MongoDB Atlas; Catálogo utiliza PostgreSQL; Pedidos utiliza MySQL. El frontend consume únicamente Gateway y ninguna API consulta directamente la base de otra.

El sistema comprende cuatro procesos backend contando el Gateway: Gateway, Usuarios, Catálogo y Pedidos. Son tres APIs de dominio y tres bases propias. Los alojamientos previstos se mantienen en la sección de despliegue.
