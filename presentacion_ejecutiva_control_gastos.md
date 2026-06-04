# Control de Gastos MARAMESA — Presentación Ejecutiva

**Fecha:** 2026-06-04
**Audiencia:** Dueño / Dirección General

---

## 1. Qué ya funciona hoy?

| Funcionalidad | Estado |
|---|---|
| El ERP ya manda las órdenes de compra (OdeC) al sistema | Listo |
| Se puede registrar un gasto con ticket, factura PDF y XML | Listo |
| Se puede crear proveedores | Listo |
| Se pueden ver los detalles de cada orden y su historial de gastos | Listo |
| Al seleccionar un producto de la orden, se auto-llena tipo, cantidad y precio | Listo |
| El sistema calcula cuánto se ha gastado vs el presupuesto | Listo |

---

## 2. ¿Qué falta construir?

### Lo urgente (próximos 3 meses)

**A. Arreglar bugs en el registro de gastos**
- Que el monto se calcule solo (precio × cantidad)
- Que una línea ya pagada no vuelva a aparecer
- Que no deje guardar tickets repetidos sin avisar claro
- Que al guardar se actualice el resumen en pantalla

**B. Poder dar de alta bancos y cuentas desde la pantalla** (hoy solo se ven)

**C. Semáforo 33%**
- Verde: gasto ≤ 32% del cobro
- Amarillo: justo en 33%
- Rojo: excedido

**D. Reportes básicos**
- Por orden de servicio: ¿cuánto costó vs cuánto se cobró?
- Por proveedor: ¿a quién le compramos más?
- Por banco: ¿cuánto salió de cada cuenta/tarjeta?

**E. Gastos operativos (sin orden de compra)**
- Poder registrar gastos del taller que no son de una orden específica: renta, luz, papelería, herramientas, etc.
- Se clasifican por categoría (ya existe el catálogo)

### Lo siguiente (3-6 meses)

**F. Aliases de proveedores**
- "AutoZone" en el sistema, pero en el banco aparece como "autozonemx" o "AUTOZONE MERIDA"
- Poder agregar todos los nombres con los que aparece un proveedor en los estados de cuenta

**G. Subir Excel del banco y que haga match solo**
- Cargas el Excel de movimientos del banco
- El sistema busca coincidencias por nombre del proveedor, alias o monto exacto
- Lo que no hace match te lo muestra para que tú decidas: ¿es alias de alguien? ¿es proveedor nuevo? ¿es un gasto no registrado?

**H. Registrar compras a meses (MSI)**
- Si algo se difirió a 6, 12, 18 meses, se registra
- El sistema calcula el pago mensual
- Muestra un calendario: cuánto falta, cuántos pagos van, próximo pago
- Ayuda a cuadrar cuando el banco hace abonos globales por lo diferido

**I. Escanear código de barras**
- Usando la app que ya tienen, escaneas el producto y se auto-llena

---

## 3. ¿Quién hace qué? (Roles)

Hay dos personas que tocan el sistema en el día a día:

| | 🛵 **Surtidor / Mensajero** | 📋 **Gerente de Compras** |
|---|---|---|
| **¿Quién es?** | El que va a la refaccionaria y recoge las piezas | El encargado de compras y proveedores |
| **¿Qué puede cambiar?** | Método de pago, monto total, evidencias, código de barras | Lo mismo que el surtidor **+ cambiar proveedor** |
| **¿Puede cambiar producto?** | Solo si la línea tiene alternativas (ej: no hay NGK, compra Bosch) | Sí |
| **¿Puede cambiar proveedor?** | ❌ No | ✅ Sí. Si lo cambia, el sistema busca OdeC activa con ese proveedor. Si no hay, crea una nueva. |

### ¿Qué pasa si el Gerente cambia el proveedor?

```
1. El gerente cambia "Refaccionaria ABC" → "Autozone"
2. El sistema busca: ¿hay una OdeC activa con proveedor Autozone?
   ├── Sí → vincula los productos a esa OdeC
   └── No  → crea una nueva OdeC con proveedor Autozone
3. El gasto queda registrado contra la OdeC correcta
```

---

## 4. ¿Cómo se va a ver?

### Flujo principal: Registrar un gasto

