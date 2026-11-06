# Plan de fases — ERP ligero multiempresa

Documento maestro del proyecto. Está diseñado para que puedas ejecutar `/clear` entre fases:
cada fase es autocontenida e incluye el contexto, los entregables, el criterio de aceptación y el
prompt exacto con el que arrancar una sesión nueva.

**Regla de uso:** al empezar una sesión nueva, lee este archivo y `CLAUDE.md`, actualiza la tabla de
estado y trabaja solo en la fase activa.

---

## 1. Estado actual

| Fase | Nombre | Estado | Notas |
|---|---|---|---|
| 0 | Fundación del repositorio e infraestructura local | ⬜ Pendiente | |
| 1 | Diseño de base de datos y esquema Prisma | ⬜ Pendiente | |
| 2 | Autenticación y multi-tenancy (aislamiento real) | ⬜ Pendiente | |
| 3 | RBAC: roles y permisos | ⬜ Pendiente | |
| 4 | Catálogo: productos, categorías, almacenes | ⬜ Pendiente | |
| 5 | Inventario: libro de movimientos y stock | ⬜ Pendiente | |
| 6 | Concurrencia: imposible sobrevender (fase estrella) | ⬜ Pendiente | |
| 7 | Compras y proveedores | ⬜ Pendiente | |
| 8 | Ventas | ⬜ Pendiente | |
| 9 | Reportes y caché con Redis | ⬜ Pendiente | |
| 10 | Colas BullMQ y alertas de stock bajo | ⬜ Pendiente | |
| 11 | Auditoría de operaciones | ⬜ Pendiente | |
| 12 | Frontend Next.js | ⬜ Pendiente | |
| 13 | Panel administrativo | ⬜ Pendiente | |
| 14 | Endurecimiento, observabilidad y carga | ⬜ Pendiente | |
| 15 | Despliegue en AWS con Docker | ⬜ Pendiente | |

Estados: ⬜ Pendiente · 🟨 En curso · ✅ Hecho · ⛔ Bloqueada

---

## 2. Objetivo del proyecto

SaaS de gestión empresarial (ERP ligero) donde varias empresas se registran y administran su propio
inventario, compras y ventas de forma aislada.

Competencias que el proyecto debe demostrar de forma explícita y verificable:

1. **Diseño de base de datos**: normalización, claves compuestas por tenant, índices intencionados.
2. **Transacciones**: operaciones de negocio atómicas de extremo a extremo.
3. **Concurrencia**: dos usuarios no pueden vender las últimas unidades del mismo producto.
4. **Multi-tenancy**: aislamiento garantizado por la base de datos, no solo por el código.
5. **Seguridad**: autenticación, RBAC, validación de entrada, auditoría.
6. **Arquitectura modular**: módulos de dominio con fronteras claras.
7. **Escalabilidad**: caché, colas, trabajo asíncrono, servicios sin estado.

---

## 3. Decisiones de arquitectura fijadas

Estas decisiones ya están tomadas. No se renegocian en cada fase; si una cambia, se anota aquí con
la fecha y el motivo.

### 3.1 Estructura del repositorio

Monorepo con pnpm workspaces:

```
gestion/
├── apps/
│   ├── api/        # NestJS
│   └── web/        # Next.js (App Router)
├── packages/
│   ├── database/   # Prisma schema, migraciones, cliente extendido
│   ├── contracts/  # DTOs y tipos compartidos API <-> web (zod)
│   └── config/     # tsconfig y eslint compartidos
├── docker/
├── CLAUDE.md
└── PHASES.md
```

### 3.2 Estrategia de multi-tenancy

**Base de datos compartida, esquema compartido, discriminador `companyId` + Row Level Security de
PostgreSQL.**

Dos capas de defensa:

- **Capa de aplicación**: una extensión del cliente Prisma inyecta `companyId` en todo `where`,
  `create` y `update`. Impide el error humano.
- **Capa de base de datos**: políticas RLS sobre cada tabla con tenant. Cada transacción ejecuta
  `SET LOCAL app.company_id = '<uuid>'`, y la política compara `company_id` con
  `current_setting('app.company_id')`. Impide la fuga incluso si la capa de aplicación falla.

