# ⚡ Odoo Quick Fixes & Patches

**Purpose**: Fast solutions for common Odoo issues that don't require deep refactoring  
**Use Case**: When you need to fix something quickly in development or production  
**Last Updated**: 2026-05-12

---

## 🎯 Navigation

- [Module Issues](#-module-issues)
- [Database Issues](#-database-issues)
- [View & UI Issues](#-view--ui-issues)
- [Performance Quick Wins](#-performance-quick-wins)
- [Security Fixes](#-security-fixes)
- [API & Integration](#-api--integration)

---

## 📦 Module Issues

### ⚠️ Campo de modelo no encontrado en vista — archivo Python no importado en `__init__.py`

**Contexto**: `lt_optica_base` — Odoo.sh staging, 2026-05-12  
**Error**: `ParseError: El campo "is_optical_client" no existe en el modelo "res.partner"`

**Causa raíz**:  
El archivo `models/res_partner.py` existía con el campo definido correctamente, pero **nunca fue importado** en `models/__init__.py`. Odoo carga los modelos solo a través de los imports del `__init__.py` — si el archivo no está listado, el campo no existe para el ORM aunque el `.py` esté en el disco.

```python
# models/__init__.py — INCORRECTO (faltaban las últimas dos líneas)
from . import opt_prescription_stage
from . import opt_prescription
from . import opt_work_order
# res_partner.py y sale_order.py existen en disco pero Odoo no los conoce

# models/__init__.py — CORRECTO
from . import opt_prescription_stage
from . import opt_prescription
from . import opt_work_order
from . import res_partner   # ← campo is_optical_client vive aquí
from . import sale_order    # ← también faltaba este
```

**Fix**:  
Agregar las líneas faltantes en `models/__init__.py` y hacer `-u` del módulo.

**Cómo detectarlo**:  
```bash
# Compara los archivos .py del directorio con los imports del __init__.py
ls lt_optica_base/models/*.py | grep -v __init__ | xargs -I{} basename {} .py
# Si aparece algún nombre que NO está en __init__.py → está sin cargar
```

**Regla de oro**: Cada vez que creas un nuevo `.py` en `models/`, agrégalo al `__init__.py` de inmediato — antes de escribir una sola línea de código en él.

---

### Module Won't Install - Dependency Error

**Problem**: `Module X depends on Y which is not available`

**Quick Fix**:
```python
# In __manifest__.py, temporarily comment out the dependency
{
    "name": "My Module",
    "depends": [
        "base",
        # "problematic_module",  # Temporarily disabled
    ],
}
```

**Proper Solution**: Install the missing dependency or remove the actual code dependencies.

---

### Module Stuck in "To Upgrade" State

**Problem**: Module shows "To upgrade" but won't complete

**Quick Fix**:
```sql
-- Reset module state in database
UPDATE ir_module_module 
SET state='installed' 
WHERE name='your_module_name';
```

**Then restart Odoo server**:
```bash
sudo systemctl restart odoo
```

---

### Module Import Error After Upgrade

**Problem**: `ImportError: cannot import name 'X' from 'Y'`

**Quick Fix**:
```bash
# Clear Python cache
cd /path/to/odoo/addons/your_module
find . -type d -name "__pycache__" -exec rm -r {} +
find . -name "*.pyc" -delete

# Restart with clean cache
odoo-bin -c /etc/odoo.conf --stop-after-init
odoo-bin -c /etc/odoo.conf
```

---

## 💾 Database Issues

### Duplicate Key Error on Create

**Problem**: `psycopg2.errors.UniqueViolation: duplicate key value`

**Quick Fix - Reset Sequence**:
```sql
-- Find the table and sequence
SELECT * FROM pg_sequences WHERE sequencename LIKE '%your_table%';

-- Reset sequence to max ID + 1
SELECT setval('your_table_id_seq', (SELECT MAX(id) FROM your_table) + 1);
```

---

### Database Connection Pool Exhausted

**Problem**: `FATAL: sorry, too many clients already`

**Quick Fix**:
```bash
# Edit PostgreSQL config
sudo nano /etc/postgresql/14/main/postgresql.conf

# Increase max_connections
max_connections = 200  # Default is 100

# Restart PostgreSQL
sudo systemctl restart postgresql
```

**Odoo Config Adjustment**:
```ini
# /etc/odoo.conf
db_maxconn = 64  # Default
db_maxconn_gevent = 8  # For gevent mode
```

---

### Slow Query - Missing Index

**Problem**: Queries taking >5 seconds

**Quick Fix - Add Index**:
```sql
-- Create index on frequently queried field
CREATE INDEX idx_model_field ON model_table(field_name);

-- For composite searches
CREATE INDEX idx_model_multi ON model_table(field1, field2);

-- Analyze to update statistics
ANALYZE model_table;
```

---

## 🎨 View & UI Issues

### ✅ Kanban con etapas configurables para módulos custom (patrón robusto)

**Contexto**: `lt_optica_base` — Módulo de recetas ópticas, 2026-05-12  
**Necesidad**: Kanban con columnas configurables, sin perder el flujo de negocio controlado por `state`.

**Patrón recomendado — `state` + `stage_id` duales**:

```python
# models/opt_prescription_stage.py
class OptPrescriptionStage(models.Model):
    _name = 'opt.prescription.stage'
    _description = 'Etapa de receta'
    _order = 'sequence, id'

    name = fields.Char(required=True)
    sequence = fields.Integer(default=10)
    fold = fields.Boolean(default=False)  # Oculta columna vacía en kanban
    state_code = fields.Selection([
        ('draft', 'Borrador'), ('confirmed', 'Confirmada'),
        ('in_lab', 'En Laboratorio'), ('ready', 'Lista para Entrega'),
        ('delivered', 'Entregada'), ('cancelled', 'Cancelada'),
    ])

# models/opt_prescription.py
class OptPrescription(models.Model):
    _name = 'opt.prescription'
    _inherit = ['mail.thread']

    state = fields.Selection([...])          # Controla botones y lógica de negocio
    stage_id = fields.Many2one(              # Controla el kanban
        'opt.prescription.stage',
        group_expand='_read_group_stage_ids'
    )

    @api.model
    def _read_group_stage_ids(self, stages, domain, order):
        """Garantiza que todas las columnas aparezcan aunque estén vacías."""
        return stages.search([], order=order)

    def _sync_stage_from_state(self):
        """Sincroniza stage_id automáticamente cada vez que state cambia."""
        state_to_code = {
            'draft': 'draft', 'confirmed': 'confirmed',
            'in_lab': 'in_lab', 'ready': 'ready',
            'delivered': 'delivered', 'cancelled': 'cancelled',
        }
        for rec in self:
            code = state_to_code.get(rec.state)
            if code:
                stage = self.env['opt.prescription.stage'].search(
                    [('state_code', '=', code)], limit=1
                )
                if stage:
                    rec.stage_id = stage
```

**Etapas precargadas en `data/opt_prescription_stage_data.xml`**:

| seq | name | fold |
|-----|------|------|
| 10 | Borrador | No |
| 20 | Confirmada | No |
| 30 | En Laboratorio | No |
| 40 | Lista para Entrega | No |
| 50 | Entregada | ✅ Sí |
| 99 | Cancelada | ✅ Sí |

**Por qué este patrón**:
- `state` → flujo de negocio, botones del formulario, reglas de negocio
- `stage_id` → kanban visual, configurable por el manager sin tocar código
- `fold=True` en estados finales → kanban limpio cuando no hay registros activos
- `_read_group_stage_ids` → todas las columnas siempre visibles

---

### View Not Refreshing After Changes

**Problem**: XML changes not appearing in UI

**Quick Fix**:
```bash
# Method 1: Update module with view override
odoo-bin -c /etc/odoo.conf -u your_module -d your_db --stop-after-init

# Method 2: Clear assets cache
DELETE FROM ir_attachment WHERE name LIKE '%assets%';
```

**In UI**: Apps → Update Apps List → Update your module

---

### Form View Error - "Field X does not exist"

**Problem**: View references non-existent field

**Quick Fix**:
```xml
<!-- Add invisible field if it's removed from model -->
<field name="legacy_field" invisible="1" optional="hide"/>

<!-- Or wrap in groups to hide from regular users -->
<field name="debug_field" groups="base.group_no_one"/>
```

---

### Kanban View Not Loading

**Problem**: `QWeb error: Invalid template`

**Quick Fix**:
```xml
<!-- Minimal working kanban template -->
<kanban>
    <field name="id"/>
    <field name="name"/>
    <templates>
        <t t-name="kanban-box">
            <div class="oe_kanban_global_click">
                <div class="oe_kanban_details">
                    <strong><field name="name"/></strong>
                </div>
            </div>
        </t>
    </templates>
</kanban>
```

---

### Widget Not Rendering

**Problem**: Custom widget shows empty

**Quick Fix - Check Dependencies**:
```javascript
// In your widget JS file
odoo.define('your_module.widget', function (require) {
    "use strict";
    
    var AbstractField = require('web.AbstractField');
    var fieldRegistry = require('web.field_registry');
    
    // Your widget code
    var YourWidget = AbstractField.extend({
        className: 'o_field_your_widget',
        _render: function () {
            this.$el.text(this.value || 'No value');
        },
    });
    
    fieldRegistry.add('your_widget', YourWidget);
    
    return YourWidget;
});
```

**In view**:
```xml
<field name="your_field" widget="your_widget"/>
```

---

## 🚀 Performance Quick Wins

### Slow List View - Too Many Records

**Problem**: Tree view loading >10 seconds

**Quick Fix - Add Default Filter**:
```xml
<record id="view_model_search" model="ir.ui.view">
    <field name="name">model.search</field>
    <field name="model">your.model</field>
    <field name="arch" type="xml">
        <search>
            <filter name="recent" string="Recent (30 days)" 
                    domain="[('create_date','&gt;=',(context_today()-datetime.timedelta(days=30)).strftime('%Y-%m-%d'))]"
                    default="1"/>
        </search>
    </field>
</record>

<record id="action_model" model="ir.actions.act_window">
    <field name="context">{'search_default_recent': 1}</field>
</record>
```

---

### Form View Slow to Open

**Problem**: Form takes >3 seconds to load

**Quick Fix - Lazy Load Tabs**:
```xml
<notebook>
    <!-- First tab loads immediately -->
    <page string="General" name="general">
        <group>
            <field name="name"/>
        </group>
    </page>
    
    <!-- Other tabs load on click -->
    <page string="Details" name="details" autofocus="autofocus">
        <field name="description"/>
    </page>
</notebook>
```

---

### Compute Field Recalculation Storm

**Problem**: Saving one record triggers thousands of computations

**Quick Fix - Add Store and Dependencies**:
```python
# Before - Expensive computation on every access
total = fields.Float(compute='_compute_total')

@api.depends('line_ids.price')
def _compute_total(self):
    for rec in self:
        rec.total = sum(rec.line_ids.mapped('price'))

# After - Computed once and stored
total = fields.Float(compute='_compute_total', store=True)

@api.depends('line_ids.price')  # Only recompute when price changes
def _compute_total(self):
    for rec in self:
        rec.total = sum(rec.line_ids.mapped('price'))
```

---

## 🔒 Security Fixes

### Access Rights Error for New Model

**Problem**: `AccessError: You are not allowed to access X`

**Quick Fix - Create Security Rules**:
```csv
# security/ir.model.access.csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_model_user,access_model_user,model_model,base.group_user,1,1,1,1
access_model_manager,access_model_manager,model_model,base.group_system,1,1,1,1
```

**In __manifest__.py**:
```python
'data': [
    'security/ir.model.access.csv',
]
```

---

### Record Rules Blocking Legitimate Access

**Problem**: Users can't see their own records

**Quick Fix - Add Record Rule**:
```xml
<record id="rule_model_user" model="ir.rule">
    <field name="name">User can see own records</field>
    <field name="model_id" ref="model_your_model"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
</record>

<!-- Allow managers to see all -->
<record id="rule_model_manager" model="ir.rule">
    <field name="name">Managers see all</field>
    <field name="model_id" ref="model_your_model"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('base.group_system'))]"/>
</record>
```

---

## 🔌 API & Integration

### External API Timeout

**Problem**: `requests.exceptions.Timeout: HTTPConnectionPool`

**Quick Fix - Increase Timeout**:
```python
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_with_retry(url, timeout=30):
    session = requests.Session()
    retry = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    
    return session.get(url, timeout=timeout)

# Usage
response = get_with_retry('https://api.example.com/data')
```

---

### JSON-RPC Endpoint Returns 500

**Problem**: Custom controller failing

**Quick Fix - Add Error Handling**:
```python
from odoo import http
from odoo.http import request
import logging

_logger = logging.getLogger(__name__)

class APIController(http.Controller):
    
    @http.route('/api/endpoint', type='jsonrpc', auth='public', csrf=False)
    def api_endpoint(self, **kw):
        try:
            # Your logic here
            result = self._process_request(kw)
            return {'success': True, 'data': result}
        except Exception as e:
            _logger.error('API Error: %s', str(e), exc_info=True)
            return {
                'success': False, 
                'error': str(e),
                'error_type': type(e).__name__
            }
```

---

### CORS Issues with External Calls

**Problem**: `Access-Control-Allow-Origin` error

**Quick Fix**:
```python
@http.route('/api/public', type='jsonrpc', auth='public', 
            csrf=False, cors='*')  # Allow all origins
def public_api(self, **kw):
    return {'data': 'value'}

# Or specific domain
@http.route('/api/restricted', type='jsonrpc', auth='public',
            csrf=False, cors='https://your-domain.com')
def restricted_api(self, **kw):
    return {'data': 'value'}
```

---

### Webhook Not Receiving Payload

**Problem**: External service webhook returns 200 but Odoo doesn't process

**Quick Fix - Log Everything**:
```python
import json
import logging

_logger = logging.getLogger(__name__)

class WebhookController(http.Controller):
    
    @http.route('/webhook/receive', type='json', auth='public', 
                csrf=False, methods=['POST'])
    def receive_webhook(self, **kw):
        # Log raw request
        _logger.info('=== WEBHOOK RECEIVED ===')
        _logger.info('Headers: %s', request.httprequest.headers)
        _logger.info('Data: %s', json.dumps(kw, indent=2))
        
        try:
            # Process webhook
            self._process_webhook(kw)
            return {'status': 'success'}
        except Exception as e:
            _logger.error('Webhook processing failed: %s', e, exc_info=True)
            return {'status': 'error', 'message': str(e)}
```

---

## 🛠️ Development Quick Fixes

### Python Import Error in Custom Module

**Problem**: `ModuleNotFoundError: No module named 'X'`

**Quick Fix**:
```python
# Add to your module's __init__.py
import sys
import os

# Add parent directory to path
sys.path.insert(0, os.path.dirname(__file__))

# Now import your modules
from . import models
from . import controllers
```

---

### Odoo Server Won't Start - Port Already in Use

**Problem**: `OSError: [Errno 98] Address already in use`

**Quick Fix**:
```bash
# Find process using port 8069
sudo lsof -i :8069

# Kill the process
sudo kill -9 <PID>

# Or change Odoo port
odoo-bin -c /etc/odoo.conf --xmlrpc-port=8070
```

---

### Assets Not Loading After Module Update

**Problem**: CSS/JS changes not appearing

**Quick Fix**:
```bash
# Clear browser cache + Odoo assets
# In browser: Ctrl+Shift+R (hard refresh)

# In database
psql your_database -c "DELETE FROM ir_attachment WHERE name LIKE '%assets%';"

# Restart Odoo
sudo systemctl restart odoo
```

---

## 📊 Common SQL Fixes

### Fix Orphaned Records

```sql
-- Find records without parent
SELECT * FROM child_table 
WHERE parent_id NOT IN (SELECT id FROM parent_table);

-- Delete orphaned records
DELETE FROM child_table 
WHERE parent_id NOT IN (SELECT id FROM parent_table);
```

---

### Fix Sequence Gaps

```sql
-- Reset sequence after bulk delete
SELECT setval('model_table_id_seq', 
              COALESCE((SELECT MAX(id) FROM model_table), 1));
```

---

### Clean Up Duplicate Records

```sql
-- Find duplicates by field
SELECT field_name, COUNT(*) 
FROM model_table 
GROUP BY field_name 
HAVING COUNT(*) > 1;

-- Keep oldest, delete duplicates
DELETE FROM model_table a
USING model_table b
WHERE a.id > b.id 
  AND a.field_name = b.field_name;
```

---

## 🎓 Best Practices for Quick Fixes

### When to Use Quick Fixes

✅ **Good situations**:
- Development/staging environment testing
- Emergency production issue requiring immediate fix
- Temporary workaround while proper solution is developed
- Proof of concept / rapid prototyping

❌ **Bad situations**:
- As permanent solution in production
- When proper fix takes <1 hour
- Without documenting the technical debt
- Without creating ticket for proper fix

---

### Document Your Quick Fixes

Always add comments:
```python
# FIXME: Quick fix for issue #123
# TODO: Refactor to use proper computed field pattern
# Temporary solution applied: 2026-02-23
# See: https://github.com/org/repo/issues/123
total = fields.Float()

def calculate_total(self):
    # Quick calculation without proper caching
    return sum(self.mapped('amount'))
```

---

## 🔗 Related Documents

- [Odoo 19 Lessons Learned](./odoo-19-lessons.md)
- [Performance Lessons](./performance-lessons.md)
- [Integration Lessons](./integration-lessons.md)

---

**Remember**: Quick fixes are temporary solutions. Always plan for proper implementation!

**Document Version**: 1.1  
**Last Updated**: 2026-05-12  
**Maintainer**: Largotek SRL Development Team
