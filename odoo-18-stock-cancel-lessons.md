# Odoo 18 — Cancelar Transferencias de Inventario (stock.picking)

**Fecha:** 2026-05-26  
**Autor:** Largotek SRL  
**Módulo relacionado:** `lt_stock_cancel_reverse` (largotekodoo/staging)  
**Versión:** 18.0

---

## Contexto

Se necesitaba una forma de cancelar transferencias de inventario ya validadas (`state = done`) de forma masiva, eliminando su efecto en existencias y costeo, sin generar documentos de devolución.

---

## Lecciones aprendidas

### 1. Acciones de servidor: `safe_eval` no expone builtins

**Problema:** Al usar `hasattr()` o `getattr()` en el código Python de una acción de servidor (`ir.actions.server`), Odoo lanza:
```
NameError: name 'hasattr' is not defined
NameError: name 'getattr' is not defined
```

**Causa:** El entorno `safe_eval` de Odoo restringe los builtins disponibles. Solo están disponibles las variables documentadas en el bloque de ayuda: `env`, `records`, `model`, `time`, `datetime`, `log`, `UserError`, `Command`, etc.

**Solución:** Nunca usar `hasattr`, `getattr`, `isinstance` ni otros builtins en acciones de servidor. Llamar directamente el método si se sabe que existe en el modelo estándar:
```python
# MAL
if hasattr(picking, 'button_cancel'):
    picking.button_cancel()

# BIEN
picking.button_cancel()
```

---

### 2. `stock.return.picking`: método renombrado en Odoo 18

**Problema:** En versiones anteriores el wizard de devolución exponía `create_returns()`. En Odoo 18 fue renombrado:
```
AttributeError: 'stock.return.picking' object has no attribute 'create_returns'. Did you mean: '_create_return'?
```

**Causa:** Refactoring interno de Odoo 18.

**Diferencia clave:**

| Versión | Método | Retorna |
|---|---|---|
| Odoo 15/16/17 | `create_returns()` | `dict` con `{'res_id': picking_id, ...}` |
| **Odoo 18** | `_create_return()` | Recordset `stock.picking` directamente |

**Solución Odoo 18:**
```python
# MAL (Odoo 15/16/17)
res = wizard.create_returns()
new_picking = env['stock.picking'].browse(res['res_id'])

# BIEN (Odoo 18)
new_picking = wizard._create_return()  # devuelve el picking directamente
```

---

### 3. `stock.move.line`: campo `qty_done` eliminado en Odoo 18

**Problema:**
```
AttributeError: 'stock.move.line' object has no attribute 'qty_done'
```

**Causa:** En Odoo 18, `stock.move.line` unificó los campos de cantidad:

| Campo | Odoo 15/16/17 | Odoo 18 |
|---|---|---|
| Cantidad hecha | `qty_done` | `quantity` |
| Cantidad reservada | `product_uom_qty` | `reserved_qty` |

**Solución:**
```python
# MAL (Odoo 17 y anteriores)
ml.qty_done = 5.0
if ml.qty_done == 0: ...

# BIEN (Odoo 18)
ml.quantity = 5.0
if ml.quantity == 0: ...
```

---

### 4. No se puede `action_cancel()` sobre pickings en estado `done`

**Problema:**
```
Operación no válida
No puede cancelar un movimiento de existencias que se haya configurado como 'Hecho'.
Cree una devolución para revertir los movimientos que se realizaron.
```

**Causa:** Odoo 18 bloquea explícitamente `action_cancel()` en pickings `done` para proteger la integridad del inventario.

**Solución:** Si se quiere forzar el estado (aceptando la responsabilidad del impacto en inventario), usar `write` directo sobre el picking y sus moves:
```python
picking.move_ids.filtered(
    lambda m: m.state == 'done'
).write({'state': 'cancel'})
picking.write({'state': 'cancel'})
```

⚠️ **Importante:** Esto solo cambia el estado del documento. El stock (`stock.quant`) y la valoración (`stock.valuation.layer`) NO se revierten automáticamente. Hay que manejarlos por separado.

---

### 5. Cómo revertir inventario sin devolución (eliminación directa)

Cuando se quiere cancelar un picking `done` sin crear documentos de devolución, el flujo completo es:

