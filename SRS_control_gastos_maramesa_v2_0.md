# Software Requirements Specification (SRS) — Módulo de Control de Gastos MARAMESA

**Versión:** 2.0
**Fecha:** 2026-06-03
**Proyecto:** Control de Gastos integrado con ERP de talleres (mar-erp-nodejs-api)
**Tipo:** SRS según IEEE 830 (adaptado)

---

## Tabla de Contenido

1. [Introducción](#1-introducción)
2. [Descripción General](#2-descripción-general)
3. [Modelo de Dominio](#3-modelo-de-dominio)
4. [Historias de Usuario](#4-historias-de-usuario)
5. [Requerimientos Funcionales](#5-requerimientos-funcionales)
6. [Requerimientos No Funcionales](#6-requerimientos-no-funcionales)
7. [Contratos de API](#7-contratos-de-api)
8. [Definition of Done por Historia](#8-definition-of-done-por-historia)
9. [Reglas de Negocio](#9-reglas-de-negocio)
10. [Análisis de Brecha (Gap Analysis)](#10-análisis-de-brecha-gap-analysis)
11. [Plan de Implementación](#11-plan-de-implementación)
12. [Criterios de Aceptación Final](#12-criterios-de-aceptación-final)
13. [Fuera del MVP](#13-fuera-del-mvp)

---

## 1. Introducción

### 1.1 Propósito

Este documento especifica los requerimientos funcionales y no funcionales del Módulo de Control de Gastos para MARAMESA, un sistema de gestión de talleres mecánicos. El módulo debe permitir registrar, controlar y reportar los gastos relacionados con compras de refacciones, servicios, fletes, consumibles y otros conceptos necesarios para surtir las Órdenes de Servicio (OdeS).

### 1.2 Alcance

El sistema cubre:
- Catálogo de proveedores
- Catálogo de bancos/cuentas/tarjetas con estructura jerárquica
- Órdenes de Compra (OdeC) agrupadas por proveedor
- Líneas de OdeC con tipos mixtos (productos trazables, conceptos no trazables, consumibles, etc.)
- Aplicación de líneas de OdeC a Órdenes de Servicio (OdeS)
- Registro de gastos con evidencias (factura, ticket, comprobante de pago)
- Cálculo de rentabilidad: % de refacciones por OdeS (regla del 33%)
- Estado de cuenta interno por banco/cuenta/tarjeta
- Reportes operativos y gerenciales

### 1.3 Definiciones, Acrónimos y Abreviaturas

| Término | Definición |
|---|---|
| **OdeS** | Orden de Servicio. Unidad de trabajo autorizada para un cliente/vehículo. Es la **unidad de rentabilidad**. |
| **OdeC** | Orden de Compra. Agrupación de productos/conceptos por proveedor. Es la **unidad logística de compra**. |
| **Gasto** | Registro de salida de dinero asociado a una OdeC. Contiene factura, ticket o comprobante. |
| **Aplicación** | Distribución de una línea de OdeC hacia una o varias OdeS. |
| **KPI 33%** | Indicador clave: el gasto en refacciones no debe superar el 33% de la cantidad a cobrar al cliente. |
| **ERP** | Enterprise Resource Planning — el backend mar-erp-nodejs-api. |

### 1.4 Referencias

- [API Backend](mar-erp-nodejs-api/) — NestJS + TypeORM
- [Knowledge Graph](mar-erp-nodejs-api/.understand-anything/knowledge-graph.json)
- [Código Frontend actual](expenses/src/) — React + TypeScript + Tailwind

### 1.5 Audiencia

- **Stakeholders de negocio:** Dueños y administradores de talleres MARAMESA
- **Equipo de desarrollo:** Full-stack developers trabajando en React (frontend) y NestJS (backend)
- **QA:** Testers para validación funcional

---

## 2. Descripción General

### 2.1 Perspectiva del Producto

El Módulo de Control de Gastos es una aplicación web (React SPA) que se comunica con el ERP central (NestJS API) mediante REST/JSON. Los datos maestros (clientes, vehículos, OdeS) provienen del ERP. El módulo de gastos extiende el ERP con capacidades de compras, proveedores, bancos y análisis de rentabilidad.

### 2.2 Objetivo de Negocio

> El gasto promedio en refacciones no debe ser mayor al 33% de la cantidad a cobrar al cliente.

La métrica se calcula por OdeS y se consolida mensualmente como promedio ponderado.

### 2.3 Flujo Operativo

```
[ERP] OdeS autorizadas/agendadas
   ↓
[ERP] Necesidades de piezas/refacciones → Agrupación por proveedor → Generación de OdeC
   ↓
[Expenses] Recepción de OdeC desde ERP (order_type.class='purchase')
   ↓
[Expenses] Clasificación de líneas (tipo, afecta_refacciones, método de asignación)
   ↓
[Expenses] Registro de gasto/factura/ticket/comprobante sobre línea de OdeC
   ↓
[Expenses] Aplicación de cada línea a una o varias OdeS (line_application)
   ↓
[Expenses] Cálculo de % de refacciones y % de costo total por OdeS
   ↓
[Expenses] Registro automático en estado de cuenta bancario (bank_transaction)
   ↓
[Expenses] Conciliación bancaria (Excel) + Reportes
```

**Nota:** La creación de OdeC (agrupar productos por proveedor, generar folio) ocurre en el ERP. El módulo expenses **recibe** las OdeC y se enfoca en: clasificación de líneas, registro de gastos con evidencias, aplicación de costos a OdeS, estado de cuenta bancario y reportes de rentabilidad.

### 2.4 Roles de Usuario

| Rol | ¿Quién es? | ¿Qué puede hacer en un gasto? |
|---|---|---|
| **Administrador** | Dueño o gerente general | Acceso total. Ajustes manuales de saldo. Configuración del sistema. |
| **Gerente de Compras** | Encargado de compras y proveedores | **Cambiar proveedor** en el gasto. Si el nuevo proveedor tiene OdeC activa → se vincula. Si no → crea nueva OdeC. Cambiar método de pago, monto, producto (con alternativas), evidencias, código de barras. |
| **Surtidor / Mensajero** | Quien recoge y entrega las refacciones | **Solo puede cambiar:** método de pago (banco/cuenta/tarjeta) y monto total. Puede cambiar producto solo si la línea tiene alternativas. Capturar evidencias (fotos, ticket). Escanear código de barras. **No puede** cambiar proveedor. |
| **Contador / Auditor** | Revisión financiera | Ver reportes, estado de cuenta. No registra gastos operativos. |

### 2.5 Reglas de Permisos en Formulario de Gasto

| Campo | Surtidor | Gerente Compras | Admin |
|---|---|---|---|
| OdeC / Línea | Solo lectura | Solo lectura (viene del ERP) | Solo lectura |
| **Proveedor** | **Bloqueado** | **Editable** (cambia OdeC o crea nueva) | Editable |
| **Método de pago (Banco)** | **Editable** | Editable | Editable |
| **Producto** | Solo si hay alternativas | Editable | Editable |
| Cantidad | Solo lectura | Editable | Editable |
| Precio unitario | Solo lectura | Editable | Editable |
| **Monto total** | **Editable** (cantidad × precio) | Editable | Editable |
| Ticket/Folio | Editable | Editable | Editable |
| Evidencias (fotos, PDF, XML) | **Editable** | Editable | Editable |
| Código de barras | **Editable** | Editable | Editable |

### 2.6 Nota: Cálculo del Monto Total

```
monto_total = cantidad × precio_unitario
```

Es lo que gasta el taller. Se pre-calcula automáticamente al seleccionar línea. El surtidor puede ajustarlo (ej: diferencia de precio real vs estimado).

---

## 3. Modelo de Dominio

### 3.1 Arquitectura

El ERP (mar-erp-nodejs-api) es el sistema de registro maestro. Crea y gestiona las Órdenes de Compra (OdeC) y las Órdenes de Servicio (OdeS). El módulo **expenses** consume estas entidades y agrega la capa de gastos, conciliación bancaria y rentabilidad.

```
[ERP]                            [Expenses]
  │                                  │
  ├─ Crea OdeS (service orders)      │
  ├─ Crea OdeC (purchase orders) ────│─ Recibe OdeC + líneas
  ├─ Catálogo de productos           │─ Clasifica líneas
  ├─ Catálogo de proveedores         │─ Registra gastos + evidencias
  ├─ Clientes y vehículos            │─ Aplica costos a OdeS
  │                                  │─ Genera movimientos bancarios
  │                                  │─ Importa Excel bancario
  │                                  │─ Calcula rentabilidad (33%)
  │                                  │─ Reportes
```

### 3.2 Entidades Existentes (NO modificar estructura)

#### `order` — Orden de Compra (OdeC)
Las OdeC llegan del ERP. El módulo expenses las consume para registrar gastos.

| Columna | Tipo | Uso en expenses |
|---|---|---|
| `id` | int PK | ID de OdeC |
| `code` | varchar(64) | Folio |
| `name` | text | Descripción de la compra |
| `supplier_id` | int FK → supplier | Proveedor de la compra |
| `folio` | varchar(256) | Número de folio/OC |
| `total` | numeric(10,2) | Total de la compra |
| `subtotal` | numeric(10,2) | Subtotal compra |
| `tax_amount` | numeric(10,2) | IVA compra |
| `status_id` | int FK → status | Estado de la compra |
| `customer_id` | int FK | Cliente (si la OdeC es para un cliente específico) |
| `fleet_vehicle_id` | int FK | Vehículo (si aplica) |

#### `order_detail` — Líneas de la OdeC
Cada línea representa un producto, servicio o concepto dentro de una OdeC.

| Columna | Tipo | Uso |
|---|---|---|
| `id` | int PK | — |
| `order_id` | int FK → order | OdeC padre |
| `product_id` | int FK → product | Producto |
| `product_type_id` | int FK → product_type | Tipo de producto |
| `quantity` | numeric(5,2) | Cantidad |
| `unit_cost` | numeric(10,2) | Costo unitario |
| `unit_price` | numeric(10,2) | Precio de venta |
| `total_price` | numeric(10,2) | Total de la línea |
| `supplier_id` | int FK → supplier | Proveedor de la línea |

#### `expense` — Gastos registrados
| Columna | Tipo | Uso |
|---|---|---|
| `id` | int PK | — |
| `order_id` | int FK → order | OdeC (NULL si es gasto operativo) |
| `order_detail_id` | int FK → order_detail | Línea específica (NULL si es gasto general) |
| `bank_id` | int FK → bank | Banco/cuenta desde donde se pagó |
| `supplier_id` | int FK → supplier | Proveedor |
| `expense_category_id` | int FK → purchase_category | Categoría del gasto (requerido si no hay OdeC) |
| `amount_spent` | numeric(10,4) | Monto gastado |
| `code` | varchar(30) | Folio/ticket/factura |
| `invoice_pdf` | text | Factura PDF (base64) |
| `invoice_xml` | text | Factura XML |
| `receipt_file` | text | Ticket/comprobante (base64) |
| `receipt_is_image` | boolean | Si el comprobante es imagen |
| `evidence_file` | text | Evidencia física/foto (base64) |
| `evidence_is_image` | boolean | Si la evidencia es imagen |
| `parent_id` | int FK → expense | Auto-referencia (para reversiones) |

**Nota sobre dump.sql:** `order_detail_id`, `receipt_is_image`, `evidence_is_image` ya existen en el dump actual.

#### `purchase_category` — Categorías de gasto
Ya existe en la DB. Se reutiliza para clasificar tanto líneas de OdeC como gastos operativos sin OdeC.

| Columna | Tipo |
|---|---|
| `id` | int PK |
| `code` | varchar(64) |
| `name` | varchar(256) |
| `parent_id` | int FK → purchase_category (jerarquía) |
| `is_active` | boolean |

#### `bank` — Bancos/cuentas/tarjetas (jerarquía auto-referenciada)
| Columna | Tipo | Uso |
|---|---|---|
| `id` | int PK | — |
| `name` | varchar(100) | Nombre descriptivo |
| `account` | varchar(30) | Número de cuenta (enmascarado en UI) |
| `is_active` | boolean | Activo/inactivo |
| `parent_id` | int FK → bank | Jerarquía: institución → cuenta → tarjeta |

```
bank (id=1, name="Clara", parent_id=null)              ← Institución
  └── bank (id=2, name="Cuenta Principal", parent_id=1) ← Cuenta
        └── bank (id=3, name="Omar ****9126", parent_id=2) ← Tarjeta
```

#### `supplier` — Proveedores
| Columna | Tipo |
|---|---|
| `id` | int PK |
| `code` | varchar(64) |
| `name` | varchar(256) |
| `is_active` | boolean |
| `supplier_type_id` | int FK → supplier_type |

---

### 3.3 Entidades NUEVAS Requeridas (Migraciones)

#### `line_application` — Aplicación de líneas de OdeC a OdeS

```sql
CREATE TABLE public.line_application (
    id serial PRIMARY KEY,
    organization_id integer NOT NULL,
    order_detail_id integer NOT NULL REFERENCES public.order_detail(id),
    service_order_id integer NOT NULL,  -- OdeS destino (gestionada por ERP)
    applied_quantity numeric(10,2),     -- Cantidad aplicada (productos trazables)
    applied_amount numeric(10,2),       -- Importe aplicado (conceptos no trazables)
    assignment_method varchar(20) NOT NULL,
    created timestamp with time zone DEFAULT now(),
    created_by integer NOT NULL,
    updated timestamp with time zone DEFAULT now(),
    updated_by integer NOT NULL,
    version integer DEFAULT 0 NOT NULL,
    is_active boolean DEFAULT true NOT NULL
);
```

#### `bank_transaction` — Movimientos bancarios (estado de cuenta interno)

```sql
CREATE TABLE public.bank_transaction (
    id serial PRIMARY KEY,
    organization_id integer NOT NULL,
    bank_id integer NOT NULL REFERENCES public.bank(id),
    expense_id integer REFERENCES public.expense(id),
    transaction_date timestamp with time zone NOT NULL,
    transaction_type varchar(10) NOT NULL,  -- 'in', 'out', 'adjustment'
    amount numeric(10,2) NOT NULL,
    balance numeric(10,2) NOT NULL,
    description varchar(500),
    supplier_id integer REFERENCES public.supplier(id),
    purchase_order_id integer REFERENCES public."order"(id),
    created timestamp with time zone DEFAULT now(),
    created_by integer NOT NULL,
    updated timestamp with time zone DEFAULT now(),
    updated_by integer NOT NULL,
    version integer DEFAULT 0 NOT NULL,
    is_active boolean DEFAULT true NOT NULL
);
```

#### `supplier_alias` — Aliases de proveedores para matching bancario

```sql
CREATE TABLE public.supplier_alias (
    id serial PRIMARY KEY,
    organization_id integer NOT NULL,
    supplier_id integer NOT NULL REFERENCES public.supplier(id),
    alias varchar(200) NOT NULL,
    is_active boolean DEFAULT true NOT NULL,
    UNIQUE(organization_id, alias)
);
```

#### `deferred_payment` — Compras diferidas (MSI / con interés)

```sql
CREATE TABLE public.deferred_payment (
    id serial PRIMARY KEY,
    organization_id integer NOT NULL,
    bank_transaction_id integer NOT NULL REFERENCES public.bank_transaction(id),
    deferred_type varchar(10) NOT NULL,    -- 'msi', 'interest'
    total_months integer NOT NULL,
    interest_rate numeric(5,2),            -- Solo si deferred_type = 'interest'
    total_amount numeric(10,2) NOT NULL,
    monthly_payment numeric(10,2) NOT NULL,
    start_date date NOT NULL,
    months_paid integer DEFAULT 0,
    total_paid numeric(10,2) DEFAULT 0,
    is_active boolean DEFAULT true NOT NULL,
    created timestamp with time zone DEFAULT now(),
    created_by integer NOT NULL,
    updated timestamp with time zone DEFAULT now(),
    updated_by integer NOT NULL
);
```

---

### 3.4 Columnas NUEVAS en Tablas Existentes

#### `order_detail` — Columnas de clasificación

```sql
ALTER TABLE public.order_detail
    ADD COLUMN line_type varchar(20) DEFAULT 'product',
    ADD COLUMN affects_parts boolean DEFAULT true,
    ADD COLUMN affects_total_cost boolean DEFAULT true,
    ADD COLUMN assignment_method varchar(20) DEFAULT 'direct';
```

#### `expense` — Columnas para gastos operativos

```sql
ALTER TABLE public.expense
    ADD COLUMN expense_category_id integer REFERENCES public.purchase_category(id);
```

**Nota:** `order_detail_id` ya existe en el dump actual (`db/dump.sql`). `receipt_is_image` y `evidence_is_image` también.

---

### 3.5 Diagrama ER Simplificado

```
[ERP]                              [Expenses]
  │                                    │
  │ OdeS (service order)               │ OdeS es referenciada por line_application
  │  ├─ customer                       │
  │  ├─ fleet_vehicle                  │
  │  └─ amount_to_charge               │
  │                                    │
  │ OdeC = order + order_detail ───────│─ Recibe y consume
  │                                    │
  ▼                                    ▼
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   supplier   │     │  order (OdeC)    │     │purchase_category │
│              │     │                  │     │(expense_category)│
│  id PK       │◄────│  supplier_id FK  │     │                  │
│  name        │     │  folio           │     │  id PK           │
│  is_active   │     │  total           │     │  code            │
└──────┬───────┘     │  status_id       │     │  name            │
       │             └────────┬─────────┘     └────────▲─────────┘
       │                      │                        │
       │ 1:N                  │ 1:N                    │ (si no hay OdeC)
       │                      │                        │
       ▼                      ▼                        │
┌──────────────┐     ┌──────────────────┐              │
│supplier_alias│ *** │  order_detail    │              │
│              │     │  (líneas OdeC)   │              │
│  supplier_id │     │                  │              │
│  alias       │     │  order_id FK     │              │
└──────────────┘     │  product_id FK   │              │
                     │  quantity        │              │
                     │  unit_cost       │              │
                     │  total_price     │              │
                     │  line_type ***   │              │
                     │  affects_parts***│              │
                     └────────┬─────────┘              │
                              │                        │
                              │ 1:N (line_application) │
                              ▼                        │
                     ┌──────────────────┐              │
                     │ line_application │ ***          │
                     │  order_detail_id │              │
                     │  service_order_id│→ OdeS (ERP)  │
                     │  applied_quantity│              │
                     │  applied_amount  │              │
                     └──────────────────┘              │
                                                       │
       ┌───────────────────────────────────────────────┘
       │ 1:N
       ▼
┌──────────────┐
│   expense    │
│  order_id FK │ (NULL si operativo)
│  order_detail_id FK │ (NULL si sin línea)
│  expense_category_id│→ purchase_category
│  bank_id FK  │
│  supplier_id │
│  amount_spent│
│  code (folio)│
│  archivos    │
└──────┬───────┘
       │ trigger: al crear expense → insert bank_transaction
       ▼
┌──────────────────┐     ┌──────────────────┐
│ bank_transaction │ *** │      bank        │
│  bank_id FK ──────────│→ id PK           │
│  expense_id FK   │     │  name            │
│  transaction_date│     │  account         │
│  type (in/out)   │     │  parent_id (FK)  │
│  amount          │     │  is_active       │
│  balance         │     └──────────────────┘
└────────┬─────────┘
         │ 1:N (opcional)
         ▼
┌──────────────────┐
│ deferred_payment │ ***
│  bank_transaction│
│  deferred_type   │ (msi / interest)
│  total_months    │
│  monthly_payment │
│  months_paid     │
│  total_paid      │
└──────────────────┘

*** = NUEVA tabla o columna
```

### 3.6 Tipos de Línea de OdeC (`order_detail.line_type`)

| Tipo | Ejemplo | Se asigna por | Afecta 33% |
|---|---|---|---|
| `product` | Bujía, filtro, balata | Cantidad | Sí |
| `non_traceable` | Flete, cargo urgente | Importe | Normalmente no |
| `consumable` | Aceite a granel, tornillos | Importe/cantidad | Según regla |
| `tool` | Llave, scanner | Importe | No |
| `discount` | Descuento proveedor | Importe | Depende |
| `operational` | Papelería, comida | Importe | No |

### 3.7 Métodos de Asignación (`line_application.assignment_method`)

| Método | Uso |
|---|---|
| `direct` | Una línea pertenece a una sola OdeS |
| `manual` | El usuario decide cuánto cargar a cada OdeS |
| `prorrateo_amount` | Prorrateo por importe de refacciones |
| `prorrateo_quantity` | Prorrateo por cantidad de piezas |
| `equal_split` | División igualitaria |
| `none` | No se aplica a ninguna OdeS |

---

## 4. Historias de Usuario

### Épica 1: Catálogos Base

#### HU-01: Gestión de Proveedores

**Como** comprador,
**quiero** registrar y administrar proveedores con su información comercial y fiscal,
**para** tener un catálogo actualizado al momento de generar órdenes de compra.

**Criterios de Aceptación:**
- CA-01.1: Puedo crear un proveedor con nombre comercial (requerido), razón social, RFC, teléfono y estado activo/inactivo.
- CA-01.2: Puedo editar cualquier campo de un proveedor existente.
- CA-01.3: Puedo desactivar un proveedor sin eliminarlo (soft delete).
- CA-01.4: Puedo buscar proveedores por nombre, código o RFC.
- CA-01.5: El sistema muestra el nombre del proveedor en toda la UI, nunca solo su ID.
- CA-01.6: Puedo ver el historial de compras asociado a cada proveedor.

---

#### HU-02: Gestión de Bancos, Cuentas y Tarjetas

**Como** administrador,
**quiero** registrar instituciones financieras, cuentas principales y tarjetas/subcuentas con su estructura jerárquica,
**para** controlar desde qué instrumento se paga cada compra y mantener un estado de cuenta interno.

**Criterios de Aceptación:**
- CA-02.1: Puedo registrar una institución financiera (nombre, tipo, límite global opcional, activa/inactiva).
- CA-02.2: Puedo crear cuentas principales asociadas a una institución (número de cuenta enmascarado, titular, fecha de corte, día de pago, saldo inicial, límite de crédito, moneda).
- CA-02.3: Puedo crear tarjetas/subcuentas asociadas a una cuenta principal (titular, terminación, tipo, límite individual).
- CA-02.4: El sistema enmascara números de cuenta/tarjeta (ej: `****4467`).
- CA-02.5: El sistema NUNCA almacena tokens bancarios, contraseñas, NIPs ni credenciales sensibles.
- CA-02.6: Puedo ver el disponible por tarjeta y el acumulado global por cuenta principal.

---

### Épica 2: Órdenes de Compra (OdeC)

#### HU-03: Visualización y Gestión de OdeC

**Como** comprador,
**quiero** ver las Órdenes de Compra que llegan del ERP con sus líneas, y poder clasificarlas,
**para** saber qué proveedor surte qué y preparar la aplicación de costos a las OdeS.

**Contexto:** Las OdeC son creadas por el ERP (no desde expenses). Expenses las recibe vía API y permite gestionar la clasificación de líneas y el registro de gastos asociados.

**Criterios de Aceptación:**
- CA-03.1: Veo el listado de OdeC con: folio, proveedor, total, estado, fecha.
- CA-03.2: Puedo ver el detalle de cada OdeC con sus líneas (productos, cantidades, costos).
- CA-03.3: Cada línea permite marcar si afecta refacciones (`affectsParts`) y si afecta costo total (`affectsTotalCost`).
- CA-03.4: Puedo cambiar el tipo de línea (`line_type`) para clasificar correctamente cada concepto.
- CA-03.5: La OdeC muestra el total de gastos registrados vs el total estimado.
- CA-03.6: Puedo cambiar el estado de una OdeC (ej: marcar como "surtida" cuando todos los gastos están registrados).

---

### Épica 3: Gastos y Evidencias

#### HU-04: Registro de Gasto sobre OdeC

**Como** comprador,
**quiero** registrar un gasto real sobre una OdeC con factura, ticket y comprobante de pago,
**para** tener trazabilidad financiera completa de cada compra.

**Criterios de Aceptación:**
- CA-04.1: Selecciono una OdeC existente (el proveedor se autocompleta desde la OdeC).
- CA-04.2: Selecciono el banco/cuenta/tarjeta desde donde se pagó.
- CA-04.3: Ingreso fecha del gasto, monto total y folio/ticket.
- CA-04.4: Puedo adjuntar factura PDF, factura XML, ticket/comprobante (imagen) y evidencia física (fotos).
- CA-04.5: Desde celular, puedo usar la cámara para tomar foto directa del ticket o productos.
- CA-04.6: El sistema valida que los archivos no excedan 10 MB y tengan formatos permitidos.
- CA-04.7: Al registrar el gasto, se crea automáticamente un movimiento en el estado de cuenta interno.

---

#### HU-05: Aplicación de Líneas de OdeC a OdeS

**Como** comprador,
**quiero** distribuir cada línea de una OdeC entre una o varias OdeS,
**para** que el costo se refleje correctamente en la rentabilidad de cada servicio.

**Criterios de Aceptación:**
- CA-05.1: Para productos trazables, distribuyo por cantidad (no puedo exceder la cantidad comprada).
- CA-05.2: Para conceptos no trazables (fletes, servicios), distribuyo por importe usando prorrateo o asignación manual.
- CA-05.3: Puedo ver en tiempo real cuánto se ha aplicado y cuánto falta por aplicar de cada línea.
- CA-05.4: El sistema no permite aplicar más cantidad de la comprada (validación estricta).
- CA-05.5: Una línea puede no aplicarse a ninguna OdeS si es gasto operativo o herramienta (`assignmentMethod: none`).
- CA-05.6: Puedo modificar aplicaciones mientras la OdeC no esté cerrada (queda registro de auditoría).

---

### Épica 4: Rentabilidad

#### HU-06: Visualización de Rentabilidad por OdeS

**Como** administrador,
**quiero** ver el porcentaje de refacciones y costo total de cada OdeS,
**para** identificar rápidamente servicios con sobrecosto y tomar decisiones correctivas.

**Criterios de Aceptación:**
- CA-06.1: Cada OdeS muestra: cantidad a cobrar, refacciones aplicadas, % refacciones, otros costos, costo total, % costo total.
- CA-06.2: El % de refacciones se calcula como: `(total refacciones aplicadas / cantidad a cobrar) × 100`.
- CA-06.3: El semáforo 33% se aplica visualmente (verde ≤ 32.99%, amarillo = 33.00%, rojo > 33%).
- CA-06.4: Puedo filtrar OdeS por: todas, alerta (>33%), críticas (>100%).
- CA-06.5: Al hacer clic en una OdeS, veo el detalle de qué OdeC, líneas y gastos la afectaron.
- CA-06.6: La cantidad a cobrar se obtiene del ERP y no se recaptura manualmente.

---

#### HU-07: Dashboard de Rentabilidad Mensual

**Como** administrador,
**quiero** ver un resumen mensual del % de refacciones ponderado,
**para** monitorear la salud financiera del taller mes a mes.

**Criterios de Aceptación:**
- CA-07.1: Se muestra una tabla con: mes, total a cobrar, refacciones aplicadas, % refacciones mensual, estado (correcto/límite/excedido).
- CA-07.2: El % mensual es promedio ponderado: `total refacciones mes / total a cobrar mes × 100`.
- CA-07.3: Puedo navegar de un mes a los detalles de OdeS que lo componen (drill-down).
- CA-07.4: Se muestra una gráfica de tendencia de los últimos 12 meses.

---

### Épica 5: Reportes y Control Financiero

#### HU-08: Estado de Cuenta Interno

**Como** contador,
**quiero** ver un estado de cuenta interno por banco/cuenta/tarjeta con entradas, salidas y saldo,
**para** conciliar gastos sin depender del portal bancario.

**Criterios de Aceptación:**
- CA-08.1: Veo movimientos ordenados por fecha/hora con: fecha, hora, tipo, proveedor, OdeC, OdeS, entrada, salida, saldo.
- CA-08.2: El saldo se calcula como: `saldo_inicial + entradas - salidas` (cheques) o `limite - cargos + pagos` (crédito).
- CA-08.3: Puedo filtrar por rango de fechas, banco/cuenta, usuario/tarjeta, proveedor y OdeC.
- CA-08.4: Cada movimiento es trazable a la OdeC y gasto que lo originó.
- CA-08.5: Los ajustes manuales de saldo requieren permiso especial y quedan auditados (quién, cuándo, motivo).

---

#### HU-09: Reportes Operativos

**Como** administrador,
**quiero** generar reportes por OdeS, OdeC, proveedor y banco,
**para** tener visibilidad completa de las operaciones del taller.

**Criterios de Aceptación:**
- CA-09.1: Reporte por OdeS: muestra todas las columnas de rentabilidad + OdeC relacionadas.
- CA-09.2: Reporte por OdeC: muestra proveedor, total, banco, estado, OdeS surtidas, factura (sí/no).
- CA-09.3: Reporte por proveedor: muestra total comprado, número de OdeC, OdeS surtidas en un rango de fechas.
- CA-09.4: Reporte por banco: muestra total salidas, entradas, saldo/disponible por tarjeta y cuenta.
- CA-09.5: Todos los reportes permiten exportar a PDF y Excel.
- CA-09.6: Los reportes no tienen fondo de imagen que dificulte la lectura.

---

### Épica 6: Seguridad y Auditoría

#### HU-10: Trazabilidad y Auditoría

**Como** administrador,
**quiero** que cada operación crítica quede registrada con usuario, fecha y hora,
**para** poder auditar cambios y detectar irregularidades.

**Criterios de Aceptación:**
- CA-10.1: Toda creación/edición/cancelación de OdeC, gasto, aplicación y ajuste de saldo queda registrada.
- CA-10.2: Al cancelar un gasto, se revierten automáticamente sus acumulados y movimientos bancarios.
- CA-10.3: Los números de cuenta/tarjeta se muestran siempre enmascarados en UI y reportes.
- CA-10.4: Las vistas de detalle muestran quién creó y quién modificó cada registro.
- CA-10.5: Los intentos de aplicar más cantidad de la comprada se registran como advertencia de seguridad.

---

### Épica 7: Conciliación Bancaria

#### HU-11: Aliases de Proveedores

**Como** contador,
**quiero** registrar múltiples alias para un mismo proveedor,
**para** que al importar estados de cuenta bancarios el sistema pueda identificar al proveedor aunque el nombre en el banco sea distinto al registrado.

**Criterios de Aceptación:**
- CA-11.1: Desde la ficha de un proveedor, puedo agregar alias (ej: "autozonemx", "AUTOZONE MERIDA").
- CA-11.2: Un mismo alias no puede estar asociado a dos proveedores distintos.
- CA-11.3: Puedo eliminar un alias sin afectar al proveedor.
- CA-11.4: El matching de importación bancaria busca por nombre de proveedor y por todos sus aliases.

---

#### HU-12: Importación de Estado de Cuenta Bancario (Excel)

**Como** contador,
**quiero** subir el archivo Excel del estado de cuenta del banco y que el sistema haga match automático contra proveedores y OdeCs,
**para** reducir el tiempo de conciliación manual y detectar gastos no registrados.

**Criterios de Aceptación:**
- CA-12.1: Subo un Excel bancario y el sistema muestra: matched (con proveedor/OdeC), unmatched (sin identificar).
- CA-12.2: Para cada movimiento, el matching intenta: nombre del proveedor → alias → monto exacto contra OdeC.
- CA-12.3: Para unmatched, tengo 3 opciones: asignar alias a proveedor existente, crear nuevo proveedor, marcar como "gasto huérfano" (sin OdeC).
- CA-12.4: Los movimientos matcheados generan automáticamente registros en el estado de cuenta (`bank_transaction`).
- CA-12.5: Puedo filtrar la vista de conciliación por banco, rango de fechas y estado (matched/unmatched/huérfano).

---

#### HU-13: Seguimiento de Compras Diferidas (MSI)

**Como** administrador,
**quiero** registrar qué compras fueron diferidas a meses sin intereses o con intereses y dar seguimiento a los pagos mensuales,
**para** tener visibilidad del flujo de efectivo futuro y conciliar correctamente los abonos del banco.

**Criterios de Aceptación:**
- CA-13.1: Desde un movimiento bancario, puedo marcarlo como "Diferido" e indicar: tipo (MSI/interés), número de meses, tasa si aplica.
- CA-13.2: El sistema calcula automáticamente el pago mensual estimado.
- CA-13.3: Veo una tabla de diferimientos activos con: proveedor, monto total, meses pagados/restantes, monto pendiente, próximo pago.
- CA-13.4: Puedo registrar cada pago mensual contra el diferimiento, reduciendo el saldo pendiente.
- CA-13.5: El abono del banco (entrada) queda vinculado a los cargos diferidos (salidas), manteniendo la conciliación cuadrada.

---

### Épica 8: Productividad en Captura

#### HU-14: Escaneo de Código de Barras

**Como** comprador,
**quiero** escanear el código de barras de un producto al registrar un gasto,
**para** evitar buscar manualmente el producto en los catálogos y reducir errores de captura.

**Criterios de Aceptación:**
- CA-14.1: En el formulario de gasto, hay un botón "Escanear" junto al campo de código de barras.
- CA-14.2: Al escanear, si el código existe en el catálogo, se auto-completan: producto, tipo de producto y precio unitario.
- CA-14.3: Si el código no existe, se muestra mensaje claro y se permite registro manual.
- CA-14.4: El campo de código de barras también acepta entrada manual por teclado.

---

## 5. Requerimientos Funcionales

### 5.1 RF-01: Catálogo de Proveedores

| Campo | Requerido | Tipo | Validación |
|---|---|---|---|
| Código | Sí | string(20) | Único, autogenerado o manual |
| Nombre comercial | Sí | string(200) | No vacío |
| Razón social | No | string(200) | — |
| RFC | No | string(13) | Formato RFC si se ingresa |
| Tipo de proveedor | Sí | FK → supplierTypes | Debe existir en catálogo |
| Teléfono/Contacto | No | string(50) | — |
| Activo | Sí | boolean | Default: true |

**Endpoints requeridos:**
- `GET /suppliers` — Listar con filtros (nombre, código, activo, tipo)
- `GET /suppliers/:id` — Obtener uno con historial de compras
- `POST /suppliers` — Crear
- `PUT /suppliers/:id` — Actualizar
- `DELETE /suppliers/:id` — Soft-delete (desactivar)

---

### 5.2 RF-02: Catálogo de Bancos/Cuentas/Tarjetas

#### Institución Financiera

| Campo | Requerido | Tipo |
|---|---|---|
| Nombre | Sí | string(100) |
| Tipo | Sí | enum(checks, credit_card, cash, digital) |
| Límite global | No | decimal |
| Activa | Sí | boolean |

#### Cuenta Principal

| Campo | Requerido | Tipo |
|---|---|---|
| Institución | Sí | FK → institutions |
| Número de cuenta | Sí | string(50) — enmascarado en UI |
| Titular | Sí | string(200) |
| Fecha de corte | No | date |
| Día de pago | No | int(1-31) |
| Saldo inicial | No | decimal |
| Límite de crédito | No | decimal |
| Moneda | Sí | string(3), default: MXN |

#### Tarjeta/Subcuenta

| Campo | Requerido | Tipo |
|---|---|---|
| Cuenta principal | Sí | FK → bankAccounts |
| Titular/Usuario | Sí | string(200) |
| Terminación | Sí | string(4) — últimos 4 dígitos |
| Tipo | Sí | enum(physical, virtual, subaccount) |
| Límite individual | No | decimal |
| Activa | Sí | boolean |

**Endpoints requeridos:**
- `GET /institutions` — Listar instituciones
- `POST /institutions` — Crear
- `PUT /institutions/:id` — Actualizar
- `GET /bank-accounts` — Listar cuentas (con filtro por institución)
- `POST /bank-accounts` — Crear cuenta
- `PUT /bank-accounts/:id` — Actualizar
- `GET /cards` — Listar tarjetas (con filtro por cuenta)
- `POST /cards` — Crear tarjeta
- `PUT /cards/:id` — Actualizar

---

### 5.3 RF-03: Órdenes de Compra (OdeC)

| Campo | Requerido | Tipo |
|---|---|---|
| Folio | Sí | string(30) — autogenerado: `ODC-{id}` |
| Proveedor | Sí | FK → suppliers |
| Fecha creación | Sí | datetime — autogenerado |
| Estado | Sí | enum(pending, purchased, partial, closed, cancelled) |
| Total estimado | Sí | decimal — suma de líneas |
| Total real | No | decimal — se llena al registrar gasto |
| Banco/Cuenta/Tarjeta | No | FK → cards — se llena al pagar |
| Factura PDF | No | base64/file |
| Factura XML | No | text/xml |
| Comprobante de pago | No | base64/file |
| Evidencia física | No | base64/file |

**Endpoints requeridos:**
- `GET /purchase-orders` — Listar con filtros (proveedor, estado, fecha, OdeS)
- `GET /purchase-orders/:id` — Obtener con líneas, gastos y aplicaciones
- `POST /purchase-orders` — Crear
- `PUT /purchase-orders/:id` — Actualizar (solo en estado pendiente)
- `PATCH /purchase-orders/:id/status` — Cambiar estado

---

### 5.4 RF-04: Líneas de OdeC

| Campo | Requerido | Tipo |
|---|---|---|
| OdeC | Sí | FK → purchaseOrders |
| Tipo de línea | Sí | enum(product, non_traceable, consumable, tool, discount, operational) |
| Producto | No | FK → products |
| Descripción | Sí | string(300) |
| Código de barras | No | string(100) |
| Cantidad | No | decimal — requerido si es trazable |
| Costo unitario | No | decimal |
| Total línea | Sí | decimal — calculado o manual |
| Afecta refacciones | Sí | boolean |
| Afecta costo total | Sí | boolean |
| Método de asignación | Sí | enum(direct, manual, prorrateo_amount, prorrateo_quantity, equal_split, none) |

**Endpoints requeridos:**
- `GET /purchase-orders/:id/lines` — Listar líneas de una OdeC
- `POST /purchase-orders/:id/lines` — Agregar línea
- `PUT /purchase-order-lines/:id` — Actualizar línea
- `DELETE /purchase-order-lines/:id` — Eliminar línea (solo pendiente)

---

### 5.5 RF-05: Gastos

| Campo | Requerido | Tipo |
|---|---|---|
| OdeC | Sí | FK → purchaseOrders |
| Proveedor | Sí | FK → suppliers (autocompletado de OdeC) |
| Banco/Cuenta/Tarjeta | Sí | FK → cards |
| Fecha del gasto | Sí | datetime |
| Total del gasto | Sí | decimal |
| Folio/Ticket | Sí | string(50) |
| Factura PDF | No | base64 |
| Factura XML | No | text |
| Comprobante de pago | No | base64 |
| Evidencia física | No | base64 |

**Endpoints requeridos:**
- `GET /expenses` — Listar con filtros (OdeC, proveedor, banco, fecha)
- `GET /expenses/:id` — Obtener con archivos adjuntos
- `POST /expenses` — Crear (acepta archivos en base64)
- `DELETE /expenses/:id` — Cancelar (revierte acumulados y movimientos)

---

### 5.6 RF-06: Aplicación de Líneas a OdeS

| Campo | Requerido | Tipo |
|---|---|---|
| Línea de OdeC | Sí | FK → purchaseOrderLines |
| OdeS | Sí | FK → serviceOrders (vive en ERP) |
| Cantidad aplicada | No | decimal — para productos trazables |
| Importe aplicado | No | decimal — para conceptos no trazables |
| Método de asignación | Sí | enum — heredado de la línea |

**Endpoints requeridos:**
- `GET /purchase-order-lines/:lineId/applications` — Ver aplicaciones de una línea
- `POST /purchase-order-lines/:lineId/applications` — Aplicar línea a OdeS
- `PUT /line-applications/:id` — Modificar aplicación
- `DELETE /line-applications/:id` — Eliminar aplicación (revierte acumulados)

---

### 5.7 RF-07: Estado de Cuenta Interno

Cada gasto registrado genera automáticamente un movimiento (Transaction) en el banco/cuenta/tarjeta correspondiente.

| Campo | Descripción |
|---|---|
| Fecha | Fecha del movimiento |
| Hora | Hora de registro |
| Banco/Cuenta/Tarjeta | FK → cards |
| Tipo | entrada, salida, pago, ajuste |
| Proveedor/Concepto | Descripción |
| OdeC | FK → purchaseOrders |
| OdeS relacionadas | FK → serviceOrders (si aplica) |
| Entrada | decimal positivo |
| Salida | decimal negativo |
| Saldo | decimal — calculado |
| Registrado por | FK → users |

---

### 5.8 RF-08: Reportes

**Todos los reportes deben permitir selección de rango de fechas (desde / hasta).**

#### RF-08.1 — Reporte por OdeS (Rentabilidad por Servicio)

| Columna | Fuente |
|---|---|
| Número OdeS | OdeS.code |
| Cliente | customer.name |
| Vehículo | fleet_vehicle.plate_number |
| Cantidad a cobrar | OdeS.total (amountToCharge) |
| Refacciones aplicadas | SUM(line_application.applied_amount WHERE affects_parts=true) |
| **% Refacciones** | Refacciones / Cantidad a cobrar × 100 |
| Otros costos | SUM(line_application.applied_amount WHERE affects_parts=false) |
| Costo total | Refacciones + Otros costos |
| **% Costo total** | Costo total / Cantidad a cobrar × 100 |
| OdeC relacionadas | Lista de OdeC que surtieron esta OdeS |

**Filtros:** rango de fechas, % refacciones (todas / alerta >33% / críticas >100%)

#### RF-08.2 — Reporte por OdeC

| Columna | Fuente |
|---|---|
| Número OdeC | order.code + order.folio |
| Proveedor | supplier.name |
| Total compra | SUM(expense.amount_spent WHERE expense.order_id = OdeC.id) |
| Banco/Cuenta/Tarjeta | bank.name + bank.account |
| Estado | status.name |
| OdeS surtidas | Lista vía line_application |
| Factura/Ticket | Sí/No (¿tiene evidencia?) |

#### RF-08.3 — Reporte por Proveedor

| Columna | Fuente |
|---|---|
| Proveedor | supplier.name |
| Número de OdeC | COUNT(DISTINCT order.id) |
| **Total gastado** | SUM(expense.amount_spent) |
| OdeS surtidas | COUNT(DISTINCT line_application.service_order_id) |

**Filtro principal:** rango de fechas. Orden: por total gastado descendente.

#### RF-08.4 — Reporte por Banco/Cuenta/Tarjeta

| Columna | Fuente |
|---|---|
| Banco/Cuenta/Tarjeta | bank.name + últimos 4 dígitos |
| Total salidas | SUM(bank_transaction.amount WHERE type='out') |
| Total entradas | SUM(bank_transaction.amount WHERE type='in') |
| Saldo final | Calculado desde bank_transaction.balance |

**Filtro principal:** rango de fechas. Agrupado por banco individual y jerarquía (cuenta principal).

#### RF-08.5 — Reporte Mensual del 33% (Promedio Ponderado)

Este es el reporte principal para el dueño. Mide el % de gasto en refacciones entre todas las OdeS en el mes.

| Columna | Cálculo |
|---|---|
| Mes | Agrupado por mes/año |
| Total a cobrar | SUM(OdeS.total) en el mes |
| Total refacciones | SUM(line_application.applied_amount WHERE affects_parts=true) |
| **% Refacciones mensual** | Total refacciones / Total a cobrar × 100 (promedio ponderado) |
| Estado | 🟢 Correcto (≤32.99%) / 🟡 Límite (33.00%) / 🔴 Excedido (>33%) |

**¿Por qué promedio ponderado y no simple?**
```
Ejemplo:
  OdeS A: cobra $100,000, refacciones $50,000 → 50%
  OdeS B: cobra $5,000,   refacciones $500    → 10%
  
  Promedio simple: (50 + 10) / 2 = 30% ← engañoso
  Promedio ponderado: (50,000 + 500) / (100,000 + 5,000) = 48.1% ← real
```
El ponderado mide el impacto real en dinero, no promedios que esconden desbalances.

**Filtros:** por mes, por año. Vista de 12 meses con gráfica de tendencia.

---

### 5.9 RF-09: Aliases de Proveedores

Cada proveedor puede tener múltiples alias para facilitar la conciliación bancaria. Los estados de cuenta bancarios frecuentemente usan variaciones del nombre comercial según la terminal, sucursal o franquicia.

| Campo | Requerido | Tipo | Validación |
|---|---|---|---|
| Proveedor | Sí | FK → supplier | Proveedor padre |
| Alias | Sí | string(200) | No vacío, único por organización |
| Activo | Sí | boolean | Default: true |

**Ejemplo:**
```
Proveedor: AutoZone (id=5)
  ├── Alias: autozonemx
  ├── Alias: autozonemp
  ├── Alias: autzonemty
  └── Alias: AUTOZONE MERIDA
```

**Reglas:**
- Un alias no puede estar asociado a más de un proveedor (`UNIQUE(organization_id, alias)`)
- El matching en importación de Excel debe buscar primero por nombre exacto del proveedor, luego por alias
- Si no hay match, se ofrece crear un nuevo alias para un proveedor existente o registrar uno nuevo

**Endpoints requeridos:**
- `GET /suppliers/:id/aliases` — Listar aliases de un proveedor
- `POST /suppliers/:id/aliases` — Agregar alias
- `DELETE /suppliers/:id/aliases/:aliasId` — Eliminar alias

---

### 5.10 RF-10: Importación de Estados de Cuenta Bancarios (Excel)

El sistema debe permitir cargar archivos Excel de movimientos bancarios (estados de cuenta) y realizar matching automático contra proveedores, aliases y OdeCs registradas.

**Columnas esperadas del Excel bancario:**
| Columna | Descripción |
|---|---|
| Fecha | Fecha del movimiento |
| Descripción | Concepto/beneficiario (contiene nombre del proveedor) |
| Cargo | Monto de salida |
| Abono | Monto de entrada |
| Saldo | Saldo posterior |

**Proceso de matching automático:**
1. Por cada fila del Excel, buscar en `descripción` coincidencia con:
   - `supplier.name` (nombre exacto o contenido)
   - `supplier_alias.alias` (nombre alternativo)
   - `expense.amount_spent` = monto del movimiento (match por monto exacto contra OdeC)
2. Si hay match → vincular automáticamente a la OdeC y proveedor correspondientes
3. Si NO hay match → presentar interfaz de resolución:
   - **Opción A:** "Es alias de..." → seleccionar proveedor existente y crear alias automáticamente
   - **Opción B:** "Es proveedor nuevo..." → abrir formulario rápido de registro de proveedor
   - **Opción C:** "Gasto no registrado en OdeC" → marcar como gasto sin orden de compra (huérfano)
4. Los movimientos conciliados generan registros en `bank_transaction`

**Endpoints requeridos:**
- `POST /bank-reconciliation/upload` — Subir Excel, devuelve resultado de matching (matched, unmatched, partial)
- `GET /bank-reconciliation/unmatched` — Listar movimientos sin match pendientes de resolver
- `POST /bank-reconciliation/:id/resolve` — Resolver un movimiento no matcheado (asignar alias, crear proveedor, o marcar huérfano)

---

### 5.11 RF-11: Seguimiento de Compras Diferidas (MSI)

Cuando un movimiento bancario corresponde a una compra diferida a Meses Sin Intereses (MSI) o con intereses, el sistema debe permitir registrarlo para efectos de planeación financiera y conciliación.

**Campos del registro de diferimiento:**
| Campo | Requerido | Tipo | Descripción |
|---|---|---|---|
| Movimiento bancario | Sí | FK → bank_transaction | Movimiento origen |
| Tipo | Sí | enum(msi, interest) | Meses sin intereses o con intereses |
| Número de meses | Sí | int(1-48) | Plazo del diferimiento |
| Tasa de interés | Si aplica | decimal | Solo si `tipo = interest` |
| Monto total diferido | Sí | decimal | Suma total diferida |
| Pago mensual estimado | Sí | decimal | Calculado: `monto / meses` (MSI) o con interés compuesto |
| Fecha de inicio | Sí | date | Fecha del primer pago |
| Activo | Sí | boolean | Para seguimiento de pagos pendientes |

**Vista de seguimiento de diferidos:**
| Columna | Descripción |
|---|---|
| Proveedor / Concepto | Origen de la compra |
| Tipo | MSI o Con Interés |
| Monto total | Total diferido |
| Pagos realizados | # de meses pagados / total |
| Monto pagado | Suma de pagos realizados |
| Monto restante | Total - pagado |
| Próximo pago | Fecha estimada |
| Tasa | Si aplica |

**Reglas de conciliación:**
- Al diferir, el banco hace un **abono global** del total diferido o **abonos parciales** por cada compra. El sistema debe soportar ambos casos.
- Los pagos mensuales subsecuentes deben poder registrarse contra el diferimiento para llevar el seguimiento.
- Los diferimientos afectan el estado de cuenta: el abono del banco es una entrada que compensa las salidas de las compras diferidas.

**Endpoints requeridos:**
- `POST /deferred-payments` — Registrar un diferimiento
- `GET /deferred-payments` — Listar diferimientos activos e históricos
- `POST /deferred-payments/:id/payments` — Registrar un pago mensual
- `GET /deferred-payments/summary` — Resumen: total diferido, total pendiente, próximos pagos

---

### 5.12 RF-12: Integración con Código de Barras

El formulario de gasto debe permitir escanear un código de barras usando la app existente para auto-completar producto, tipo de producto y precio.

**Flujo:**
1. Usuario hace clic en "Escanear código de barras"
2. Se invoca la app de escaneo (integración existente)
3. El código de barras escaneado se recibe en el campo
4. El sistema busca el producto por `barcode` en el catálogo de productos
5. Si encuentra → auto-completa: producto, tipo de producto, precio unitario
6. Si no encuentra → muestra mensaje "Producto no encontrado. Registrar manualmente."

**Campos adicionales en producto (`product` o `product_document`):**
| Campo | Tipo | Descripción |
|---|---|---|
| `barcode` / `upc` | varchar(100) | Código de barras o UPC del producto |

---

### 5.13 RF-13: Gastos Operativos sin OdeC

El sistema debe permitir registrar gastos que no están vinculados a una Orden de Compra (gastos operativos del taller). Para clasificarlos se reutiliza la tabla existente `purchase_category`.

| Campo | Requerido | Tipo | Descripción |
|---|---|---|---|
| Categoría | Sí | FK → purchase_category | Clasificación del gasto operativo |
| Banco/Cuenta/Tarjeta | Sí | FK → bank | Medio de pago |
| Proveedor | No | FK → supplier | Si el gasto tiene proveedor identificable |
| Monto | Sí | decimal | Importe del gasto |
| Folio/Ticket | No | varchar(30) | Referencia si existe |
| Fecha | Sí | datetime | Fecha del gasto |
| Evidencias | No | archivos | Ticket, factura, comprobante |

**Categorías pre-cargadas sugeridas (seed):**

| Código | Nombre |
|---|---|
| `PAPELERIA` | Papelería y oficina |
| `LIMPIEZA` | Limpieza e insumos |
| `SERVICIOS` | Servicios (luz, agua, internet, teléfono) |
| `RENTA` | Renta de local |
| `HERRAMIENTA` | Herramienta y equipo |
| `MANTENIMIENTO` | Mantenimiento de instalaciones |
| `COMBUSTIBLE` | Combustible y transporte |
| `CAPACITACION` | Capacitación |
| `VIATICOS` | Comidas y viáticos |
| `PUBLICIDAD` | Publicidad y marketing |
| `SEGUROS` | Seguros y fianzas |
| `IMPUESTOS` | Impuestos y licencias |
| `OTRO` | Otro |

**Endpoints requeridos:**
- `GET /purchase-categories` — Ya existe, listar categorías
- `POST /expenses` — Ya existe. Si `order_id` es NULL, se requiere `expense_category_id`

**Reglas:**
- Si `order_id` es NULL → `expense_category_id` es requerido
- Si `order_id` tiene valor → `expense_category_id` es opcional (la OdeC ya da contexto)
- Los gastos operativos SÍ generan `bank_transaction` (afectan el estado de cuenta)
- Los gastos operativos NO aplican al KPI del 33% (no hay OdeS asociada)

---

## 6. Requerimientos No Funcionales

### 6.1 Rendimiento

- **NFR-01:** El listado de OdeS (hasta 500 registros) debe cargar en < 2 segundos.
- **NFR-02:** El cálculo de rentabilidad (% refacciones) debe recalcularse en < 500ms tras registrar un gasto.
- **NFR-03:** Las imágenes de evidencia deben comprimirse antes de enviarse (máx. 1920px en lado mayor).

### 6.2 Seguridad

- **NFR-04:** Toda comunicación con la API debe usar HTTPS en producción.
- **NFR-05:** El token JWT debe almacenarse exclusivamente en sessionStorage (nunca localStorage).
- **NFR-06:** Los números de cuenta/tarjeta deben almacenarse encriptados en base de datos.
- **NFR-07:** Nunca almacenar tokens bancarios, NIPs, contraseñas ni claves dinámicas.
- **NFR-08:** Auto-logout por expiración de JWT con margen configurable (default: 1 minuto antes).

### 6.3 Usabilidad

- **NFR-09:** La aplicación debe ser responsive (móvil, tablet, desktop).
- **NFR-10:** Desde celular, debe poder usarse la cámara nativa para capturar tickets/comprobantes.
- **NFR-11:** Los nombres de proveedores y bancos deben mostrarse legibles, nunca como IDs.
- **NFR-12:** Los reportes deben tener fondo blanco o claro para legibilidad de impresión.

### 6.4 Disponibilidad

- **NFR-13:** La aplicación debe funcionar offline-safe: si se pierde conexión durante el registro de un gasto, los datos no deben perderse.
- **NFR-14:** La API debe responder con status codes HTTP estándar y mensajes de error descriptivos.

### 6.5 Mantenibilidad

- **NFR-15:** El frontend debe usar TypeScript estricto (`strict: true`).
- **NFR-16:** Los tipos de datos compartidos entre frontend y backend deben definirse en un solo lugar (paquete compartido o código generado).
- **NFR-17:** Cada endpoint del backend debe tener pruebas unitarias y de integración.

---

## 7. Contratos de API

### 7.1 Convenciones Generales

- **Base URL:** `{host}/api/v1`
- **Autenticación:** Bearer JWT en header `Authorization`
- **Content-Type:** `application/json`
- **Formato de fecha:** ISO 8601 (`2026-06-03T15:30:00Z`)
- **Moneda:** MXN por defecto, montos en centavos (integer) o decimal con 2 posiciones
- **Errores:** `{ "statusCode": 400, "message": "descripción", "error": "Bad Request" }`

### 7.2 Endpoints Nuevos Requeridos

```
# ── Suppliers ──
GET    /api/v1/suppliers?search=&isActive=&type=&page=&limit=
GET    /api/v1/suppliers/:id
POST   /api/v1/suppliers
PUT    /api/v1/suppliers/:id
DELETE /api/v1/suppliers/:id          # soft-delete

# ── Institutions ──
GET    /api/v1/institutions
POST   /api/v1/institutions
PUT    /api/v1/institutions/:id

# ── Bank Accounts ──
GET    /api/v1/bank-accounts?institutionId=
POST   /api/v1/bank-accounts
PUT    /api/v1/bank-accounts/:id

# ── Cards/Subaccounts ──
GET    /api/v1/cards?bankAccountId=
POST   /api/v1/cards
PUT    /api/v1/cards/:id

# ── Purchase Orders (OdeC) — READ-ONLY desde expenses, los crea el ERP ──
GET    /api/v1/purchase-orders?supplierId=&status=&dateFrom=&dateTo=&odeSId=
GET    /api/v1/purchase-orders/:id
PATCH  /api/v1/purchase-orders/:id/status        # Cambiar estado (ej: marcar surtida)

# ── Purchase Order Lines — READ-ONLY, las crea el ERP ──
GET    /api/v1/purchase-orders/:id/lines
PUT    /api/v1/purchase-order-lines/:id           # Solo para clasificar (line_type, affects_parts, etc.)

# ── Line Applications ──
GET    /api/v1/purchase-order-lines/:lineId/applications
POST   /api/v1/purchase-order-lines/:lineId/applications
PUT    /api/v1/line-applications/:id
DELETE /api/v1/line-applications/:id

# ── Expenses ──
GET    /api/v1/expenses?purchaseOrderId=&supplierId=&bankId=&dateFrom=&dateTo=
GET    /api/v1/expenses/:id
POST   /api/v1/expenses
DELETE /api/v1/expenses/:id

# ── Transactions (Estado de Cuenta) ──
GET    /api/v1/transactions?cardId=&bankAccountId=&dateFrom=&dateTo=&type=
GET    /api/v1/transactions/:id

# ── Reports ──
GET    /api/v1/reports/odes-profitability?dateFrom=&dateTo=&minPct=&maxPct=
GET    /api/v1/reports/odec-detail/:id
GET    /api/v1/reports/supplier-summary?dateFrom=&dateTo=
GET    /api/v1/reports/bank-statement?bankAccountId=&cardId=&dateFrom=&dateTo=
GET    /api/v1/reports/monthly-33?year=&month=

# ── Service Orders (desde ERP, read-only para este módulo) ──
GET    /api/v1/service-orders?status=authorized,scheduled&search=
GET    /api/v1/service-orders/:id
```

### 7.3 Endpoints Existentes a Modificar

| Endpoint actual | Cambio requerido |
|---|---|
| `GET /purchaseOrders/0` | Renombrar a `GET /api/v1/purchase-orders`. Agregar query params de filtro. |
| `POST /expenses` | Agregar campos: `supplierId` (autocompletar de OdeC), `receiptFile`, `evidenceFile`, `invoicePdf`, `invoiceXml`. Al crear, generar Transaction automáticamente. |
| `GET /banks/0` | Migrar a estructura jerárquica: institutions → bankAccounts → cards. |
| `GET /suppliers/0` | Agregar query params de búsqueda y filtro. |
| `POST /suppliers` | Ya existe, verificar que acepte todos los campos del RF-01. |

---

## 8. Definition of Done por Historia

### 8.1 DoD General (Aplica a Todas las HU)

- [ ] Código en TypeScript estricto compila sin errores.
- [ ] Pruebas unitarias para lógica de negocio (cálculos, validaciones).
- [ ] Pruebas de integración para endpoints nuevos/modificados.
- [ ] UI responsive probada en mobile (375px), tablet (768px) y desktop (1280px).
- [ ] Code review aprobado por otro desarrollador.
- [ ] Migraciones de base de datos reversibles (up/down).
- [ ] Sin regresiones en funcionalidad existente.
- [ ] Documentación de API actualizada (Swagger/OpenAPI).

### 8.2 DoD Específicos

#### HU-01: Proveedores

- [ ] **Prueba 1:** Crear proveedor "Refaccionaria ABC" con RFC y ver que aparece en el listado.
- [ ] **Prueba 2:** Editar el RFC del proveedor y verificar persistencia.
- [ ] **Prueba 3:** Desactivar proveedor y verificar que no aparece en el selector de OdeC pero sí en reportes históricos.
- [ ] **Prueba 4:** Buscar "ABC" y ver que filtra correctamente.

#### HU-02: Bancos

- [ ] **Prueba 1:** Crear institución "Clara", cuenta principal con terminación 4467, tarjeta "Omar" con terminación 9126.
- [ ] **Prueba 2:** Verificar que la UI muestra "Clara ****9126".
- [ ] **Prueba 3:** Verificar que el número de cuenta NO aparece completo en ninguna vista.
- [ ] **Prueba 4:** Crear segunda tarjeta y verificar acumulados independientes.

#### HU-03: OdeC

- [ ] **Prueba 1:** Crear OdeC con proveedor "Refaccionaria ABC", agregar 2 líneas de productos y 1 de flete.
- [ ] **Prueba 2:** Verificar que el total estimado es la suma correcta de las 3 líneas.
- [ ] **Prueba 3:** Cambiar estado a "comprada" y verificar que las líneas ya no son editables.
- [ ] **Prueba 4:** Cancelar OdeC y verificar que el estado cambia a "cancelada".

#### HU-04: Gastos

- [ ] **Prueba 1:** Registrar gasto de $1,500 sobre OdeC 5001 con ticket (imagen) y factura PDF.
- [ ] **Prueba 2:** Verificar que se creó movimiento de salida por $1,500 en el estado de cuenta.
- [ ] **Prueba 3:** Verificar que los archivos adjuntos se pueden descargar/visualizar.
- [ ] **Prueba 4:** Cancelar gasto y verificar que el movimiento bancario se revierte.

#### HU-05: Aplicación a OdeS

- [ ] **Prueba 1:** OdeC con 6 bujías. Aplicar 2 a OdeS 1000 y 4 a OdeS 1001. Verificar totales.
- [ ] **Prueba 2:** Intentar aplicar 7 bujías (más de lo comprado). Debe rechazarse con error descriptivo.
- [ ] **Prueba 3:** Flete de $300 prorrateado por importe entre OdeS 1000 ($500 refacciones) y OdeS 1001 ($1,000 refacciones). Verificar: $100 a OdeS 1000, $200 a OdeS 1001.
- [ ] **Prueba 4:** Verificar que el flete NO afecta el % de refacciones, pero SÍ afecta el costo total.

#### HU-06: Rentabilidad por OdeS

- [ ] **Prueba 1:** OdeS 1000 con $3,000 a cobrar, $500 refacciones → 16.67% (verde).
- [ ] **Prueba 2:** OdeS 1001 con $5,000 a cobrar, $1,000 refacciones → 20.00% (verde).
- [ ] **Prueba 3:** Agregar $800 más en refacciones a OdeS 1001 ($1,800 total) → 36% (rojo, excedido).
- [ ] **Prueba 4:** Filtro "Alerta" muestra solo OdeS con >33%.

#### HU-07: Dashboard Mensual

- [ ] **Prueba 1:** Ver tabla con datos del mes actual. Suma de refacciones y total a cobrar correctos.
- [ ] **Prueba 2:** Promedio ponderado no es igual a promedio simple de porcentajes.
- [ ] **Prueba 3:** Semáforo mensual se actualiza al registrar/cancelar gastos.
- [ ] **Prueba 4:** Navegación drill-down del mes a las OdeS que lo componen.

#### HU-08: Estado de Cuenta

- [ ] **Prueba 1:** Ver lista de movimientos de "Clara ****9126" con saldo actualizado.
- [ ] **Prueba 2:** Filtrar por rango de fechas y verificar que solo muestra movimientos del período.
- [ ] **Prueba 3:** Saldo final = saldo inicial + entradas - salidas.
- [ ] **Prueba 4:** Ajuste manual de saldo requiere autorización y queda registrado en auditoría.

#### HU-09: Reportes

- [ ] **Prueba 1:** Exportar reporte de OdeS a PDF. Verificar que no tiene fondo de imagen que impida lectura.
- [ ] **Prueba 2:** Exportar reporte de proveedores a Excel con datos correctos.
- [ ] **Prueba 3:** Reporte por OdeC muestra todas las líneas y aplicaciones.
- [ ] **Prueba 4:** Reporte bancario muestra entradas, salidas y saldo correcto.

#### HU-10: Auditoría

- [ ] **Prueba 1:** Crear gasto y verificar que queda registro de usuario y timestamp.
- [ ] **Prueba 2:** Cancelar gasto y verificar que acumulados de OdeS y banco se revierten.
- [ ] **Prueba 3:** Modificar aplicación de línea y verificar registro de auditoría con valor anterior y nuevo.
- [ ] **Prueba 4:** Intentar aplicar más cantidad de la comprada → error + registro de advertencia.

---

#### HU-11: Aliases de Proveedores

- [ ] **Prueba 1:** Agregar alias "autozonemx" a proveedor "AutoZone". Buscar por "autozonemx" y encontrar el proveedor.
- [ ] **Prueba 2:** Intentar agregar "autozonemx" a otro proveedor → error de unicidad.
- [ ] **Prueba 3:** Eliminar alias sin afectar al proveedor.
- [ ] **Prueba 4:** Importar Excel con "AUTOZONE MERIDA" en descripción → hace match vía alias con proveedor "AutoZone".

#### HU-12: Importación de Estado de Cuenta Bancario (Excel)

- [ ] **Prueba 1:** Subir Excel con 20 movimientos: 15 con match (por nombre/alias/monto), 5 sin match. Verificar conteo.
- [ ] **Prueba 2:** Movimiento sin match → resolver como alias de proveedor existente. Verificar que se crea el alias.
- [ ] **Prueba 3:** Movimiento sin match → resolver como proveedor nuevo. Verificar que se crea supplier + alias.
- [ ] **Prueba 4:** Movimiento sin match → marcar como "gasto huérfano". Verificar que no se vincula a OdeC.
- [ ] **Prueba 5:** Movimientos matcheados generan `bank_transaction` automáticamente con balance actualizado.

#### HU-13: Seguimiento de Compras Diferidas (MSI)

- [ ] **Prueba 1:** Marcar movimiento de $12,000 como MSI a 12 meses. Verificar pago mensual = $1,000.
- [ ] **Prueba 2:** Marcar movimiento de $10,000 con interés del 18% a 6 meses. Verificar cálculo correcto de pago mensual.
- [ ] **Prueba 3:** Registrar 3 pagos mensuales de un diferido. Verificar: pagos restantes, monto pendiente, próximo pago.
- [ ] **Prueba 4:** El abono del banco por diferimiento queda vinculado a los cargos de las compras diferidas.

#### HU-14: Escaneo de Código de Barras

- [ ] **Prueba 1:** Escanear código de barras existente en catálogo → auto-completa producto, tipo, precio unitario.
- [ ] **Prueba 2:** Escanear código no existente → mensaje "Producto no encontrado".
- [ ] **Prueba 3:** Ingresar código manualmente (sin escáner) → misma búsqueda y auto-completado.
- [ ] **Prueba 4:** El campo de cantidad se enfoca automáticamente después del auto-completado para agilizar la captura.

---

## 9. Reglas de Negocio

| ID | Regla | Tipo | Implementación |
|---|---|---|---|
| **RB-001** | La OdeS pertenece a cliente/vehículo/trabajo y es la unidad de rentabilidad. | Estructural | FK de LineApplication a serviceOrders |
| **RB-002** | La OdeC pertenece a proveedor y agrupa piezas/conceptos de una o varias OdeS. | Estructural | FK de PurchaseOrder a Supplier |
| **RB-003** | Todo gasto del módulo debe asociarse a una OdeC. | Validación | Expense.purchaseOrderId NOT NULL |
| **RB-004** | Las líneas de una OdeC deben aplicarse a una o varias OdeS cuando correspondan. | Regla de negocio | Validación al cerrar OdeC |
| **RB-005** | Los productos trazables se aplican por cantidad. | UI + Validación | `lineType=product` → campo `quantity` requerido |
| **RB-006** | Los conceptos no trazables se aplican por importe. | UI + Validación | `lineType=non_traceable` → campo `amount` requerido |
| **RB-007** | No se puede aplicar más cantidad de producto que la comprada. | Validación | `SUM(applications.quantity) <= line.quantity` |
| **RB-008** | La cantidad a cobrar debe vivir en la OdeS, no en el gasto. | Estructural | Campo `amountToCharge` en ServiceOrder |
| **RB-009** | El % de refacciones debe calcularse sobre la OdeS. | Cálculo | `SUM(applications con affectsParts=true) / odeS.amountToCharge × 100` |
| **RB-010** | Solo las líneas marcadas como `affectsParts=true` entran al KPI del 33%. | Cálculo | WHERE clause en query de agregación |
| **RB-011** | El costo total debe calcularse separado del % de refacciones. | Cálculo | Dos métricas independientes en UI y DB |
| **RB-012** | Todo gasto debe tener proveedor, banco/cuenta/tarjeta, fecha y monto. | Validación | NOT NULL constraints |
| **RB-013** | Los nombres de proveedor y banco deben mostrarse legibles, no como IDs. | UI | JOIN en queries, preload en frontend |
| **RB-014** | El sistema debe acumular correctamente por OdeS, OdeC, proveedor, banco y tarjeta. | Cálculo | Aggregate queries con GROUP BY |
| **RB-015** | El sistema debe generar movimientos en el estado de cuenta interno al registrar gastos. | Trigger lógico | Insert en Transaction al crear Expense |
| **RB-016** | Los números de cuenta/tarjeta deben mostrarse enmascarados. | UI + Serialización | `****` + últimos 4 dígitos en DTO de salida |
| **RB-017** | No se deben guardar tokens bancarios ni credenciales sensibles. | Seguridad | No existen esos campos en el modelo |
| **RB-018** | Todo ajuste manual de saldo debe requerir permiso especial y quedar auditado. | Autorización | Role guard + AuditLog |
| **RB-019** | Toda aplicación de costo a OdeS debe ser auditable. | Auditoría | Trigger en LineApplication (created_by, updated_by) |
| **RB-020** | Un gasto cancelado o eliminado debe revertir sus acumulados y movimientos relacionados. | Consistencia | Soft-delete + recálculo de acumulados |
| **RB-021** | Un proveedor puede tener múltiples aliases, pero un alias solo pertenece a un proveedor. | Integridad | UNIQUE(organization_id, alias) en tabla `supplier_alias` |
| **RB-022** | Al importar un Excel bancario, el matching debe intentar en orden: nombre exacto → alias → monto exacto contra OdeC. | Proceso | Algoritmo de matching en `BankReconciliationService` |
| **RB-023** | Un movimiento bancario sin match después de agotar las opciones de resolución se marca como "gasto huérfano" (sin OdeC). | Proceso | Estado `orphan` en `bank_transaction` |
| **RB-024** | Los diferimientos (MSI) no afectan el saldo real del banco, pero sí afectan la proyección de flujo de efectivo. | Contabilidad | Tabla separada `deferred_payment`, no modifica `bank_transaction.balance` |
| **RB-025** | El ticket/folio de un gasto debe ser único por organización. Si se intenta duplicar, el sistema debe rechazar con mensaje descriptivo. | Validación | Catch unique constraint violation → 409 Conflict con mensaje claro |
| **RB-026** | Una línea de OdeC (`order_detail`) que ya tiene un gasto registrado no debe aparecer como disponible para nuevo gasto. | UI | Filtrar líneas con expense asociado en el dropdown |
| **RB-027** | El monto del gasto debe pre-calcularse automáticamente como `cantidad × precio_unitario` de la línea seleccionada, pero el usuario puede sobrescribirlo. | UI | Auto-cálculo en frontend al seleccionar línea |

---

## 10. Análisis de Brecha (Gap Analysis)

### 10.1 Lo que Existe en Base de Datos (PostgreSQL)

| Tabla | Estado | Relevancia para el módulo |
|---|---|---|
| `order` | Existente | Tabla polimórfica: OdeS (`order_type.class='service'`) y OdeC (`order_type.class='purchase'`). Ya tiene `supplier_id`, `folio`, `total`, `fleet_vehicle_id`, `customer_id`. |
| `order_detail` | Existente | Líneas de cualquier orden. Ya tiene `product_id`, `product_type_id`, `quantity`, `unit_cost`, `unit_price`, `total_price`, `supplier_id`. Faltan columnas de clasificación. |
| `expense` | Existente | Gastos con archivos (base64). FKs a `order`, `bank`, `supplier`. Ya funciona POST. Falta `order_detail_id` en DB (sí en entidad NestJS). |
| `bank` | Existente | Jerarquía auto-referenciada (`parent_id`). Ya soporta la estructura institución→cuenta→tarjeta. |
| `supplier` | Existente | Con `code`, `name`, `is_active`, `supplier_type_id`. |
| `supplier_type` | Existente | Catálogo de tipos de proveedor. |
| `fleet_vehicle` | Existente | Vehículo con `plate_number`, vinculado a `vehicle` y `fleet`. |
| `customer` | Existente | Clientes del ERP. |
| `transaction` | Existente | **NO sirve** para estado de cuenta bancario (es para inventarios: `input`/`output` de productos en ubicaciones). |
| `transaction_detail` | Existente | Detalle de inventario con `location_id`, `product_id`, `input`, `output`. |

### 10.2 Lo que Existe en Backend (NestJS)

| Módulo | Estado | Endpoints |
|---|---|---|
| `orders` | Funcional | `GET /orders/:id?classes='purchase'`, `POST`, `PATCH`, `DELETE`, `toPurchaseOrder` |
| `expenses` | Funcional básico | Solo `POST /expenses`. Falta GET, DELETE, filtros. |
| `banks` | Solo lectura | Solo `GET /banks/:id`. Falta POST, PUT. |
| `suppliers` | Parcial | `GET`, `POST` funcionando. Usa `suppliers/0` para listar. |
| `products` / `productTypes` | Funcional | `GET` con filtros. |
| `supplierTypes` | Funcional | `GET` disponible. |
| `purchaseCategories` | Existente | Catálogo de categorías de compra. |

### 10.3 Lo que Existe en Frontend (React)

| Componente | Estado | Observaciones |
|---|---|---|
| `LoginPage` | Funcional | JWT, sessionStorage, auto-logout |
| `OdesPage` | UI lista | Muestra purchaseOrders. Bien diseñada pero nomenclatura incorrecta. |
| `OdesDetailPage` | UI lista | Detalle con gastos y cálculos. |
| `GastosPage` | Funcional | Crea expense contra order. Flujo OdeC→líneas→aplicación ausente. |
| `BancosPage` | Solo lectura | CRUD requiere endpoints backend. |
| `ProveedoresPage` | Solo lectura | CRUD requiere endpoints backend. |
| `ReportesPage` | Placeholder | No implementado. |
| `api.ts` | Bien estructurado | Modela `OC` mezclando conceptos OdeS/OdeC. Necesita separación. |

### 10.4 Lo que NO Existe y Debe Crearse

| Artefacto | Tipo | Prioridad | Esfuerzo |
|---|---|---|---|
| `line_application` | Tabla nueva (migración) | **Crítico** | 1-2 días |
| `bank_transaction` | Tabla nueva (migración) | **Crítico** | 1-2 días |
| Columnas en `order_detail` (`line_type`, `affects_parts`, `affects_total_cost`, `assignment_method`) | ALTER TABLE | **Crítico** | 1 día |
| Columna `order_detail_id` en `expense` | ALTER TABLE | Alto | 0.5 día |
| CRUD backend para `bank` (POST, PUT) | NestJS controller | **Crítico** | 2-3 días |
| CRUD backend para `supplier` (PUT, DELETE soft) | NestJS controller | Alto | 1-2 días |
| Endpoints `line_application` | NestJS module | **Crítico** | 3-5 días |
| Endpoints `bank_transaction` | NestJS module | Alto | 3-5 días |
| Lógica de recálculo de acumulados | NestJS service | **Crítico** | 3-5 días |
| Reportes (OdeS, OdeC, proveedor, banco, mensual 33%) | NestJS + React | Alto | 8-12 días |
| UI: Clasificación de líneas de OdeC (tipo, affects_parts) | React | **Crítico** | 3-5 días |
| UI: Aplicación de líneas a OdeS | React | **Crítico** | 5-8 días |
| UI: Dashboard 33% mensual | React | Alto | 3-5 días |
| UI: Estado de cuenta interno | React | Alto | 3-5 días |
| UI: Reportes exportables | React | Alto | 5-8 días |

### 10.5 Bugs y Correcciones Necesarias

| ID | Problema | Severidad | Causa raíz | Solución |
|---|---|---|---|---|
| **BUG-CG-001** | Gasto → order directo, sin OdeC→líneas→aplicación | Crítico | El modelo actual salta la capa de `order_detail` y `line_application` | Crear tabla `line_application`. Refactorizar `GastosPage` y `POST /expenses`. |
| **BUG-CG-002** | No acumula gastos por OdeS | Crítico | Sin `line_application`, no hay trazabilidad de qué OdeS recibe qué costo | Implementar aggregates sobre `line_application` GROUP BY `service_order_id`. |
| **BUG-CG-003** | No acumula por banco/cuenta/tarjeta | Alto | No existe `bank_transaction`. El `transaction` existente es de inventario. | Crear tabla `bank_transaction`. Al crear expense, insertar movimiento automáticamente. |
| **BUG-CG-004** | No acumula por proveedor | Alto | Los reportes no hacen aggregate por `supplier_id` | Agregar queries de agregación en servicio de reportes. |
| **BUG-CG-005** | No recalcula % refacciones | Crítico | Sin `affects_parts` en `order_detail`, no se puede filtrar qué aplica al 33% | Agregar columnas de clasificación. Implementar servicio de recálculo. |
| **BUG-CG-006** | `amount_to_charge` se captura en expense en vez de leerse de OdeS | Crítico | El DTO `CreateExpenseDto` tiene `amountToCharge`. Debe eliminarse y leer de `order.total` (OdeS). | Eliminar campo del DTO. Leer `amountToCharge` desde OdeS (`order.total` donde `order_type.class='service'`). |
| **BUG-CG-007** | Muestra IDs en vez de nombres | Medio | El frontend no resuelve nombres antes de mostrar. | JOINs en queries backend. Cache de mapas (bankMap, supplierMap) en frontend. |
| **BUG-CG-008** | Gastos no ordenados por fecha/hora | Medio | Sin ORDER BY explícito en queries. | `ORDER BY expense.created DESC` en listados. |
| **BUG-CG-009** | Reportes con fondo de imagen | Medio | Fondo CSS heredado del tema general. | Eliminar `background-image` en componentes de reporte. Usar fondo blanco. |
| **BUG-CG-010** | Conceptos no trazables clasificados como refacciones | Alto | Sin `line_type` ni `affects_parts`, todo se trata como refacción. | Implementar columnas de clasificación en `order_detail`. Validar en UI. |
| **BUG-CG-011** | El monto del gasto no se auto-calculó de precio × cantidad | Alto | El frontend no multiplica `unitPrice * quantity` al seleccionar línea. Los campos `unitPrice` y `quantity` se llenan pero `amountSpent` queda vacío. | Efectuar el cálculo `setAmountSpent(quantity * unitPrice)` en el `useEffect` que ya precarga los datos de la línea (`orderDetailId`). |
| **BUG-CG-012** | Una línea ya gastada sigue apareciendo en el dropdown | Alto | El selector de línea no filtra las que ya tienen un expense registrado. | Agregar filtro: `items.filter(item => !expenses.some(e => e.orderDetailId === item.id))`. Si todos los items tienen gasto, deshabilitar el selector. |
| **BUG-CG-013** | Ticket/folio duplicado da error genérico | Medio | La constraint `UQ_expense_code` rechaza el duplicado pero el backend no devuelve mensaje descriptivo. El frontend muestra "Error" sin indicar la causa. | Backend: atrapar la excepción de unicidad y devolver `409 Conflict` con mensaje "El ticket/folio ya existe". Frontend: mostrar mensaje claro al usuario. |
| **BUG-CG-014** | El resumen de la orden (sidebar) no se actualiza al guardar un gasto | Alto | El estado `orders` no se refresca tras un `createExpense` exitoso. | Después de guardar, re-fetchear la orden con `getOrderByIdNormalized(orderId)` y actualizar `selectedOrder`. |
| **BUG-CG-015** | No se puede dar de alta un banco/cuenta desde la UI | Alto | Los endpoints `POST /banks` y `PUT /banks` no existen en el backend (solo GET). | Implementar CRUD completo en `BanksController`. Agregar campos en formulario: tipo, fecha de corte, día de pago, límite de crédito, saldo inicial. |
| **BUG-CG-016** | No se puede guardar un gasto sin OdeC (gasto operativo) | Medio | El campo `orderId` es requerido en el DTO y en el formulario. No existe opción "Sin orden de compra". | Hacer `orderId` opcional en backend y frontend. Agregar opción "Gasto general / Sin OdeC" en el selector. |
| **BUG-CG-017** | Falta campo de código de barras en el formulario de gasto | Bajo | No hay integración con la app de código de barras existente. | Agregar campo de texto + botón de escaneo que invoque la app de código de barras. Al escanear, buscar producto por barcode y auto-completar producto, tipo y precio. |
| **BUG-CG-018** | No se puede marcar OdeC como "surtida" al completar todos los gastos | Medio | No existe lógica para detectar que todos los items tienen gasto registrado y ofrecer cambio de estado. | Al guardar el último gasto faltante, sugerir al usuario cambiar estado a "surtida". Implementar endpoint `PATCH /purchase-orders/:id/status`. |

### 10.6 Preguntas de Diseño Pendientes

| ID | Pregunta | Impacto | Estado |
|---|---|---|---|
| **Q-001** | ¿Qué pasa si el usuario cambia el proveedor en un gasto que pertenece a una OdeC? ¿Se actualiza el proveedor de la OdeC o se permite un proveedor distinto por gasto? | Alto — afecta integridad de datos | Sin decidir |
| **Q-002** | ¿Se puede exceder el 100% del presupuesto sin autorización? ¿Qué rol se requiere para autorizar un excedente? | Alto — afecta regla de negocio RB-007 | Sin decidir |
| **Q-003** | ¿Los gastos operativos (sin OdeC) deben afectar el estado de cuenta bancario? | Medio — afecta `bank_transaction` | Sin decidir |

---

## 11. Plan de Implementación

### Fase 1: Estabilizar (Días 1-3)

- [ ] **Migraciones SQL:** 4 tablas nuevas + 2 ALTER TABLE
- [ ] **CRUD backend:** banks POST/PUT, suppliers PUT/DELETE + aliases
- [ ] **8 bugs CG-011 a CG-018** resueltos
- [ ] **RF-13:** seed purchase_category, formulario gasto operativo (sin OdeC)
- [ ] **Roles:** permisos Surtidor vs Gerente Compras en formulario

### Fase 2: Rentabilidad + Reportes (Días 4-8)

- [ ] CRUD `line_application` + clasificación `order_detail`
- [ ] Endpoints: rentabilidad por OdeS, dashboard mensual ponderado
- [ ] 5 reportes con filtro por fecha (RF-08.1 a RF-08.5)
- [ ] UI: semáforo 33%, dashboard 12 meses, ReportesPage
- [ ] Validación: `SUM(applied_quantity) <= quantity`

### Fase 3: Conciliación + Excel (Días 9-12)

- [ ] Trigger: expense → bank_transaction automático
- [ ] GET/DELETE /expenses con filtros y reversión
- [ ] RF-10: Importación Excel bancario + matching (nombre → alias → monto)
- [ ] RF-09: Aliases de proveedores (UI + backend)
- [ ] UI: Estado de cuenta interno + conciliación

### Fase 4: MSI + Cierre (Días 13-15)

- [ ] RF-11: deferred_payment, calendario pagos MSI
- [ ] RF-12: Escaneo código de barras
- [ ] PATCH /purchase-orders/:id/status
- [ ] Exportar PDF/Excel en reportes
- [ ] Bugs menores (nombres, orden, fondo)
- [ ] QA final

---

## 12. Criterios de Aceptación Final

El módulo se considera funcional únicamente si puede responder correctamente estas 12 preguntas:

| # | Pregunta | Fuente de datos |
|---|---|---|
| 1 | ¿Qué OdeC se generó por cada proveedor? | `purchase_orders` JOIN `suppliers` |
| 2 | ¿Qué productos/conceptos contiene cada OdeC? | `purchase_order_lines` |
| 3 | ¿Qué OdeS fueron surtidas por cada OdeC? | `line_applications` JOIN `service_orders` |
| 4 | ¿Cuánto costo de refacciones recibió cada OdeS? | Aggregate `line_applications` WHERE `affectsParts=true` |
| 5 | ¿Qué % de refacciones tiene cada OdeS vs cantidad a cobrar? | Cálculo: `refacciones / amountToCharge × 100` |
| 6 | ¿Qué otros costos recibió cada OdeS? | Aggregate WHERE `affectsParts=false AND affectsTotalCost=true` |
| 7 | ¿Cuál es el costo total de cada OdeS? | Suma de todas las aplicaciones |
| 8 | ¿Qué proveedor surtió cada pieza/concepto? | `line_applications` → `purchase_order_lines` → `purchase_orders` → `suppliers` |
| 9 | ¿Desde qué banco/cuenta/tarjeta se pagó? | `expenses` → `cards` → `bank_accounts` → `institutions` |
| 10 | ¿Qué saldo queda por usuario/tarjeta y global? | Aggregate `transactions` GROUP BY `card_id` / `bank_account_id` |
| 11 | ¿Qué movimientos aparecen en el estado de cuenta interno? | `transactions` ORDER BY date, time |
| 12 | ¿El promedio mensual de refacciones está por debajo, en límite o arriba del 33%? | Cálculo ponderado mensual con semáforo |

**Si alguna de estas preguntas no puede responderse con datos confiables, el módulo no está completo.**

---

## 13. Fuera del MVP / Fases Posteriores

| Función | Fase sugerida | Dependencia |
|---|---|---|
| OCR de tickets/facturas | P3 | Servicio externo de OCR |
| Lectura de PDF bancario | P3 | Parser de PDF |
| Validación automática SAT/CFDI | P3 | API del SAT o PAC |
| Conciliación automática con Excel/CSV | P2/P3 | Importación de estados de cuenta |
| Solicitud automática de factura al proveedor | P3 | Integración con email/API |
| Matching automático de movimientos bancarios | P3 | Algoritmo de reconciliación |
| App móvil nativa (iOS/Android) | P4 | React Native o PWA avanzada |
| Notificaciones push de excesos de gasto | P2 | Integración con servicio de push |
| Workflow de aprobación de OdeC | P2 | Motor de reglas de negocio |

---

## Historial de Versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 1.0 | 2026-06-03 | Original | Requerimiento funcional base |
| 1.1 | 2026-06-03 | Original | Correcciones y bugs detectados |
| 2.0 | 2026-06-03 | Arquitecto | Reestructuración como SRS formal: +historias de usuario, +contratos API, +modelo de dominio, +DoD por HU, +gap analysis, +plan de implementación, +NFRs |
| 2.1 | 2026-06-03 | Arquitecto | **Análisis de DB real (dump-postgres):** modelo de dominio alineado a esquema existente. `order` es tabla polimórfica (OdeS+OdeC), `order_detail` es líneas, `bank` es jerarquía auto-referenciada. Se precisan solo 2 tablas nuevas (`line_application`, `bank_transaction`) + 5 columnas en tablas existentes. Gap analysis actualizado. |
| 2.2 | 2026-06-04 | Arquitecto | **Feedback de funcionalidad real:** +8 bugs detectados en GastosPage (BUG-CG-011 a BUG-CG-018). +4 nuevos RFs: aliases de proveedores (RF-09), importación Excel bancario (RF-10), diferimientos MSI (RF-11), escaneo código de barras (RF-12). +4 nuevas HU (HU-11 a HU-14). +7 reglas de negocio (RB-021 a RB-027). +3 preguntas de diseño pendientes. Plan de fases reestructurado: OdeC las crea el ERP, expenses las consume. Eliminadas referencias a crear OdeC desde expenses. |

---

> **Nota:** Este documento reemplaza a `requerimiento_control_gastos_maramesa_v1_1.md` como la especificación canónica del módulo. El documento anterior se conserva como referencia histórica.
>
> **Fuentes del análisis:** `db/dump-postgres-202603102359.sql`, `db/Tablas.csv`, `db/Relaciones (foreign keys).csv`, `mar-erp-nodejs-api/src/` (entidades y controladores NestJS), `expenses/src/` (frontend React), sesión de revisión funcional 2026-06-04.