```
1. El ERP manda la orden de compra (ej: 6 bujías Denso, proveedor Refaccionaria ABC)

2. En la pantalla de gastos:
   ┌──────────────────────────────────────────────────────────┐
   │  Registrar Gasto                                         │
   │                                                          │
   │  OdeC: ODC-5021    Proveedor: Refaccionaria ABC          │
   │                                                          │
   │  Línea: [4 bujías Denso ▼]                               │
   │  Producto: Bujía Denso    Tipo: Refacciones              │
   │  Cantidad: 4    Precio U.: $250    Monto: $1,000 ✨       │
   │                                      ↑ cantidad × precio │
   │  ─────────────────────────────────────────────────────── │
   │  🛵 Método de pago: [Clara ****9126 ▼]   ← Surtidor edita│
   │  🔒 Proveedor: Refaccionaria ABC          ← Surtidor NO   │
   │     Solo Gerente de Compras puede cambiar proveedor       │
   │  Ticket/Folio: FAC-88421                                 │
   │                                                          │
   │  📎 Ticket: [factura.jpg]            ← Surtidor puede     │
   │  📎 Factura PDF: [factura.pdf]                           │
   │  📎 XML: [factura.xml]                                   │
   │  [ 📷 Escanear código de barras ]    ← Surtidor puede     │
   │                                                          │
   │  🟢 Alternativas: NGK, Bosch, Champion (si no hay Denso) │
   │                                                          │
   │                   [ 🚀 Guardar Gasto ]                   │
   └──────────────────────────────────────────────────────────┘

   🔒 = Solo Gerente de Compras
   🛵 = Surtidor puede editar

3. Al guardar:
   - El gasto se registra contra la línea de la OdeC
   - El estado de cuenta del banco se actualiza solo
   - La línea ya no aparece para volver a gastar
   - El resumen de la orden se refresca

4. Cuando todos los productos tienen gasto registrado:
   - El sistema sugiere marcar la orden como "Surtida"
```

### Dashboard de Rentabilidad

