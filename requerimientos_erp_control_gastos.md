# Requerimientos en el ERP para soportar Control de Gastos

**Fecha:** 2026-06-04
**Proyecto:** mar-erp-nodejs-api
**Relación:** Estos cambios en el ERP son necesarios para que el módulo de Control de Gastos (expenses) funcione completo.

---

## 1. Catálogo de Cuentas de Banco

### 1.1 Objetivo

Que el ERP tenga una pantalla para administrar el catálogo de bancos, cuentas y tarjetas. Actualmente la tabla `bank` ya existe en la DB pero no tiene CRUD completo desde la UI del ERP.

### 1.2 Tabla existente: `bank`

| Columna | Tipo | Uso |
|---|---|---|
| `id` | int PK | — |
| `name` | varchar(100) | Nombre descriptivo (ej: "Clara", "Omar ****9126") |
| `account` | varchar(30) | Número de cuenta (enmascarado en UI) |
| `is_active` | boolean | Activo/inactivo |
| `parent_id` | int FK → bank | Jerarquía: institución → cuenta → tarjeta |
| `organization_id` | int | Organización |

### 1.3 Jerarquía con usuarios asignados

```
bank (name="Clara", type="institution", parent_id=null)
  └── bank (name="Cuenta Principal", account="****1234", parent_id=1)    ← cuenta
        └── bank (name="Omar", account="****9126", user_id=5, parent_id=2)  ← tarjeta + usuario
        └── bank (name="Laura", account="****3344", user_id=8, parent_id=2) ← tarjeta + usuario
```

Cada tarjeta/subcuenta puede asociarse a un usuario. Esto permite:
- **Pre-selección:** al abrir el gasto, carga la tarjeta de quien está logueado (si tiene una asignada)
- **Trazabilidad:** reportes de quién usó qué cuenta
- **Flexibilidad:** todos ven todas las cuentas. El mensajero puede escoger otra si necesita

### 1.4 Cambio en la tabla `bank`

```sql
ALTER TABLE public.bank
    ADD COLUMN user_id integer REFERENCES public."user"(id);
```

`user_id` es informativo y para pre-selección. **No restringe visibilidad.** Todos los usuarios ven todas las cuentas. El ERP solo pre-selecciona la que corresponde al usuario logueado; el mensajero puede cambiarla.

### 1.5 Pantalla requerida

```
┌──────────────────────────────────────────────────────────┐
│  Catálogo de Bancos / Cuentas                            │
│                                                          │
│  [+ Nuevo]                                               │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ 🏦 Clara                                           │  │
│  │   ├── 💳 Cuenta Principal (****1234)               │  │
│  │   │     ├── 👤 Omar (****9126) - Activo            │  │
│  │   │     └── 👤 Laura (****3344) - Activo           │  │
│  │   └── 💳 Cuenta Secundaria (****5678)              │  │
│  │         └── 👤 Omar (****7890) - Inactivo          │  │
│  │                                                    │  │
│  │ 🏦 Afirme                                          │  │
│  │   └── 💳 Cuenta Principal (****9999)               │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 1.7 Formulario (crear/editar)

| Campo | Tipo | Requerido | Nota |
|---|---|---|---|
| Nombre | texto(100) | Sí | Ej: "Clara", "Omar ****9126", "Cuenta Principal" |
| Número de cuenta | texto(30) | Solo cuentas/tarjetas | Se guarda completo, se muestra enmascarado |
| Padre | select (bank) | No | NULL = institución. Si selecciona institución = cuenta. Si selecciona cuenta = tarjeta |
| **Usuario asignado** | **select (user)** | **Solo tarjetas** | **Dueño de esta tarjeta. Si se asigna, solo él la ve.** |
| Activo | checkbox | Sí | Default: true |

### 1.8 Endpoints

Ya existe `GET /banks/:id`. Agregar:
- `POST /banks` — Crear
- `PUT /banks/:id` — Actualizar
- `DELETE /banks/:id` — Soft-delete (desactivar)

---

## 2. Configuración de Método de Pago en Orden de Compra

### 2.1 Objetivo

Al crear una OdeC en el ERP, poder seleccionar con qué banco/cuenta/tarjeta se va a pagar. Este valor se hereda al módulo de expenses al registrar el gasto.

### 2.2 Cambio en la tabla `order`

Agregar columna:

```sql
ALTER TABLE public."order"
    ADD COLUMN bank_id integer REFERENCES public.bank(id);
```

### 2.3 Comportamiento

1. Al crear OdeC → campo opcional "Cuenta/Banco para pago" (select del catálogo de banks)
2. Al recibir la OdeC en expenses → el campo `bank_id` se pre-selecciona en el formulario de gasto
3. El surtidor o gerente puede cambiarlo si es necesario

---

## 3. Órdenes de Compra para Stock

### 3.1 Objetivo

Poder crear OdeC que no están ligadas a una OdeS específica, sino que son para **stock** (inventario del taller). Ejemplos: refacciones comunes, aceites, filtros que se tienen en existencia.

### 3.2 Tipos de OdeC

| Tipo | ¿Ligada a OdeS? | Uso |
|---|---|---|
| **Etiquetada** | Sí | Refacciones para un servicio específico de un cliente |
| **Stock** | No | Refacciones e insumos para inventario del taller |

### 3.3 Cambio en la tabla `order`

Agregar columna:

```sql
ALTER TABLE public."order"
    ADD COLUMN purchase_type varchar(20) DEFAULT 'labeled';
    -- 'labeled' = etiquetada a OdeS
    -- 'stock'   = para inventario