La API se conecta con un rol de PostgreSQL **sin** `BYPASSRLS`. Las migraciones usan un rol distinto.

Motivo de descartar esquema por tenant: miles de esquemas rompen las migraciones y el pool de
conexiones. El objetivo es demostrar aislamiento correcto, no aislamiento físico.

### 3.3 Modelo de inventario

Libro de movimientos **append-only** (`InventoryMovement`) como fuente de verdad, más una tabla de
saldos (`StockLevel`) que es una caché transaccional actualizada en la misma transacción que el
movimiento.

- `InventoryMovement`: sin `UPDATE` ni `DELETE`. Una corrección es un movimiento inverso.
- `StockLevel`: una fila por `(companyId, productId, warehouseId)` con `quantity` y `reserved`.
- Invariante en base de datos: `CHECK (quantity >= 0)` y `CHECK (reserved >= 0)`.

Esto da trazabilidad completa y consultas de saldo en O(1).

### 3.4 Control de concurrencia

Estrategia primaria: **decremento condicional atómico**.

```sql
UPDATE stock_levels
   SET quantity = quantity - $qty
 WHERE company_id = $company
   AND product_id = $product
   AND warehouse_id = $warehouse
   AND quantity - reserved >= $qty
RETURNING quantity;
```

Si devuelve 0 filas, no había stock suficiente y se lanza `InsufficientStockException`. El `UPDATE`
toma un lock de fila implícito, de modo que dos transacciones simultáneas se serializan y solo una
puede llevarse la última unidad. Sin lectura previa, por tanto sin ventana de carrera.

Reglas complementarias:

- **Ventas multilínea**: ordenar siempre las líneas por `(productId, warehouseId)` antes de aplicar
  los decrementos. Un orden de locks determinista elimina los deadlocks.
- **Nivel de aislamiento**: `READ COMMITTED` por defecto. En los flujos que leen y después escriben
  (cierres, ajustes) se usa `REPEATABLE READ` con reintento ante `40001`.
- **Reservas**: el carrito de venta reserva stock con TTL. La reserva incrementa `reserved` y guarda
  una clave en Redis; un job de BullMQ libera las expiradas.
- **Timeout de lock**: `SET LOCAL lock_timeout = '3s'` para fallar rápido en vez de acumular esperas.
- **Idempotencia**: cada venta acepta un `Idempotency-Key` persistido con índice único, para que un
  reintento de red no genere dos ventas.

### 3.5 Convenciones transversales

- Todos los identificadores son UUID v7 (ordenables por tiempo, buenos para índices).
- Todo importe monetario es `Decimal(18,4)`. Nunca `Float`.
- Toda cantidad de inventario es `Decimal(18,4)` para admitir unidades fraccionadas.
- Toda tabla con tenant lleva índice compuesto que empieza por `company_id`.
- Todo endpoint valida su entrada con zod desde `packages/contracts`.
- Todo error de dominio es una excepción tipada, nunca un `throw new Error`.
- Las fechas se guardan en UTC con `timestamptz`.

---

## 4. Fases

Cada fase indica: objetivo, entregables, criterio de aceptación, comando de verificación y prompt de
arranque.

---

### Fase 0 — Fundación del repositorio e infraestructura local

**Objetivo.** Dejar el monorepo arrancable con un solo comando y la infraestructura local en Docker.

**Entregables.**
- pnpm workspace con `apps/api`, `apps/web`, `packages/database`, `packages/contracts`, `packages/config`.
- NestJS inicializado con configuración estricta de TypeScript y validación de variables de entorno.
- Next.js con App Router y TypeScript estricto.
- `docker-compose.yml` con PostgreSQL 16 y Redis 7, volúmenes nombrados y healthchecks.
- `.env.example` completo y `README.md` con instrucciones de arranque.
- ESLint y Prettier compartidos; hook de pre-commit con lint-staged.
- Jest configurado en `apps/api` con un test trivial que pase.

**Criterio de aceptación.**
- `docker compose up -d` deja PostgreSQL y Redis saludables.
- `pnpm dev` levanta API y web sin errores.
- `pnpm test` pasa.
- `pnpm typecheck` pasa en todos los paquetes.

