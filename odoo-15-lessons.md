# Odoo 15 Lessons Learned

> **Plataforma**: Odoo.sh (Community/Enterprise)
> **Módulo de referencia**: `carga_internacional` — Freight Forwarding/Logística
> **Última actualización**: 2026-06-10

---

## Breaking Changes vs Odoo 14

### `sale_ids` no existe en `crm.lead` — usar `order_ids`

**Síntoma**: `AttributeError: 'crm.lead' object has no attribute 'sale_ids'`

**Causa**: En Odoo 15, el campo que relaciona `crm.lead` con `sale.order` se llama `order_ids`, no `sale_ids` como en versiones anteriores.

**Solución**:
```python
# ❌ INCORRECTO (Odoo 14 y anterior)
@api.depends('sale_ids.carga_id')
def _compute_carga_from_sale(self):
    for lead in self:
        for sale in lead.sale_ids:
            ...

# ✅ CORRECTO (Odoo 15)
@api.depends('order_ids.carga_id')
def _compute_carga_from_sale(self):
    for lead in self:
        for order in lead.order_ids:
            ...
```

**Verificar en shell**:
```python
lead = env['crm.lead'].browse(1)
print(lead.order_ids)   # ✓ funciona
print(lead.sale_ids)    # ✗ AttributeError
```

---

## Common Issues

### Issue 1: Campo `related` con `store=True` no se recalcula en asignación manual desde shell

**Síntoma**: Se asigna `lead.carga_id = carga` desde el shell, pero `lead.carga_name` (campo `related` de `carga_id.name`) queda vacío aunque el dato esté en BD.

**Causa raíz**: Al escribir directamente en BD desde el shell ORM, el mecanismo de `related` no dispara automáticamente el recalculo del campo derivado si el registro ya existía antes de la asignación.

**Solución**: Forzar el recalculo explícitamente después de la asignación masiva:
```python
leads_con_carga = env['crm.lead'].search([('carga_id', '!=', False)])
for lead in leads_con_carga:
    lead.carga_name = lead.carga_id.name
env.cr.commit()
```

**Alternativa limpia**: Correr `-u <modulo>` y luego el `_compute` para que el ORM recalcule todo correctamente.

---

### Issue 2: Vista kanban no se actualiza hasta hacer `-u` del módulo

**Síntoma**: Se agrega herencia de vista XML en el módulo, se hace commit al repo, pero la vista kanban en la UI no refleja los cambios. Los datos están correctos en BD.

**Causa raíz**: En Odoo.sh, el servidor no recarga automáticamente las vistas XML del módulo cuando hay cambios en el repositorio sin un upgrade explícito.

**Solución en Odoo.sh** (no hay `python odoo-bin` directo):
1. Activar modo desarrollador
2. Ir a **Apps → buscar el módulo → Upgrade (Actualizar)**

**Nota**: En Odoo.sh el comando `python` no está en PATH. El binario es `~/odoo-bin` si se necesita por terminal.

---

### Issue 3: Campos `tracking=True` en submodelos sin `mail.thread`

**Síntoma**: Warning en logs al cargar el módulo:
```
Tracking field defined on model without mail.thread: carga.internacional.detalle
```

**Causa raíz**: Se define `tracking=True` en campos de submodelos pero esos modelos no heredan de `mail.thread`.

**Solución opción A** — Agregar `mail.thread`:
```python
class CargaInternacionalDetalle(models.Model):
    _name = 'carga.internacional.detalle'
    _inherit = ['mail.thread', 'mail.activity.mixin']
```

**Solución opción B** — Quitar `tracking` si no se necesita historial:
```python
campo = fields.Char(string='Campo', tracking=False)  # o simplemente omitir tracking
```

---

### Issue 4: Modelos sin `ir.model.access` — bloqueante en producción

**Síntoma**: Usuarios no-admin no pueden leer/escribir en ciertos modelos. Error: `Access Denied`.

**Causa raíz**: Se crearon modelos (`carga.internacional.gasto`, `carga.internacional.producto`) sin agregar registros en `security/ir.model.access.csv`.

**Solución** — agregar en `ir.model.access.csv`:
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_carga_gasto_user,carga.internacional.gasto user,model_carga_internacional_gasto,base.group_user,1,1,1,0
access_carga_producto_user,carga.internacional.producto user,model_carga_internacional_producto,base.group_user,1,1,1,0
```

**Regla**: Todo modelo nuevo debe tener al menos una regla de acceso antes de pasar a producción.

---

### Issue 5: Duplicate label warnings en `sale.order`

**Síntoma**: Warning en logs:
```
Duplicate label for field carga_naviera / carga_naviera_id in model sale.order
```

**Causa raíz**: Dos campos distintos tienen el mismo `string=` en la vista, o un campo fue renombrado pero el `string` visual quedó igual al de otro campo.

**Solución**: Diferenciar los strings o renombrar el campo:
```python
# ❌ Ambiguo
carga_naviera = fields.Char(string='Naviera')
carga_naviera_id = fields.Many2one('res.partner', string='Naviera')  # duplicado!

