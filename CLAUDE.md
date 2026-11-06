# gestion — ERP ligero multiempresa

SaaS de inventario, compras y ventas con varias empresas aisladas en la misma base de datos.

El plan de trabajo vive en `PHASES.md`. Léelo al empezar cualquier sesión, identifica la fase activa
en la tabla de estado y trabaja solo en esa fase.

## Stack

Next.js y TypeScript en el frontend. NestJS en el backend. PostgreSQL con Prisma. Redis para caché.
BullMQ para colas. Docker y AWS para infraestructura. Jest y Supertest para pruebas.

Monorepo pnpm: `apps/api`, `apps/web`, `packages/database`, `packages/contracts`, `packages/config`.

## Invariantes

Estas reglas no se rompen. Si una tarea parece exigirlo, para y pregunta.

1. **Aislamiento entre empresas.** Toda tabla de negocio lleva `companyId`. Nunca escribas una
   consulta que lo omita. El acceso pasa por el cliente Prisma extendido, que lo inyecta desde el
   contexto de petición. Las políticas RLS de PostgreSQL son la segunda barrera: la transacción debe
   ejecutar `SET LOCAL app.company_id` antes de cualquier consulta.

2. **Una sola puerta al stock.** `stock_levels` solo se modifica desde `InventoryService`. Ningún
   otro servicio, controlador ni script escribe en esa tabla.

3. **Nunca leer y después escribir para descontar stock.** Usa el decremento condicional atómico
   (`UPDATE ... WHERE quantity - reserved >= $qty`). Comprobar el saldo y luego restarlo en dos pasos
   es el error que este proyecto existe para demostrar que se evita.

4. **El libro de movimientos es inmutable.** `inventory_movements` y `audit_log` no admiten `UPDATE`
   ni `DELETE`. Una corrección es un movimiento inverso.

5. **Orden determinista de locks.** En operaciones multilínea, ordena siempre por
   `(productId, warehouseId)` antes de aplicar cambios. Evita deadlocks.

6. **Dinero y cantidades en `Decimal`.** Nunca `Float` ni `Number` para importes o cantidades de
   inventario.

7. **Precios del servidor.** El cliente nunca envía precios, totales ni impuestos. Se calculan en el
   backend.

8. **Validación en el borde.** Todo endpoint valida con los esquemas zod de `packages/contracts`, que
   el frontend reutiliza.

9. **Permisos explícitos.** Todo endpoint de mutación lleva `@RequirePermissions(...)`. Sin decorador
   no se mergea.

10. **SQL parametrizado.** En `$queryRaw` solo parámetros. Nunca interpolación de cadenas.

## Convenciones

- Identificadores UUID v7.
- Fechas en UTC con `timestamptz`.
- Índice compuesto encabezado por `company_id` en toda tabla con tenant.
- Errores de dominio como excepciones tipadas, nunca `throw new Error`.
- Un módulo NestJS por dominio: `controller`, `service`, `repository`, `dto`, `entities`.
- Nombres de tabla en `snake_case` plural; modelos Prisma en `PascalCase` singular con `@@map`.
- Mensajes de commit en formato Conventional Commits.

## Frontend

Antes de maquetar o modificar cualquier vista, aplica la skill `frontend-design`. La dirección visual
acordada se documenta en `docs/design.md` y se respeta. Nada de degradados morados ni estética de
panel genérico.

## Comandos

```bash
pnpm dev                        # API y web en paralelo
docker compose up -d            # PostgreSQL y Redis
pnpm --filter database migrate:dev
pnpm --filter database seed
pnpm -r typecheck
pnpm -r test
pnpm --filter api test:e2e
pnpm --filter api test:concurrency
```

## Pruebas

Las pruebas de integración usan PostgreSQL real, no mocks: el proyecto trata sobre transacciones y
concurrencia, y un mock no las demuestra. Cada fase tiene su comando de verificación en `PHASES.md`.

Las pruebas de concurrencia deben pasar de forma repetida. Un fallo intermitente es un fallo.
