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
new_picking = wizard._create_return()
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
# MAL
ml.qty_done = 5.0

# BIEN (Odoo 18)
ml.quantity = 5.0
```

---

### 4. No se puede `action_cancel()` sobre pickings en estado `done`

**Problema:**
```
Operación no válida
No puede cancelar un movimiento de existencias que se haya configurado como 'Hecho'.
```

**Solución:** Forzar el estado con `write` directo:
```python
picking.move_ids.filtered(
    lambda m: m.state == 'done'
).write({'state': 'cancel'})
picking.write({'state': 'cancel'})
```

⚠️ Esto solo cambia el estado del documento. El stock (`stock.quant`) y la valoración (`stock.valuation.layer`) NO se revierten automáticamente.

---

### 5. Cómo revertir inventario sin devolución

```python
# 1. Cancelar asientos contables
account_moves = env['account.move'].search([
    ('stock_move_id', 'in', moves.ids),
    ('state', '!=', 'cancel'),
])
if account_moves:
    account_moves.filtered(lambda m: m.state == 'posted').button_draft()
    account_moves.button_cancel()

# 2. Eliminar SVL
svl = env['stock.valuation.layer'].search([('stock_move_id', 'in', moves.ids)])
svl.sudo().unlink()

# 3. Ajustar quants
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

| Método | Soporte |
|---|---|
| Precio estándar | ✅ Eliminar SVL basta |
| AVCO | ✅ Recalcular `standard_price` con SVLs restantes |
| FIFO | ❌ No soportado |

---

### 7. Recargar vista automáticamente tras cambio de estado

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

---

### 8. Vista XML desincronizada con el modelo Python

**Regla:** Siempre actualizar modelo y vistas en el **mismo commit** cuando se eliminan o renombran campos.

---

### 9. Forzar rebuild en Odoo.sh

Hacer commit en `.odoo_sh_deployment_trigger` actualizando fecha/motivo para disparar un nuevo build automáticamente.

---

### 10. Ajuste manual de quants vía shell (caso de emergencia)

**Escenario:** Se canceló un picking `done` forzando el estado con `write({'state': 'cancel'})` directamente (sin pasar por el módulo), por lo que los `stock.quant` no fueron ajustados. Los productos siguen mostrando existencias incorrectas en inventario.

**Solución:** Corregir los quants directamente desde el shell de Odoo.

#### Entrar al shell en servidor on-premise

```bash
sudo -u odoo /opt/odoo/odoo-server/venv/bin/python \
  /opt/odoo/odoo-server/odoo-bin \
  shell -c /etc/odoo.conf \
  -d guembe_prod \
  --no-http
```

#### Script de corrección

```python
# Buscar el picking por nombre
picking = env['stock.picking'].search([('name', '=', 'ASGEN/IN/00003')], limit=1)
print(picking.id, picking.state)

# Ajustar quants: restar la cantidad de cada move_line
for move in picking.move_ids:
    for ml in move.move_line_ids:
        quant = env['stock.quant'].search([
            ('product_id', '=', ml.product_id.id),
            ('location_id', '=', ml.location_dest_id.id),
            ('lot_id', '=', ml.lot_id.id if ml.lot_id else False),
        ], limit=1)
        if quant:
            nueva = quant.quantity - ml.quantity
            print(f"{ml.product_id.name}: {quant.quantity} -> {nueva}")
            quant.sudo().write({'quantity': nueva})

# Confirmar cambios
env.cr.commit()
print("Listo")
```

**Recomendaciones:**
- Revisar los prints antes de hacer `commit()`. Si algo se ve raro, salir con `Ctrl+D` sin commitear.
- Este script es seguro para pickings de **entrada** (IN). Para pickings de **salida** (OUT) la lógica se invierte: hay que **sumar** la cantidad de vuelta.
- No usar si el picking tiene SVLs o asientos contables pendientes de revertir.

---

## Resumen de cambios de API Odoo 18 relevantes para inventario

| Elemento | Odoo 17 y anteriores | Odoo 18 |
|---|---|---|
| Wizard devolución | `create_returns()` → dict | `_create_return()` → recordset |
| Cantidad hecha en move.line | `qty_done` | `quantity` |
| Cantidad reservada en move.line | `product_uom_qty` | `reserved_qty` |
| Cancelar picking done | `action_cancel()` (permitido) | Bloqueado — usar `write({'state': 'cancel'})` |
| Builtins en `safe_eval` | No disponibles | Igual — nunca disponibles |
