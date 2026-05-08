# Odoo 18 — Portal & eCommerce: Lecciones Aprendidas

> Fecha: 2026-05-07  
> Módulo de referencia: `one_page_checkout` (largotekodoo/staging)  
> Versión: Odoo 18 (odoo.sh)

---

## 1. Herencia de vistas en `res.partner` — Odoo 18

### ❌ Problema
Usar `xpath expr="//page[@name='internal']"` en una vista heredada de `res.partner` falla en Odoo 18:

```xml
<!-- ESTO FALLA en Odoo 18 -->
<xpath expr="//page[@name='internal']" position="after">
    <page string="Mi Tab" name="mi_tab">...</page>
</xpath>
```

Error: `El elemento no se puede localizar en la vista principal`

### ✅ Solución
Usar `<notebook position="inside">` para agregar tabs sin depender del nombre de una página existente:

```xml
<field name="arch" type="xml">
    <notebook position="inside">
        <page string="Mi Tab" name="mi_tab">
            <group>...</group>
        </page>
    </notebook>
</field>
```

### 📌 Por qué
En Odoo 18 la vista base de `res.partner` cambió la estructura interna del notebook. Los nombres de páginas internas (`internal`, `sales`, etc.) pueden no existir o haberse renombrado según la versión exacta. `notebook position="inside"` es agnóstico a la estructura interna y siempre funciona.

---

## 2. Flujo de datos en el checkout portal — El form que nunca hace submit

### Contexto
En Odoo 18 el checkout de eCommerce salta directamente a `/shop/payment` si el cliente ya tiene dirección guardada:

```
GET /shop/checkout?try_skip_step=true  → 303
GET /shop/confirm_order                → 303
GET /shop/payment                      → 200  ← el widget ya está aquí
```

Si el widget (mapa, selector, etc.) está en `/shop/payment`, cualquier dato capturado en inputs ocultos del form **nunca llega al servidor** porque ese form de pago no hace POST con campos custom.

### ❌ Anti-patrón
```javascript
// ESTO NO FUNCIONA — los valores quedan en el DOM pero nunca se envían
document.getElementById('opc_partner_lat').value = lat;
document.getElementById('opc_partner_lng').value = lng;
```

### ✅ Solución: endpoint AJAX dedicado
Crear un endpoint `POST` propio que el JS llame **al instante** cuando el usuario interactúa con el widget, sin esperar ningún form submit:

```javascript
async function saveGeoToServer(lat, lng, addressData) {
    const a = (addressData && addressData.address) ? addressData.address : {};
    const body = new URLSearchParams({
        lat, lng,
        street:       [a.road, a.house_number].filter(Boolean).join(' '),
        city:         a.city || a.town || a.village || '',
        country_code: (a.country_code || 'bo').toUpperCase(),
    });
    await fetch('/shop/update_geo', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: body.toString(),
        credentials: 'same-origin',
    });
}

// Llamar en CADA evento de interacción del usuario:
marker.on('dragend', async () => {
    const pos = marker.getLatLng();
    const data = await reverseGeocode(pos.lat, pos.lng);
    saveGeoToServer(pos.lat, pos.lng, data);  // ← AJAX inmediato
});
```

```python
@http.route('/shop/update_geo', type='http', methods=['POST'],
            auth='public', website=True, sitemap=False, csrf=False)
def shop_update_geo(self, lat=None, lng=None, street=None, city=None,
                    country_code=None, **kw):
    order = request.website.sale_get_order()
    partner = order.sudo().partner_invoice_id
    public_partner = request.website.user_id.sudo().partner_id

    if partner and partner.id != public_partner.id:
        vals = {'opc_latitude': float(lat or 0), 'opc_longitude': float(lng or 0)}
        if street: vals['street'] = street
        if city:   vals['city']   = city
        if country_code:
            country = request.env['res.country'].sudo().search(
                [('code', '=', country_code)], limit=1)
            if country:
                vals['country_id'] = country.id
        partner.sudo().write(vals)
```

### 📌 Regla general
> En Odoo 18 portal, **nunca asumir** que un input oculto en `/shop/payment` llegará al servidor. Si necesitas persistir datos del cliente durante el paso de pago, siempre usar un endpoint AJAX propio.

---

## 3. `partner_id` vs `partner_invoice_id` en ordenes de venta

### ❌ Problema
Usar `order.partner_id` para leer/escribir datos de facturación es incorrecto. En Odoo 18 el partner de facturación puede diferir del partner principal.

### ✅ Solución
Siempre usar `order.sudo().partner_invoice_id` y verificar que no sea el partner público:

```python
partner = order.sudo().partner_invoice_id
public_partner = request.website.user_id.sudo().partner_id

if partner and partner.id != public_partner.id:
    # Cliente logueado — escribir en DB
    partner.sudo().write({...})
else:
    # Cliente anónimo — guardar en sesión
    request.session['opc_lat'] = lat
    request.session['opc_lng'] = lng
```

---

## 4. Campos custom en `res.partner` — patrón correcto

```python
# models/res_partner.py
class ResPartner(models.Model):
    _inherit = 'res.partner'

    opc_latitude  = fields.Float('Latitud OPC',  digits=(10, 7))
    opc_longitude = fields.Float('Longitud OPC', digits=(10, 7))
    opc_maps_url  = fields.Char(
        'Google Maps URL',
        compute='_compute_opc_maps_url',
        store=False
    )

    def _compute_opc_maps_url(self):
        for rec in self:
            if rec.opc_latitude and rec.opc_longitude:
                rec.opc_maps_url = (
                    f'https://www.google.com/maps?q={rec.opc_latitude},{rec.opc_longitude}'
                )
            else:
                rec.opc_maps_url = False
```

> Siempre usar `partner.sudo().write()` al escribir desde contexto portal/public.

---

## 5. Reverse geocode eficiente — cliente vs servidor

### Patrón óptimo

```
Usuario mueve pin
  → JS llama Nominatim (cliente, gratis, sin carga al servidor)
  → JS parsea campos de dirección
  → JS llena inputs del form visualmente
  → JS llama /shop/update_geo con todos los datos ya parseados
  → Servidor solo hace un write() en DB, sin requests externos
```

### Mapeo Nominatim → campos Odoo

| Nominatim | Campo Odoo |
|---|---|
| `road + house_number` | `partner.street` |
| `suburb / neighbourhood / quarter` | `partner.street2` |
| `city / town / village / municipality` | `partner.city` |
| `postcode` (default `'0000'` en Bolivia) | `partner.zip` |
| `country_code` (ISO2) | `partner.country_id` (buscar por `res.country.code`) |

---

## 6. Upsert en nota del pedido (order.note)

Cuando se necesita actualizar una línea específica en la nota del pedido sin duplicarla:

```python
def _write_maps_link_to_order(self, order_sudo, lat, lng):
    maps_label = f'📍 Ubicación de entrega: https://www.google.com/maps?q={lat},{lng}'
    current_note = order_sudo.note or ''

    if 'google.com/maps' in current_note or 'georeferenciado' in current_note:
        # Reemplazar línea existente en lugar de concatenar
        lines = current_note.splitlines()
        new_lines = [
            maps_label if ('google.com/maps' in l or 'georeferenciado' in l) else l
            for l in lines
        ]
        order_sudo.sudo().write({'note': '\n'.join(new_lines)})
    else:
        order_sudo.sudo().write({'note': (current_note + '\n' + maps_label).strip()})
```

---

## 7. CSRF en endpoints AJAX de portal

Odoo 18 valida CSRF en rutas `type='http'`. Para endpoints AJAX desde JS del portal:

```python
@http.route('/shop/update_geo', type='http', methods=['POST'],
            auth='public', website=True, sitemap=False, csrf=False)
```

> ⚠️ Usar `csrf=False` **solo** cuando:
> 1. El endpoint modifica únicamente datos del usuario de la sesión actual
> 2. No realiza acciones financieras ni destructivas
> 3. Tiene validación interna (verificar que el order pertenece a la sesión)

---

## 8. Assets JS en Odoo.sh — limpieza de caché

Odoo.sh **no limpia automáticamente** el caché de assets JS al hacer `git push`. Después de cambios en `.js`:

| Método | Cuándo usar |
|---|---|
| `Ctrl+Shift+R` en browser | Cambios menores, prueba rápida |
| Odoo UI → Settings → Technical → Assets → Clear | Assets desactualizados en staging |
| **Rebuild completo** en Odoo.sh dashboard | Más confiable, último recurso |

---

## Resumen de patrones clave

| Situación | ❌ Evitar | ✅ Usar |
|---|---|---|
| Agregar tab a `res.partner` en v18 | `xpath //page[@name='internal']` | `<notebook position="inside">` |
| Guardar datos desde widget en `/shop/payment` | Inputs ocultos en form | Endpoint AJAX propio |
| Escribir en partner desde portal | `partner.write()` | `partner.sudo().write()` |
| Partner de facturación | `order.partner_id` | `order.partner_invoice_id` |
| Reverse geocode | Hacerlo en backend | JS cliente + enviar resultado |
| Actualizar nota del pedido | Concatenar siempre | Upsert revisando contenido existente |
| CSRF en AJAX portal | Asumir que funciona sin config | `csrf=False` con validación interna |