**Verificación.** `docker compose ps && pnpm -r typecheck && pnpm -r test`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 0 completa. No empieces la Fase 1.

---

### Fase 1 — Diseño de base de datos y esquema Prisma

**Objetivo.** Modelar todo el dominio de una vez, con integridad declarada en la base de datos.

**Entidades.**

- `Company` — tenant. Incluye `slug` único, plan y estado.
- `User` — global, identificado por correo. Un usuario puede pertenecer a varias empresas.
- `Membership` — une `User` y `Company` con un `Role`. Único por `(userId, companyId)`.
- `Role` y `Permission` — ver Fase 3.
- `Category` — jerárquica, con `parentId` autorreferenciado.
- `Product` — `sku` único por empresa, precio de coste, precio de venta, `minStock`, unidad de medida.
- `Warehouse` — almacén, con `isDefault` y dirección.
- `StockLevel` — saldo por `(companyId, productId, warehouseId)`. Único. `quantity`, `reserved`.
- `InventoryMovement` — append-only. Tipo (`IN`, `OUT`, `ADJUSTMENT`, `TRANSFER_IN`, `TRANSFER_OUT`),
  cantidad, `quantityAfter`, referencia polimórfica al documento de origen, `userId`, `createdAt`.
- `Supplier` — proveedor, con datos fiscales y de contacto.
- `Customer` — cliente.
- `Purchase` y `PurchaseItem` — compra con estado (`DRAFT`, `CONFIRMED`, `RECEIVED`, `CANCELLED`).
- `Sale` y `SaleItem` — venta con estado (`DRAFT`, `CONFIRMED`, `CANCELLED`) y `idempotencyKey`.
- `StockReservation` — reserva temporal con `expiresAt`.
- `AuditLog` — ver Fase 11.

**Entregables.**
- `packages/database/prisma/schema.prisma` con el modelo completo.
- Migración inicial aplicada.
- Restricciones que Prisma no expresa, añadidas por migración SQL manual:
  - `CHECK (quantity >= 0)` y `CHECK (reserved >= 0)` en `stock_levels`.
  - `CHECK (reserved <= quantity)` en `stock_levels`.
  - `CHECK (quantity > 0)` en las líneas de compra y venta.
  - Índice único parcial para un solo almacén por defecto por empresa.
- Índices compuestos encabezados por `company_id` en toda tabla con tenant.
- Seed con dos empresas, usuarios, productos y stock, para poder probar el aislamiento.
- Diagrama entidad-relación en `docs/erd.md` en Mermaid.

**Criterio de aceptación.**
- `prisma migrate dev` corre desde cero sin errores.
- El seed crea datos en dos empresas distintas.
- Un `INSERT` manual que viole un `CHECK` es rechazado por PostgreSQL.

**Verificación.** `pnpm --filter database migrate:reset && pnpm --filter database seed`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 1: esquema Prisma completo, migraciones,
> restricciones SQL manuales, índices, seed y diagrama entidad-relación. No escribas código de API.

---

### Fase 2 — Autenticación y multi-tenancy

**Objetivo.** Un usuario autenticado opera siempre dentro de una empresa concreta, y el aislamiento
lo garantiza PostgreSQL.

**Entregables.**
- Registro de empresa: crea `Company`, `User` propietario, `Membership` con rol `OWNER` y almacén por
  defecto, todo en una transacción.
- Login con argon2id. Access token JWT de 15 minutos y refresh token rotativo persistido y hasheado.
- El access token incluye `sub`, `companyId`, `role` y `jti`.
- Cambio de empresa activa para usuarios con varias pertenencias; emite un token nuevo.
- `AsyncLocalStorage` con el contexto de petición (`userId`, `companyId`, `requestId`).
- Extensión del cliente Prisma que inyecta `companyId` en todas las operaciones de modelos con tenant
  y lanza error si el contexto está vacío.
- Migración que habilita RLS: `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` y `FORCE ROW LEVEL SECURITY`
  en cada tabla con tenant, más la política por `current_setting('app.company_id', true)`.