# ✅ Diferenciado
carga_naviera = fields.Char(string='Naviera (texto)')
carga_naviera_id = fields.Many2one('res.partner', string='Naviera')
```

---

## Poblar Datos Masivos — Patrones Útiles

### Recompute masivo de campo computed con `store=True`

Cuando se corrige un `@api.depends` y hay registros ya existentes que no se recomputaron:

```python
# Shell: forzar recompute de todos los registros
leads = env['crm.lead'].search([('order_ids.carga_id', '!=', False)])
print(f'Leads a recomputar: {len(leads)}')
for lead in leads:
    for order in lead.order_ids:
        if order.carga_id:
            lead.carga_id = order.carga_id
            break
env.cr.commit()
```

### Cruzar registros por código embebido en el nombre

Cuando los usuarios escribieron el código de referencia en el nombre del registro (ej. `SMI05310 // CLIENTE // RUTA`) y se necesita vincular con otro modelo:

```python
import re

registros = env['crm.lead'].search([('carga_id', '=', False)])
actualizados = 0

for rec in registros:
    # Buscar TODOS los códigos en el nombre, usar el primero que exista en BD
    codigos = re.findall(r'\b([A-Z]{1,3}\d{4,6})\b', rec.name or '')
    carga = None
    for codigo in codigos:
        carga = env['carga.internacional'].search([('name', '=', codigo)], limit=1)
        if not carga:
            # Intentar normalizar a 5 dígitos (SMI2429 → SMI02429)
            m = re.match(r'^([A-Z]{1,3})(\d+)$', codigo)
            if m:
                carga = env['carga.internacional'].search(
                    [('name', '=', m.group(1) + m.group(2).zfill(5))], limit=1
                )
        if carga:
            break

    if carga:
        rec.carga_id = carga.id
        actualizados += 1

env.cr.commit()
print(f'Actualizados: {actualizados}')
```

**Lección clave**: Los usuarios suelen embeber el código de referencia en el nombre del registro. Antes de crear migraciones complejas, verificar si se puede extraer con regex.

### Diagnóstico antes de poblar — clasificar por causa

Antes de cualquier poblar masivo, clasificar los registros vacíos:

```python
sin_dato = env['crm.lead'].search([('carga_id', '=', False)])

sin_orders = sin_dato.filtered(lambda l: not l.order_ids)
con_orders_vacias = sin_dato.filtered(
    lambda l: l.order_ids and not any(o.carga_id for o in l.order_ids)
)
con_orders_con_carga = sin_dato.filtered(
    lambda l: any(o.carga_id for o in l.order_ids)
)

print(f'Sin orders (normal): {len(sin_orders)}')
print(f'Orders sin carga_id (problema real): {len(con_orders_vacias)}')
print(f'Orders con carga_id (compute fallido): {len(con_orders_con_carga)}')
```

Esto evita correr scripts masivos en registros que legítimamente no tienen dato.

---

## Vista Kanban — Patrones

### Badge con fallback en kanban de CRM

Mostrar campo principal y, si está vacío, mostrar campo alternativo (ej. primera cotización):

```xml
<!-- Exponer ambos campos al template -->
<xpath expr="//field[@name='activity_ids']" position="after">
    <field name="carga_name"/>
    <field name="order_ids"/>
</xpath>

<!-- Badge con fallback -->
<xpath expr="//div[hasclass('oe_kanban_footer')]" position="before">
    <t t-if="record.carga_name.value">
        <div class="px8 pb4">
            <span class="badge badge-pill badge-info" title="File de Carga">
                <i class="fa fa-ship mr4"/>
                <t t-esc="record.carga_name.value"/>
            </span>
        </div>
    </t>
    <t t-elif="record.order_ids.value and record.order_ids.value.length">
        <div class="px8 pb4">
            <span class="badge badge-pill badge-warning" title="Cotización vinculada">
                <i class="fa fa-file-text-o mr4"/>
                <t t-esc="record.order_ids.value[0].display_name"/>
            </span>
        </div>
    </t>
</xpath>
```

**Referencia**: [`crm_lead_inherit_carga.xml`](https://github.com/odoolargotek/odessa/blob/staging/carga_internacional/views/crm_lead_inherit_carga.xml)

---

## Referencias

- [odessa/carga_internacional](https://github.com/odoolargotek/odessa/tree/staging/carga_internacional) — módulo de referencia
- [Odoo 15 CRM Source](https://github.com/odoo/odoo/tree/15.0/addons/crm) — código fuente oficial
- [ORM API Odoo 15](https://www.odoo.com/documentation/15.0/developer/reference/backend/orm.html)