```

O alternativamente usar un `order_type_id` distinto o un campo `is_stock boolean`.

### 3.4 Pantalla requerida

La pantalla de creación de OdeC ya existe. Agregar:

```
┌──────────────────────────────────────────────────────────┐
│  Nueva Orden de Compra                                   │
│                                                          │
│  Tipo: ○ Etiquetada (para OdeS)  ● Stock (inventario)   │
│                                                          │
│  ── Si es Etiquetada ──                                  │
│  OdeS destino: [Seleccionar OdeS ▼]                     │
│                                                          │
│  ── Campos comunes ──                                    │
│  Proveedor: [Seleccionar ▼]                              │
│  Banco/Cuenta pago: [Seleccionar ▼]  ← nuevo             │
│                                                          │
│  Líneas:                                                 │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Producto         Cant  Precio   Total             │    │
│  │ Bujía NGK          6    $250    $1,500            │    │
│  │ Aceite 20W50      10     $80      $800            │    │
│  │ [+ Agregar línea]                                 │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Total estimado: $2,300                                  │
│                                                          │
│  [Guardar]                                               │
└──────────────────────────────────────────────────────────┘
```

### 3.5 Reglas

- Una OdeC tipo `labeled` requiere al menos una OdeS destino
- Una OdeC tipo `stock` NO requiere OdeS. Los productos van a inventario
- Los gastos sobre OdeC `stock` no aplican al KPI del 33% (no hay OdeS)
- En expenses, las OdeC `stock` aparecen en el listado pero sin % de rentabilidad
- Una OdeC puede surtir **múltiples OdeS** si comparten el mismo proveedor

### 3.6 Lógica de Unificación o Separación de Líneas por Alternativas

Cuando una OdeC etiquetada cubre más de una OdeS, el sistema debe comparar las alternativas de refacción de cada vehículo para decidir si unifica o separa las líneas.

#### Regla

```
Por cada producto requerido:
  1. Agrupar por producto (mismo ID de producto)
  2. Comparar alternativas disponibles para cada vehículo/OdeS
  3. Si TODOS los vehículos comparten las MISMAS alternativas → UNIFICAR (1 línea, suma de cantidades)
  4. Si algún vehículo tiene alternativas DIFERENTES → SEPARAR (1 línea por OdeS)
```

#### Ejemplo 1: Mismas alternativas → Unifica

```
Proveedor: Refaccionaria ABC
Producto: Bujía Denso (id=45)
  - OdeS 1000, Vehículo A: 2 bujías. Alternativas: NGK, Bosch, Champion
  - OdeS 1001, Vehículo B: 2 bujías. Alternativas: NGK, Bosch, Champion

Resultado: 1 sola línea en la OdeC
  ┌──────────────────────────────────────────────────────┐
  │ Producto: Bujía Denso    Cant: 4    Precio: $250     │
  │ OdeS: 1000, 1001         Total: $1,000               │
  │ Alternativas: NGK, Bosch, Champion                   │
  └──────────────────────────────────────────────────────┘
```

#### Ejemplo 2: Alternativas distintas → Separa

```
Proveedor: Refaccionaria ABC
Producto: Bujía Denso (id=45)
  - OdeS 1000, Vehículo A: 2 bujías. Alternativas: NGK, Bosch, Champion
  - OdeS 1001, Vehículo C: 2 bujías. Alternativas: NGK, Denso Iridium (no acepta Bosch ni Champion)

Resultado: 2 líneas separadas en la OdeC
  ┌──────────────────────────────────────────────────────┐
  │ #1 Producto: Bujía Denso  Cant: 2    Precio: $250    │
  │    OdeS: 1000 (Vehículo A)        Total: $500        │
  │    Alternativas: NGK, Bosch, Champion                 │
  ├──────────────────────────────────────────────────────┤
  │ #2 Producto: Bujía Denso  Cant: 2    Precio: $250    │
  │    OdeS: 1001 (Vehículo C)        Total: $500        │
  │    Alternativas: NGK, Denso Iridium                   │
  └──────────────────────────────────────────────────────┘