- Envoltorio de transacción que ejecuta `SET LOCAL app.company_id` antes de cualquier consulta.
- Rol de PostgreSQL de la aplicación sin `BYPASSRLS`; rol de migración separado.
- Rate limiting en login y registro.

**Criterio de aceptación.**
- Test de integración: un token de la empresa A no puede leer ni escribir datos de la empresa B,
  ni siquiera pasando un identificador de B explícitamente.
- Test que demuestra que una consulta cruda sin `app.company_id` no devuelve filas.
- Test de rotación de refresh token: reutilizar uno ya usado invalida la familia completa.

**Verificación.** `pnpm --filter api test:e2e -- tenancy`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 2: autenticación, contexto de petición, extensión de
> Prisma con `companyId`, políticas RLS y los tests de aislamiento entre empresas. El test de fuga
> entre tenants debe fallar primero y pasar después.

---

### Fase 3 — RBAC: roles y permisos

**Objetivo.** Permisos granulares por empresa, verificables y testeados.

**Entregables.**
- Permisos con formato `recurso:acción`, por ejemplo `product:create`, `sale:confirm`,
  `inventory:adjust`, `report:read`, `audit:read`, `member:invite`.
- Roles del sistema predefinidos: `OWNER`, `ADMIN`, `WAREHOUSE`, `SALES`, `VIEWER`.
- Roles personalizados por empresa, con conjunto propio de permisos.
- Decorador `@RequirePermissions(...)` y guard que resuelve el rol desde el contexto.
- Caché de permisos del rol en Redis, invalidada al modificar el rol.
- Invitaciones por correo con token de un solo uso y caducidad.
- Protección del último `OWNER`: no se puede degradar ni eliminar.

**Criterio de aceptación.**
- Matriz de tests: cada rol contra cada endpoint sensible, esperando 200 o 403.
- Un rol personalizado de la empresa A no es asignable en la empresa B.

**Verificación.** `pnpm --filter api test -- rbac`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 3: permisos, roles del sistema, roles
> personalizados, guard, caché en Redis, invitaciones y la matriz de tests rol por endpoint.

---

### Fase 4 — Catálogo: productos, categorías, almacenes

**Objetivo.** CRUD completo del catálogo, con validación y paginación serias.

**Entregables.**
- Módulos `products`, `categories`, `warehouses` con la estructura de capas acordada.
- `sku` único por empresa; conflicto devuelve 409 con mensaje claro.
- Categorías jerárquicas con detección de ciclos al mover una rama.
- Borrado lógico de productos con referencias en movimientos; borrado físico solo si no hay historial.
- Paginación por cursor, filtros (categoría, estado, rango de precio) y búsqueda por texto.
- Búsqueda por `nombre` y `sku` con índice GIN y `pg_trgm`.
- Importación masiva por CSV procesada en cola, con informe de errores por fila.

**Criterio de aceptación.**
- Tests unitarios de los servicios y tests e2e de los endpoints.
- Mover una categoría dentro de su propio subárbol devuelve 400.
- La paginación es estable al insertar filas nuevas durante el recorrido.

**Verificación.** `pnpm --filter api test -- catalog`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 4: módulos de productos, categorías y almacenes con
> validación, paginación por cursor, búsqueda con `pg_trgm`, importación CSV y tests.

---

### Fase 5 — Inventario: libro de movimientos y stock

**Objetivo.** Un único camino de código por el que puede cambiar el stock.

**Entregables.**
- `InventoryService` como **única** puerta de escritura sobre `stock_levels`.
- Operaciones: entrada, salida, ajuste con motivo obligatorio y traspaso entre almacenes.
- Todo cambio escribe movimiento y saldo en la misma transacción.
- El traspaso genera dos movimientos ligados por un identificador común.
- Consulta de saldo por producto, por almacén y agregado por empresa.
- Endpoint de reconstrucción que recalcula `stock_levels` desde los movimientos y compara, para
  demostrar que la caché transaccional nunca se desvía.
- Guía en `docs/inventory.md` sobre cómo extender con nuevos tipos de movimiento.