```python
# 1. Cancelar asientos contables vinculados
account_moves = env['account.move'].search([
    ('stock_move_id', 'in', moves.ids),
    ('state', '!=', 'cancel'),
])
if account_moves:
    account_moves.filtered(lambda m: m.state == 'posted').button_draft()
    account_moves.button_cancel()

# 2. Eliminar stock.valuation.layer (SVL)
svl = env['stock.valuation.layer'].search([('stock_move_id', 'in', moves.ids)])
svl.sudo().unlink()

# 3. Ajustar stock.quant (restar lo que movió cada línea)
for move in moves:
    for ml in move.move_line_ids:
        quant = env['stock.quant'].search([
            ('product_id', '=', ml.product_id.id),
            ('location_id', '=', ml.location_dest_id.id),
            ('lot_id', '=', ml.lot_id.id if ml.lot_id else False),
        ], limit=1)
        if quant:
            quant.sudo().write({'quantity': quant.quantity - ml.quantity})

# 4. Cancelar moves y picking
picking.move_line_ids.sudo().write({'state': 'cancel'})
moves.sudo().write({'state': 'cancel'})
picking.sudo().write({'state': 'cancel'})
```

---

### 6. Manejo de costeo AVCO al eliminar SVL

Cuando se eliminan SVLs de un producto con método **precio promedio (AVCO)**, hay que recalcular el `standard_price` del producto con los SVLs restantes:

```python
def revertir_avco(product, svl_a_eliminar):
    if product.categ_id.property_cost_method != 'average_price':
        return
    remaining_svl = env['stock.valuation.layer'].search([
        ('product_id', '=', product.id),
        ('id', 'not in', svl_a_eliminar.ids),
    ])
    total_qty = sum(remaining_svl.mapped('quantity'))
    total_value = sum(remaining_svl.mapped('value'))
    if total_qty > 0:
        product.sudo().write({'standard_price': total_value / total_qty})
```

**Métodos de costeo soportados con este enfoque:**
- ✅ Precio estándar (`standard`) — el SVL no afecta `standard_price`, con eliminarlo basta.
- ✅ Precio promedio (`average_price`) — recalcular `standard_price` con SVLs restantes.
- ❌ FIFO (`fifo`) — NO soportado. Las capas FIFO están encadenadas y revertirlas sin romper la secuencia requiere tratamiento especializado.

---

### 7. Recargar vista automáticamente después de una acción de botón

**Problema:** Un botón que cambia el estado de un registro retorna `display_notification` (toast) pero el formulario no se actualiza. El usuario debe hacer F5.

**Solución:** Retornar una acción `ir.actions.act_window` que apunte al mismo registro:

```python
return {
    'type': 'ir.actions.act_window',
    'res_model': 'stock.picking',
    'res_id': self[0].id if len(self) == 1 else False,
    'view_mode': 'form' if len(self) == 1 else 'list',
    'views': [(False, 'form')] if len(self) == 1 else [(False, 'list')],
    'target': 'current',
}
```

Esto recarga el formulario (o la lista en caso masivo) mostrando el nuevo estado inmediatamente.

---

### 8. Vista XML desincronizada con el modelo Python

**Problema:** Al refactorizar un modelo y eliminar campos, las vistas XML heredadas que los referencian siguen causando errores en la instalación/actualización:
```
odoo.tools.convert.ParseError: El campo "lt_reversed_picking_id" no existe en el modelo "stock.picking"
```

**Causa:** El campo fue eliminado del modelo Python pero la vista XML no fue actualizada en el mismo commit.

**Regla:** Siempre actualizar modelo y vistas en el **mismo commit** cuando se eliminan o renombran campos. Nunca dejarlos desincronizados.

---

### 9. Forzar rebuild en Odoo.sh cuando el error persiste

Si se corrige el código pero el error sigue apareciendo al actualizar el módulo en Odoo.sh, puede ser que el servidor esté corriendo un build anterior. Para forzar un nuevo rebuild:

- Hacer un commit en `.odoo_sh_deployment_trigger` actualizando la fecha/motivo.
- Esto dispara un nuevo build automáticamente con el último código.

---

## Resumen de cambios de API Odoo 18 relevantes para inventario

| Elemento | Odoo 17 y anteriores | Odoo 18 |
|---|---|---|
| Wizard devolución | `create_returns()` → dict | `_create_return()` → recordset |
| Cantidad hecha en move.line | `qty_done` | `quantity` |
| Cantidad reservada en move.line | `product_uom_qty` | `reserved_qty` |
| Cancelar picking done | `action_cancel()` (permitido) | Bloqueado — usar `write({'state': 'cancel'})` |
| Builtins en `safe_eval` | `hasattr`, `getattr` no disponibles | Igual — nunca disponibles |