```

#### ¿Por qué es importante?

El surtidor/mensajero cuando va a comprar necesita saber:
- Si el producto específico no está disponible, ¿qué alternativas puede comprar?
- Si las alternativas son las mismas para todos los vehículos, compra todo junto
- Si son diferentes, necesita separar la compra físicamente

#### Comportamiento en el formulario de creación de OdeC

```
┌──────────────────────────────────────────────────────────────┐
│  Nueva OdeC - Refaccionaria ABC                              │
│                                                              │
│  OdeS a surtir: [1000 (Vehículo A)] [1001 (Vehículo C)]     │
│                                                              │
│  ── Productos requeridos ──                                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ ☑ Bujía Denso - 4 unids.                               │  │
│  │   ⚠️ Alternativas distintas entre vehículos             │  │
│  │   → Se separará en 2 líneas                            │  │
│  │       Línea A: 2 p/ OdeS 1000 (alt: NGK, Bosch)       │  │
│  │       Línea B: 2 p/ OdeS 1001 (alt: NGK, Iridium)     │  │
│  │                                                        │  │
│  │ ☑ Filtro de aceite - 2 unids.                          │  │
│  │   ✅ Mismas alternativas → 1 línea unificada           │  │
│  │       Línea: 2 p/ OdeS 1000, 1001 (alt: Fram, Wix)    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Total líneas generadas: 3                                   │
│  Total estimado: $1,800                                      │
│                                                              │
│  [Guardar OdeC]                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Catálogo de Categorías de Gasto

### 4.1 Objetivo

Un catálogo jerárquico auto-referenciado para clasificar gastos que no están ligados a una OdeC (gastos operativos). Ya existe la tabla `purchase_category` pero se necesita una pantalla para administrarla.

### 4.2 Tabla existente: `purchase_category`

| Columna | Tipo |
|---|---|
| `id` | int PK |
| `code` | varchar(64) |
| `name` | varchar(256) |
| `parent_id` | int FK → purchase_category |
| `is_active` | boolean |
| `organization_id` | int |

### 4.3 Jerarquía (ejemplo)

```
Gastos Operativos
  ├── Instalaciones
  │     ├── Renta
  │     └── Mantenimiento
  ├── Servicios
  │     ├── Luz
  │     ├── Agua
  │     ├── Internet
  │     └── Teléfono
  ├── Oficina
  │     ├── Papelería
  │     └── Equipo de cómputo
  ├── Taller
  │     ├── Herramientas
  │     ├── Insumos de limpieza
  │     └── Uniformes
  ├── Vehículos
  │     ├── Combustible
  │     └── Mantenimiento
  └── Otros
        ├── Capacitación
        ├── Publicidad
        └── Impuestos
```

### 4.4 Pantalla requerida

```
┌──────────────────────────────────────────────────────────┐
│  Catálogo de Categorías de Gasto                         │
│                                                          │
│  [+ Nueva Categoría]                                     │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ 📁 Gastos Operativos                               │  │
│  │   ├── 📁 Instalaciones                             │  │
│  │   │     ├── 📄 Renta                               │  │
│  │   │     └── 📄 Mantenimiento                       │  │
│  │   ├── 📁 Servicios                                 │  │
│  │   │     ├── 📄 Luz                                 │  │
│  │   │     ├── 📄 Agua                                │  │
│  │   │     ├── 📄 Internet                            │  │
│  │   │     └── 📄 Teléfono                            │  │
│  │   ├── 📁 Taller                                    │  │
│  │   │     ├── 📄 Herramientas                        │  │
│  │   │     └── 📄 Insumos de limpieza                 │  │
│  │   └── 📁 Otros                                     │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 4.5 Formulario (crear/editar categoría)

| Campo | Tipo | Requerido |
|---|---|---|
| Código | texto(64) | Sí, único por organización |
| Nombre | texto(256) | Sí |
| Padre | select (purchase_category) | No |
| Activo | checkbox | Sí |

### 4.6 Endpoints

- `GET /purchase-categories` — Ya existe. Agregar soporte para vista de árbol (`?tree=true`)
- `POST /purchase-categories` — Ya existe
- `PUT /purchase-categories/:id` — Ya existe
- `DELETE /purchase-categories/:id` — Soft-delete (desactivar). Validar que no tenga gastos asociados.

---

## 5. Resumen de Cambios en ERP

| # | Cambio | Tipo | Tabla afectada |
|---|---|---|---|
| 1 | Pantalla CRUD de bancos/cuentas/tarjetas | Nueva pantalla | `bank` |
| 2 | `bank_id` en orden de compra | Nueva columna | `order` |
| 3 | Tipo de OdeC: etiquetada vs stock + **lógica unificar/separar líneas por alternativas** | Nueva columna + pantalla + lógica | `order`, `order_detail` |
| 4 | Pantalla CRUD de categorías de gasto (árbol jerárquico) | Nueva pantalla | `purchase_category` |

### Entregables

| # | ¿Qué se entrega? | Días |
|---|---|---|
| 1 | CRUD bancos (backend + pantalla ERP) | 0.5-1 |
| 2 | Columna `bank_id` en OdeC + selector en formulario | 0.5 |
| 3 | Tipo Stock/Labeled + unificar/separar líneas por alternativas + pantalla OdeC | 1-1.5 |
| 4 | CRUD purchase_category + pantalla árbol | 0.5-1 |
| **Total ERP** | | **2.5-4 días** |

---

> **Nota:** Estos cambios son en el ERP (mar-erp-nodejs-api). Son independientes de los cambios en el módulo expenses pero necesarios para que la integración funcione correctamente.