**Criterio de aceptación.**
- Test que fuerza un fallo a mitad de un traspaso y verifica que no queda stock perdido.
- La reconstrucción desde el libro coincide con `stock_levels` tras 500 movimientos aleatorios.
- Intentar escribir `stock_levels` fuera de `InventoryService` falla en revisión de código y lo
  detecta un test de arquitectura.

**Verificación.** `pnpm --filter api test -- inventory`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 5: `InventoryService` como única puerta de escritura
> de stock, movimientos append-only, traspasos, reconstrucción desde el libro y tests, incluido el de
> fallo a mitad de transacción.

---

### Fase 6 — Concurrencia: imposible sobrevender

**Objetivo.** Demostrar con pruebas que dos usuarios no pueden vender a la vez las últimas unidades.
Esta es la fase que da valor diferencial al proyecto; merece documentación propia.

**Entregables.**
- `StockAllocator` con el decremento condicional atómico descrito en 3.4.
- Orden determinista de locks en ventas multilínea, con test de deadlock.
- `SET LOCAL lock_timeout` y reintento con retroceso exponencial ante `40001` y `40P01`.
- Reservas con TTL: `reserve`, `confirm`, `release`; liberación de expiradas por job.
- Idempotencia de la confirmación de venta mediante `Idempotency-Key` con índice único.
- Suite de concurrencia:
  - 50 peticiones en paralelo contra 1 unidad disponible: exactamente 1 éxito y 49 con 409.
  - 100 peticiones contra 10 unidades: exactamente 10 éxitos, stock final 0, nunca negativo.
  - Venta multilínea cruzada A→B y B→A en paralelo: ningún deadlock sin resolver.
  - Comparación medida contra una implementación naíf de leer-luego-escribir, que debe sobrevender,
    como prueba de que el test detecta el fallo.
- `docs/concurrency.md`: el problema, las cuatro estrategias consideradas (decremento condicional,
  `SELECT FOR UPDATE`, bloqueo optimista por versión, locks de aviso), por qué se eligió la primera,
  y los resultados medidos.

**Criterio de aceptación.**
- La suite de concurrencia pasa 20 ejecuciones seguidas sin fallos intermitentes.
- `stock_levels.quantity` nunca es negativo en ninguna ejecución.
- La suma de los movimientos siempre cuadra con el saldo final.

**Verificación.** `pnpm --filter api test:concurrency` repetido con `--runs 20`

**Prompt de arranque.**
> Lee `PHASES.md`, `CLAUDE.md` y la sección 3.4. Ejecuta la Fase 6: `StockAllocator` con decremento
> condicional atómico, orden determinista de locks, reintentos, reservas con TTL, idempotencia y la
> suite completa de tests de concurrencia, incluido el caso naíf que debe sobrevender. Escribe
> `docs/concurrency.md` con las alternativas y los números medidos.

---

### Fase 7 — Compras y proveedores

**Objetivo.** Flujo de compra que incrementa stock de forma atómica.

**Entregables.**
- Módulo `suppliers` con CRUD y validación de identificador fiscal.
- Compra con estados `DRAFT` → `CONFIRMED` → `RECEIVED`, y `CANCELLED` desde `DRAFT` o `CONFIRMED`.
- La recepción genera movimientos `IN` por línea, dentro de una sola transacción.
- Recepción parcial: cantidad recibida por línea, con pendiente calculado.
- Recálculo del coste medio ponderado del producto en cada recepción.
- Totales de la compra calculados en servidor: subtotal, impuesto, descuento, total.
- Anulación de una compra recibida mediante movimientos inversos, nunca borrando historial.

**Criterio de aceptación.**
- Tests de la máquina de estados: toda transición inválida devuelve 409.
- Confirmar la misma recepción dos veces no duplica stock.
- El coste medio ponderado se verifica con un caso calculado a mano.

**Verificación.** `pnpm --filter api test -- purchases`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 7: proveedores, compras con máquina de estados,
> recepción parcial, coste medio ponderado y tests. Usa `InventoryService` para todo cambio de stock.

---

### Fase 8 — Ventas

**Objetivo.** Flujo de venta que consume stock usando el asignador de la Fase 6.