```
┌─────────────────────────────────────────────────────────┐
│  Rentabilidad - Junio 2026                              │
│                                                         │
│  Total cobrado: $500,000                                │
│  Total refacciones: $150,000                            │
│  % Refacciones: 30.0% 🟢 Correcto                      │
│                                                         │
│  ┌──────┬──────────┬──────────┬────────┬────────┐      │
│  │ OdeS │ Cliente  │ Cobrado  │ Refacc │   %    │      │
│  ├──────┼──────────┼──────────┼────────┼────────┤      │
│  │ 1000 │ Juan     │  $3,000  │  $500  │ 16.7%🟢│      │
│  │ 1001 │ María    │  $5,000  │ $1,800 │ 36.0%🔴│      │
│  │ 1002 │ Carlos   │  $8,000  │ $2,640 │ 33.0%🟡│      │
│  └──────┴──────────┴──────────┴────────┴────────┘      │
│                                                         │
│  Filtros: [Todas] [⚠️ Alerta >33%] [❌ Críticas >100%]  │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Reportes que va a tener el dueño

Todos los reportes funcionan **seleccionando un rango de fechas** (desde / hasta). Son 5:

### Reporte 1 — Lo gastado en cada servicio (OdeS)

Responde: **¿cuánto gasté en refacciones vs lo que le cobré al cliente?**

| OdeS | Cliente | Placa | Cobrado | Refacciones | % | Otros | Total | % Total |
|---|---|---|---|---|---|---|---|---|
| 1000 | Juan | ABC-123 | $3,000 | $500 | 16.7% 🟢 | $100 | $600 | 20% |
| 1001 | María | XYZ-789 | $5,000 | $1,800 | 36.0% 🔴 | $200 | $2,000 | 40% |

### Reporte 2 — Lo gastado con cada Proveedor

Responde: **¿a quién le estoy comprando más?**

| Proveedor | Órdenes | Total gastado |
|---|---|---|
| Refaccionaria ABC | 5 | $45,000 |
| Autozone | 3 | $28,500 |
| Fletes Express | 4 | $6,200 |

### Reporte 3 — Lo gastado en cada Cuenta de Banco

Responde: **¿cuánto salió de cada tarjeta/cuenta?**

| Banco/Cuenta | Salidas | Entradas | Saldo |
|---|---|---|---|
| Clara ****9126 (Omar) | $52,000 | $0 | $48,000 |
| Afirme ****3344 (Laura) | $18,500 | $0 | $81,500 |
| Efectivo Caja | $9,200 | $5,000 | $15,800 |

### Reporte 4 — Promedio Mensual del % de Refacciones (EL MÁS IMPORTANTE)

Responde: **de todo lo que cobré en el mes, ¿qué % se fue en refacciones?**

```
┌────────┬────────────┬────────────┬──────┬────────┐
│  Mes   │  Cobrado   │ Refacciones│  %   │ Estado │
├────────┼────────────┼────────────┼──────┼────────┤
│ Enero  │  $480,000  │  $144,000  │30.0% │   🟢   │
│ Feb    │  $520,000  │  $171,600  │33.0% │   🟡   │
│ Marzo  │  $450,000  │  $153,000  │34.0% │   🔴   │
│ Abril  │  $500,000  │  $155,000  │31.0% │   🟢   │
│ Mayo   │  $510,000  │  $147,900  │29.0% │   🟢   │
│ Junio  │  $500,000  │  $150,000  │30.0% │   🟢   │
└────────┴────────────┴────────────┴──────┴────────┘
```

> El % se calcula con **promedio ponderado**: divide el total de refacciones entre el total cobrado. Esto mide el impacto real en dinero. Un promedio simple puede esconder que un servicio chico tuvo mucho % de gasto pero no afecta realmente, o al revés.

### Reporte 5 — Detalle de cada OdeC

Responde: **¿qué se compró, a quién, con qué se pagó?**

| OdeC | Proveedor | Total | Pagado con | Estado | OdeS | Factura |
|---|---|---|---|---|---|---|
| ODC-5021 | Refacc. ABC | $15,000 | Clara ****9126 | Surtida | 1000, 1001 | Sí |
| ODC-5022 | Autozone | $8,500 | Afirme ****3344 | Parcial | 1003 | Sí |

---

## 6. Fases y tiempos estimados

> Desarrollo con IA full-time. Una tarea = horas, no días.

| Fase | ¿Qué incluye? | Días |
|---|---|---|
| **1. Estabilizar** | 8 bugs, CRUD bancos, gastos operativos, migraciones | 2-3 |
| **2. Rentabilidad + Reportes** | `line_application`, clasificación líneas, semáforo 33%, 5 reportes con filtro fecha | 4-5 |
| **3. Conciliación** | Aliases, importar Excel bancario, matching, `bank_transaction`, estado de cuenta | 3-4 |
| **4. MSI + Cierre** | Diferidos, calendario pagos, código barras, PDF/Excel, QA | 2-3 |

| | Tiempo |
|---|---|
| **Total Expenses** | **11-15 días hábiles (2-3 semanas)** |
| **Total ERP** (documento aparte) | **2-3 días** |
| **Gran total** | **13-18 días hábiles (3-4 semanas)** |

---

### Detalle de las 2-3 semanas

| Semana | ¿Qué se entrega? |
|---|---|
| **1** | Bugs resueltos. Alta de bancos funcionando. Gastos operativos con categorías. Tablas nuevas creadas. |
| **2** | Semáforo 33% funcionando. 5 reportes con filtro por fecha. Líneas se aplican a OdeS. |
| **2-3** | Excel del banco se importa y hace match solo. Aliases de proveedores. Estado de cuenta interno. |
| **3** | MSI con calendario de pagos. Código de barras. Exportar PDF/Excel. QA final. |

---

## 7. Decisiones que necesitamos del dueño

| # | Pregunta | Opciones |
|---|---|---|
| 1 | **Si un gasto rebasa el 100% del presupuesto de la orden, ¿se bloquea o solo avisa?** | A) Bloquear — requiere autorización de admin. B) Solo advertir — se guarda con bandera roja. |
| 2 | **¿Qué tan atrás en el tiempo necesitan cargar estados de cuenta?** | A) Solo del mes actual. B) Últimos 3 meses. C) Histórico completo. |

> **Nota:** El cambio de proveedor en un gasto ya está definido: solo el Gerente de Compras puede hacerlo. Si cambia, el sistema busca OdeC activa del nuevo proveedor o crea una.

---

## 8. ¿Qué necesita el ERP?

Estos cambios son aparte, en el sistema ERP, para que expenses funcione completo:

| # | Cambio en ERP | ¿Para qué sirve? |
|---|---|---|
| 1 | **Catálogo de bancos** (pantalla CRUD) | Dar de alta instituciones, cuentas y tarjetas. **Asignar usuario a cada tarjeta** para pre-selección y trazabilidad |
| 2 | **Seleccionar banco en la OdeC** | Que al crear una orden de compra se elija con qué cuenta se paga. Al abrir el gasto, se pre-selecciona la del usuario |
| 3 | **OdeC para Stock** | Crear órdenes de compra para inventario, no ligadas a un servicio |
| 4 | **Unificar/separar líneas por alternativas** | Si dos vehículos comparten las mismas alternativas de refacción → una línea. Si no → líneas separadas |
| 5 | **Catálogo de categorías de gasto** | Clasificar gastos operativos (renta, luz, papelería...) en árbol jerárquico sin límite de niveles |

⏱ **2.5-4 días adicionales en el ERP** (independiente de los 11-15 días de expenses).


