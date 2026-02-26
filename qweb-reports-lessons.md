# 🖨️ QWeb Reports Lessons Learned

**Purpose**: Solutions for common QWeb report rendering issues in Odoo  
**Versions**: Odoo 15, 16, 17, 18, 19  
**Last Updated**: 2026-02-26

---

## 🎯 Table of Contents

- [Blank PDF Reports](#-blank-pdf-reports)
- [Template Caching Issues](#-template-caching-issues)
- [Field Access Errors](#-field-access-errors)
- [Report Data Issues](#-report-data-issues)

---

## 📄 Blank PDF Reports

### Problem: PDF Generates but Content is Empty

**Symptoms**:
- PDF file downloads successfully (200 OK response)
- File size is very small (few KB)
- Opening PDF shows completely blank pages
- No errors in Odoo logs

**Root Causes**:

1. **No data to display** - Most common
2. **Template conditions hiding all content**
3. **Field access errors silently failing**
4. **Missing template variables**

---

### Case Study: Missing Related Records

**Context**: Certificado de Flete report generating blank PDFs

**Investigation Steps**:

```python
# Step 1: Check if report generates (look for 200 response)
# In logs: "The PDF report has been generated for model: X, records [Y]."

# Step 2: Verify data exists
odoo-bin shell
```

```python
# In Odoo shell - check record has required data
file = env['carga.internacional'].browse(65)
print(f"File: {file.name}")
print(f"Total NC lines: {len(file.nc_line_ids)}")
print(f"Lines to print: {len(file.nc_line_ids.filtered(lambda x: x.print_nc))}")

# If count is 0, that's your problem!
```

**Solution**:

```python
# Option 1: Create test data
env['carga.internacional.nc.line'].create({
    'carga_id': file.id,
    'descripcion': 'Flete Internacional',
    'cantidad': 1,
    'precio_unitario': 1500.00,
    'currency_id': env.company.currency_id.id,
    'print_nc': True,  # Critical: must be True to appear in report
})
env.cr.commit()

# Option 2: Mark existing lines for printing
file.nc_line_ids.write({'print_nc': True})
env.cr.commit()
```

**Template Pattern to Avoid**:

```xml
<!-- BAD: Silent failure if no data -->
<t t-foreach="docs" t-as="doc">
    <t t-set="nc_lines" t-value="doc.nc_line_ids.filtered(lambda x: x.print_nc)"/>
    <!-- If nc_lines is empty, nothing renders! -->
    <t t-foreach="nc_lines" t-as="line">
        <tr>...</tr>
    </t>
</t>
```

**Better Pattern**:

```xml
<!-- GOOD: Show message when no data -->
<t t-foreach="docs" t-as="doc">
    <t t-set="nc_lines" t-value="doc.nc_line_ids.filtered(lambda x: x.print_nc)"/>
    
    <t t-if="nc_lines">
        <t t-foreach="nc_lines" t-as="line">
            <tr>...</tr>
        </t>
    </t>
    <t t-else="">
        <tr>
            <td colspan="6" class="text-center" style="padding:12px;">
                No existen líneas marcadas para imprimir (Imprimir NC = True).
            </td>
        </tr>
    </t>
</t>
```

---

## 🔄 Template Caching Issues

### Problem: Template Changes Not Appearing in Reports

**Symptoms**:
- Modified QWeb template XML
- Updated module successfully
- Report still shows old content
- Fields exist in database but report shows "-" or blank

**Root Cause**: Odoo caches compiled QWeb templates in memory and sometimes in database.

---

### Solution 1: Clear QWeb Cache (Odoo Shell)

```bash
odoo-bin shell
```

```python
# Clear all QWeb template caches
env['ir.ui.view'].clear_caches()
env.cr.commit()
exit
```

---

### Solution 2: Restart Odoo Workers

```bash
# On Odoo.sh
odoosh-restart

# On regular server
sudo systemctl restart odoo

# Or kill specific workers
sudo pkill -f odoo-bin
sudo systemctl start odoo
```

---

### Solution 3: Force Module Update

```bash
# Update module with view changes
odoo-update your_module_name

# Or from command line
odoo-bin -u your_module -d your_database --stop-after-init
```

---

### Solution 4: Clear Browser Cache

```
Ctrl + Shift + R  (Windows/Linux)
Cmd + Shift + R   (Mac)
```

---

### Complete Checklist for Template Updates

```bash
# 1. Verify fields exist in database
psql your_database
\d table_name  # Check if columns exist

# 2. Verify fields registered in Odoo
odoo-bin shell
env['ir.model.fields'].search([
    ('model', '=', 'your.model'),
    ('name', 'in', ['field1', 'field2'])
])

# 3. Clear QWeb cache
env['ir.ui.view'].clear_caches()
env.cr.commit()

# 4. Restart server
odoosh-restart  # or sudo systemctl restart odoo

# 5. Clear browser cache
# Ctrl+Shift+R in browser

# 6. Test report again
```

---

## ⚠️ Field Access Errors

### Problem: Template Fails When Accessing New Fields

**Symptoms**:
- Report was working before
- Added new fields to model
- Now report is blank or shows error
- Logs show `KeyError` or `AttributeError`

**Root Cause**: Template tries to access fields before they exist in database.

---

### Solution: Safe Field Access Pattern

```xml
<!-- BAD: Direct access fails if field doesn't exist -->
<td><t t-esc="doc.new_field"/></td>

<!-- GOOD: Check if field exists first -->
<td>
    <t t-if="'new_field' in doc._fields and doc.new_field">
        <t t-esc="doc.new_field"/>
    </t>
    <t t-else="">-</t>
</td>
```

---

### Pattern: Conditional Field Access for Related Models

```xml
<!-- Example: Accessing fields from One2many records -->
<t t-set="terrestre_ids" t-value="('terrestre_ids' in doc._fields and doc.terrestre_ids) or False"/>

<!-- Safe access to fields in related records -->
<t t-if="terrestre_ids">
    <t t-set="first_terr" t-value="terrestre_ids[0]"/>
    <t t-if="'recojo_terrestre' in first_terr._fields and first_terr.recojo_terrestre">
        <t t-esc="first_terr.recojo_terrestre"/>
    </t>
</t>
```

**Why This Works**:
- Checks if field exists in `_fields` dictionary
- Won't crash if field is removed
- Handles both model-level and instance-level checks
- Provides fallback value

---

### Real Example: Adding Fields to Existing Report

**Scenario**: Adding `recojo_terrestre` and `entrega_terrestre` to terrestrial shipment report

**Step 1: Add fields to model**

```python
# In models/carga_internacional_terrestre.py
class CargaInternacionalTerrestre(models.Model):
    _name = 'carga.internacional.terrestre'
    
    recojo_terrestre = fields.Char(
        string='Recojo Terrestre',
        help='Lugar de recojo para transporte terrestre'
    )
    entrega_terrestre = fields.Char(
        string='Entrega Terrestre', 
        help='Lugar de entrega para transporte terrestre'
    )
```

**Step 2: Update template safely**

```xml
<!-- Set variables at template start -->
<t t-set="tipo_embarque_val" t-value="doc.tipo_embarque or ''"/>
<t t-set="terrestre_ids" t-value="('terrestre_ids' in doc._fields and doc.terrestre_ids) or False"/>

<!-- Extract values from related model -->
<t t-set="recojo_terrestre_val" t-value="'-'"/>
<t t-set="entrega_terrestre_val" t-value="'-'"/>
<t t-if="tipo_embarque_val == 'inland' and terrestre_ids">
    <t t-set="first_terr" t-value="terrestre_ids[0]"/>
    <t t-if="'recojo_terrestre' in first_terr._fields and first_terr.recojo_terrestre">
        <t t-set="recojo_terrestre_val" t-value="first_terr.recojo_terrestre"/>
    </t>
    <t t-if="'entrega_terrestre' in first_terr._fields and first_terr.entrega_terrestre">
        <t t-set="entrega_terrestre_val" t-value="first_terr.entrega_terrestre"/>
    </t>
</t>

<!-- Use in table with type-specific logic -->
<tr>
    <th>Recojo</th>
    <td>
        <!-- If terrestrial, use specific field -->
        <t t-if="tipo_embarque_val == 'inland' and recojo_terrestre_val != '-'">
            <t t-esc="recojo_terrestre_val"/>
        </t>
        <!-- Otherwise use generic field -->
        <t t-elif="'recojo' in doc._fields">
            <t t-esc="doc.recojo or '-'"/>
        </t>
        <t t-else="">-</t>
    </td>
</tr>
```

**Step 3: Deployment procedure**

```bash
# 1. Push code changes
git add .
git commit -m "feat: Add recojo_terrestre and entrega_terrestre fields"
git push origin staging

# 2. Wait for Odoo.sh deployment

# 3. Update base module first (has the model)
odoo-update carga_internacional

# 4. Update report module (has the template)
odoo-update carga_internacional_sales_report

# 5. Clear caches
odoo-bin shell
env['ir.ui.view'].clear_caches()
env.cr.commit()
exit

# 6. Restart workers
odoosh-restart

# 7. Test report
```

---

## 📊 Report Data Issues

### Problem: Report Shows Incorrect or Missing Data

**Debugging Technique: Add Debug Output**

```xml
<!-- Temporary debug block in template -->
<div style="border: 2px solid red; padding: 10px; margin: 10px;">
    <h3>DEBUG INFO</h3>
    <p>Doc ID: <t t-esc="doc.id"/></p>
    <p>Doc Name: <t t-esc="doc.name"/></p>
    <p>Tipo Embarque: <t t-esc="tipo_embarque_val"/></p>
    <p>Terrestre IDs count: <t t-esc="len(terrestre_ids) if terrestre_ids else 0"/></p>
    <p>NC Lines count: <t t-esc="len(nc_lines)"/></p>
    <t t-if="terrestre_ids">
        <p>First Terrestre: <t t-esc="terrestre_ids[0].id"/></p>
        <p>Has recojo_terrestre: <t t-esc="'recojo_terrestre' in terrestre_ids[0]._fields"/></p>
        <p>Recojo value: <t t-esc="terrestre_ids[0].recojo_terrestre if 'recojo_terrestre' in terrestre_ids[0]._fields else 'N/A'"/></p>
    </t>
</div>
```

Remove this after debugging!

---

### Check Data in Shell

```python
# Connect to Odoo shell
odoo-bin shell

# Load your record
record = env['your.model'].browse(RECORD_ID)

# Check all fields
print("Fields available:")
for field_name in sorted(record._fields.keys()):
    try:
        value = record[field_name]
        print(f"  {field_name}: {value}")
    except Exception as e:
        print(f"  {field_name}: ERROR - {e}")

# Check related records
if hasattr(record, 'line_ids'):
    print(f"\nRelated lines: {len(record.line_ids)}")
    for line in record.line_ids:
        print(f"  Line {line.id}: {line.display_name}")
```

---

## 🎓 Best Practices for QWeb Reports

### 1. Always Provide Fallbacks

```xml
<!-- BAD -->
<td><t t-esc="doc.field"/></td>

<!-- GOOD -->
<td><t t-esc="doc.field or '-'"/></td>
```

---

### 2. Check Field Existence

```xml
<!-- For direct fields -->
<t t-if="'field_name' in doc._fields and doc.field_name">
    <t t-esc="doc.field_name"/>
</t>

<!-- For related fields -->
<t t-if="'related_ids' in doc._fields and doc.related_ids">
    <t t-foreach="doc.related_ids" t-as="rel">
        ...
    </t>
</t>
```

---

### 3. Handle Empty Collections

```xml
<t t-if="doc.line_ids">
    <table>
        <t t-foreach="doc.line_ids" t-as="line">
            <tr><td><t t-esc="line.name"/></td></tr>
        </t>
    </table>
</t>
<t t-else="">
    <p>No hay líneas para mostrar.</p>
</t>
```

---

### 4. Use Computed Variables

```xml
<!-- Set variables at top for reuse -->
<t t-set="lines_to_print" t-value="doc.line_ids.filtered(lambda x: x.print_me)"/>
<t t-set="has_data" t-value="bool(lines_to_print)"/>

<!-- Use throughout template -->
<t t-if="has_data">
    <!-- Display logic -->
</t>
```

---

### 5. Safe Type Checking

```xml
<!-- For selection fields -->
<t t-if="tipo_embarque_val == 'inland'">
    <!-- Terrestrial specific -->
</t>
<t t-elif="tipo_embarque_val == 'air'">
    <!-- Air specific -->
</t>
<t t-elif="tipo_embarque_val == 'maritime'">
    <!-- Maritime specific -->
</t>
<t t-else="">
    <!-- Default/unknown -->
</t>
```

---

### 6. Monetary Fields Formatting

```xml
<!-- Use proper monetary widget -->
<span t-field="line.price_unit" 
      t-options="{'widget':'monetary','display_currency': currency}"/>

<!-- Or manual formatting -->
<t t-esc="'%.2f' % line.price_unit"/> <t t-esc="currency.symbol"/>
```

---

## 🔍 Debugging Checklist

When a report is blank or broken:

```
☐ Check Odoo logs for errors (tail -f ~/logs/odoo.log)
☐ Verify record exists and has data (odoo-bin shell)
☐ Check template has <t t-else> fallbacks
☐ Verify fields exist in database (\d table_name in psql)
☐ Verify fields registered in Odoo (ir.model.fields)
☐ Clear QWeb cache (env['ir.ui.view'].clear_caches())
☐ Restart Odoo workers (odoosh-restart)
☐ Clear browser cache (Ctrl+Shift+R)
☐ Add debug output to template temporarily
☐ Test with known good record
```

---

## 📚 Related Documentation

- [Odoo QWeb Documentation](https://www.odoo.com/documentation/15.0/developer/reference/frontend/qweb.html)
- [Odoo Reports Guide](https://www.odoo.com/documentation/15.0/developer/reference/backend/reports.html)
- [Quick Fixes](./quick-fixes.md#-view--ui-issues)

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-26  
**Contributors**: Juan Luis Garvía Ossio  
**Maintainer**: Largotek SRL Development Team  
**License**: CC-BY-4.0