**Entregables.**
- Módulo `customers` con CRUD.
- Venta con estados `DRAFT` → `CONFIRMED` → `CANCELLED`, y devoluciones parciales.
- La confirmación pasa por `StockAllocator`; nunca lee y después escribe.
- Validación de que todas las líneas pertenecen a la empresa y al almacén indicado.
- Precios tomados del servidor; el cliente no puede enviar el precio.
- Devolución que genera movimientos `IN` y nota de crédito.
- Cálculo de margen por venta usando el coste medio ponderado del momento.

**Criterio de aceptación.**
- Una venta que falla en la línea 3 de 5 no deja stock consumido en las líneas 1 y 2.
- Reenviar la misma petición con el mismo `Idempotency-Key` devuelve la venta original, no una nueva.
- Tests e2e del flujo completo: compra, recepción, venta, devolución, con cuadre de stock.

**Verificación.** `pnpm --filter api test -- sales`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 8: clientes, ventas sobre `StockAllocator`,
> devoluciones, margen y tests, incluido el de fallo parcial multilínea.

---

### Fase 9 — Reportes y caché con Redis

**Objetivo.** Consultas analíticas rápidas sin castigar la base de datos transaccional.

**Entregables.**
- Reportes: ingresos por periodo, productos más vendidos, rotación de inventario, valoración de
  existencias, movimientos por almacén, margen por categoría.
- Consultas SQL agregadas escritas a mano con `$queryRaw` tipado, no composición de ORM.
- Caché en Redis con clave que incluye empresa, reporte y parámetros; TTL por tipo de reporte.
- Invalidación por etiquetas al cambiar datos relevantes.
- Exportación a CSV y XLSX procesada en cola, con descarga por enlace temporal.
- `EXPLAIN ANALYZE` de cada reporte guardado en `docs/query-plans.md`, con los índices que lo
  sostienen.

**Criterio de aceptación.**
- Ningún reporte supera 200 ms con 100.000 movimientos sembrados.
- Un acierto de caché no toca PostgreSQL, verificado con contador de consultas.
- La invalidación se comprueba con test: modificar una venta cambia el reporte de ingresos.

**Verificación.** `pnpm --filter api test -- reports`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 9: reportes con SQL agregado tipado, caché en Redis
> con invalidación por etiquetas, exportaciones en cola y `docs/query-plans.md` con los planes
> medidos sobre 100.000 movimientos.

---

### Fase 10 — Colas BullMQ y alertas de stock bajo

**Objetivo.** Trabajo asíncrono fiable y la alerta de stock bajo de extremo a extremo.

**Entregables.**
- BullMQ con colas separadas: `notifications`, `reports`, `imports`, `maintenance`.
- Worker como proceso independiente, escalable por separado de la API.
- Reintentos con retroceso exponencial, cola de mensajes fallidos y job de limpieza.
- Alerta de stock bajo: al cruzar `minStock` a la baja se encola la notificación. La comprobación
  ocurre dentro de la transacción de inventario, pero el envío se hace fuera, con patrón outbox para
  no perder eventos ante un rollback.
- Agrupación de alertas para no inundar: una notificación por producto por ventana de tiempo.
- Job programado de resumen diario por empresa.
- Job de liberación de reservas caducadas.
- Panel de Bull Board protegido por permiso de administración.

**Criterio de aceptación.**
- Test: una venta que baja el stock por debajo del mínimo encola exactamente una alerta.
- Test: si la transacción revierte, no se encola ninguna alerta.
- Un job que falla tres veces acaba en la cola de fallos con su traza.

**Verificación.** `pnpm --filter api test -- queues`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 10: colas BullMQ, worker independiente, outbox
> transaccional para alertas de stock bajo, agrupación, jobs programados y tests, incluido el caso de
> rollback sin alerta.

---

### Fase 11 — Auditoría de operaciones

**Objetivo.** Rastro inmutable de quién hizo qué, cuándo y desde dónde.

**Entregables.**
- `AuditLog`: empresa, actor, acción, entidad, identificador de entidad, diferencia antes/después,
  dirección IP, agente de usuario, `requestId`, marca de tiempo.
- Captura automática por interceptor para las mutaciones, más registro explícito para los eventos de
  dominio relevantes (confirmar venta, ajustar inventario, cambiar rol).
- Inmutabilidad: permisos de PostgreSQL que impiden `UPDATE` y `DELETE` al rol de la aplicación.
- Redacción de campos sensibles antes de persistir.
- Consulta de auditoría con filtros por actor, acción, entidad y rango de fechas, protegida por
  `audit:read`.
- Retención y archivado por particionado mensual de la tabla.

**Criterio de aceptación.**
- Toda mutación de las fases 4 a 8 produce exactamente una entrada de auditoría.
- Un `UPDATE` sobre `audit_log` con el rol de la aplicación es rechazado por la base de datos.
- Las contraseñas y los tokens nunca aparecen en el rastro, verificado por test.

**Verificación.** `pnpm --filter api test -- audit`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 11: tabla de auditoría particionada, interceptor,
> inmutabilidad por permisos de PostgreSQL, redacción de datos sensibles, consulta protegida y tests.

---

### Fase 12 — Frontend Next.js

**Objetivo.** Interfaz propia y deliberada, no una plantilla genérica.

**Antes de maquetar:** aplicar la skill `frontend-design`. Sin degradados morados, sin tarjetas
flotantes por defecto, sin la estética de panel genérico. Decidir una dirección visual, un sistema
tipográfico y una escala de espaciado, y documentarlos en `docs/design.md`.

**Entregables.**
- App Router con rutas por empresa, Server Components por defecto y Client Components solo donde haga
  falta interacción.
- Autenticación con cookies `httpOnly`; refresco de token transparente.
- Selector de empresa activa en la barra superior.
- Pantallas: catálogo, almacenes, inventario con movimientos, compras, ventas, reportes, ajustes,
  miembros y roles.
- Formularios con React Hook Form y los esquemas zod de `packages/contracts`, compartidos con la API.
- Estado de servidor con TanStack Query; actualizaciones optimistas donde el error sea recuperable.
- Manejo explícito del conflicto de stock: cuando la API devuelve 409 por stock insuficiente, la
  interfaz lo explica con el saldo real, no con un error genérico.
- Tablas con paginación por cursor, filtros persistidos en la URL y estados vacíos diseñados.
- Accesibilidad: foco visible, navegación por teclado, contraste comprobado, etiquetas en formularios.
- Tema claro y oscuro.

**Criterio de aceptación.**
- Flujo completo usable sin tocar la API a mano: registrar empresa, crear producto, comprar, vender.
- El caso de sobreventa muestra un mensaje correcto y el saldo actualizado.
- Lighthouse: accesibilidad 95 o más en las pantallas principales.

**Verificación.** `pnpm --filter web build && pnpm --filter web test`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Aplica la skill `frontend-design` antes de escribir componentes.
> Ejecuta la Fase 12: frontend Next.js completo con dirección visual propia documentada en
> `docs/design.md`, formularios con los esquemas compartidos y manejo explícito del conflicto 409.

---

### Fase 13 — Panel administrativo

**Objetivo.** Administración de la plataforma, separada de la administración de cada empresa.

**Entregables.**
- Rol de superadministrador fuera del modelo de pertenencias, con su propia ruta de autenticación.
- Listado de empresas con métricas de uso: usuarios, productos, movimientos, última actividad.
- Suspender y reactivar empresas; una empresa suspendida rechaza todo acceso.
- Vista de salud: estado de las colas, trabajos fallidos, latencia de la base de datos, aciertos de
  caché.
- Impersonación con registro obligatorio en auditoría y banner visible en la interfaz.
- Límites por plan: tope de productos, usuarios y almacenes, aplicados en el servicio.

**Criterio de aceptación.**
- Un superadministrador no puede leer datos de negocio sin impersonar, y la impersonación queda
  auditada.
- Superar el límite del plan devuelve 402 con el límite y el uso actual.

**Verificación.** `pnpm --filter api test -- admin`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 13: panel administrativo con superadministrador,
> suspensión de empresas, vista de salud, impersonación auditada y límites por plan.

---

### Fase 14 — Endurecimiento, observabilidad y carga

**Objetivo.** Dejar el sistema listo para producción y con los números medidos.

**Entregables.**
- Seguridad: Helmet, CORS cerrado por lista, rate limiting por ruta y por tenant, límite de tamaño de
  cuerpo, tiempo de espera por petición.
- Validación estricta: `whitelist` y `forbidNonWhitelisted` en todos los DTO.
- Repaso de inyección SQL en todo `$queryRaw`: solo parámetros, nunca interpolación.
- Secretos por variables de entorno validadas al arrancar; el proceso no arranca si falta una.
- Logs estructurados en JSON con `requestId` y `companyId`, sin datos personales.
- Trazas con OpenTelemetry; métricas Prometheus en `/metrics`.
- Healthchecks `/health/live` y `/health/ready` que comprueban PostgreSQL y Redis.
- Pool de conexiones dimensionado y documentado; PgBouncer en modo transacción.
- Pruebas de carga con k6: lectura de catálogo, confirmación de venta en contención y reportes.
  Resultados en `docs/load-tests.md` con percentiles 50, 95 y 99.
- Cobertura mínima del 80 % en los módulos de dominio.
- Revisión de seguridad con `/security-review`.

**Criterio de aceptación.**
- k6 sostiene el objetivo acordado de ventas por segundo sin sobreventa ni errores de servidor.
- Ningún secreto aparece en los logs, verificado por búsqueda.
- La revisión de seguridad no deja hallazgos altos abiertos.

**Verificación.** `pnpm test:cov && k6 run load/sales.js`

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 14: endurecimiento de seguridad, observabilidad con
> OpenTelemetry y Prometheus, healthchecks, PgBouncer, pruebas de carga con k6 documentadas y subida
> de cobertura al 80 % en dominio.

---

### Fase 15 — Despliegue en AWS con Docker

**Objetivo.** Despliegue reproducible y documentado.

**Entregables.**
- `Dockerfile` multi-etapa para API, worker y web, con usuario sin privilegios e imagen mínima.
- `docker-compose.prod.yml` para despliegue en un solo host.
- Infraestructura en AWS: ECS Fargate para API, worker y web; RDS PostgreSQL con copias automáticas;
  ElastiCache Redis; ALB con TLS; S3 para exportaciones; Secrets Manager; CloudWatch.
- Terraform en `infra/`, dividido en módulos, con estado remoto.
- Migraciones ejecutadas como tarea aparte antes del despliegue, nunca al arrancar la API.
- CI en GitHub Actions: lint, typecheck, test con PostgreSQL y Redis de servicio, build, push a ECR,
  despliegue.
- Estrategia de rollback documentada y probada una vez.
- `docs/runbook.md`: despliegue, rollback, restauración de copia de seguridad, rotación de secretos.

**Criterio de aceptación.**
- `terraform apply` crea el entorno desde cero.
- Un push a `main` despliega sin intervención manual.
- Restauración de copia de seguridad probada y cronometrada.

**Verificación.** `terraform plan` sin cambios pendientes tras aplicar, y pipeline verde.

**Prompt de arranque.**
> Lee `PHASES.md` y `CLAUDE.md`. Ejecuta la Fase 15: Dockerfiles multi-etapa, Terraform modular para
> ECS, RDS, ElastiCache, ALB y S3, pipeline de CI con despliegue, y `docs/runbook.md`.

---

## 5. Rutina de trabajo por fase

1. Sesión nueva: `/clear`.
2. Pegar el prompt de arranque de la fase activa.
3. Marcar la fase como 🟨 En curso en la tabla de estado.
4. Implementar en pasos pequeños, delegando la escritura de código a subagentes.
5. Ejecutar el comando de verificación de la fase.
6. Si falla, reasignar la corrección indicando el fallo exacto.
7. Cuando pase, marcar ✅ Hecho, anotar en la columna de notas lo que se desvió del plan, y hacer
   commit.
8. `/clear` y pasar a la siguiente fase.

**Qué anotar en la columna de notas.** Solo lo que una sesión futura no pueda deducir del código:
decisiones cambiadas, deuda aceptada a propósito, trampas encontradas.

---

## 6. Registro de cambios del plan

| Fecha | Cambio | Motivo |
|---|---|---|
| 2026-11-05 | Versión inicial del plan | — |
