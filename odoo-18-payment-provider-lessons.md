# 💳 Odoo 18 — Payment Provider Custom: Lecciones Aprendidas

**Proyecto**: payment_qr_bo (QR Bolivia)  
**Fecha**: 2026-05-08  
**Versión Odoo**: 18.0 (Odoo.sh)  
**Módulo afectado**: `payment_qr_bo`

---

## 🎯 Resumen Ejecutivo

Al desarrollar un payment provider manual (QR estático) para el checkout de
Odoo 18 Website, se encontraron errores consecutivos derivados de:
1. Cómo Odoo 18 renderiza el formulario inline del proveedor de pago
2. Cómo Odoo 18 valida XPath en herencia de templates
3. Cómo funciona CSRF en endpoints `type='http'` con `fetch`
4. Cómo se vincula `payment.transaction` con `sale.order` en estado `pending`

**Total de errores documentados**: 7  

---

## 🔴 Error 1: Pantalla gris en el checkout (overlay bloqueante)

**Severidad**: 🔴 HIGH — el flujo de pago queda bloqueado visualmente  
**Síntoma**: Al seleccionar el método de pago y hacer clic en "Pagar", la pantalla
se pone gris translucido y no responde. No hay error visible en pantalla.

### Causa raíz

El checkout de Odoo 18 usa un sistema OWL que hace un `POST /shop/payment/transaction`
y espera recibir `redirect_form_html` (HTML renderizado del formulario inline).
Si `_get_specific_rendering_values()` devuelve datos no serializables a JSON
(binarios base64, recordsets sin serializar), el JS no puede procesar la
respuesta y el overlay queda bloqueado.

### Solución

Asegurarse que `_get_specific_rendering_values()` devuelva **solo valores
serializables**: strings, números, booleanos. Las imágenes deben pasarse como
**URL string**, nunca como binario.

```python
# ❌ MAL — devuelve binario base64, no serializable a JSON
def _get_specific_rendering_values(self, processing_values):
    res = super()._get_specific_rendering_values(processing_values)
    if self.provider_code != 'qr_bo':
        return res
    res['qr_image'] = self.provider_id.qr_bo_image  # <-- campo Binary!
    return res

# ✅ BIEN — devuelve URL string
def _get_specific_rendering_values(self, processing_values):
    res = super()._get_specific_rendering_values(processing_values)
    if self.provider_code != 'qr_bo':
        return res
    provider = self.provider_id
    res['qr_image_url'] = (
        '/web/image/payment.provider/%d/qr_bo_image' % provider.id
        if provider.qr_bo_image else False
    )
    return res
```

---

## 🔴 Error 2: `KeyError: 'rendering_values'` en el template QWeb

**Severidad**: 🔴 HIGH — RPC_ERROR, el servidor aborta el render  
**Odoo version**: 18.0  
**Traceback clave**:
```
KeyError: 'rendering_values'
Path: /t/div/div[1]/div/small/t[1]
Node: <t t-out="rendering_values.get('bank_name') or ..."/>
```

### Causa raíz

En Odoo 18, el método `_get_processing_values()` llama internamente a:

```python
# odoo/addons/payment/models/payment_transaction.py
redirect_form_html = self.env['ir.qweb']._render(
    redirect_form_view.id,
    rendering_values  # <-- este dict ES el contexto QWeb completo
)
```

`rendering_values` se pasa como el **contexto completo** de QWeb. No existe
una variable llamada `rendering_values` dentro del template — sus **claves**
son las variables directamente.

### Solución

```xml
<!-- ❌ MAL -->
<t t-out="rendering_values.get('bank_name') or 'Banco'"/>

<!-- ✅ BIEN -->
<t t-out="bank_name or 'Banco'"/>
```

---

## 🔴 Error 3: `KeyError: 'provider_sudo'` en el template QWeb

**Severidad**: 🔴 HIGH — RPC_ERROR  

### Causa raíz

`provider_sudo` y `transaction_sudo` **no se inyectan automáticamente** en el
contexto del template de pago. Deben agregarse manualmente.

### Solución

```python
def _get_specific_rendering_values(self, processing_values):
    res = super()._get_specific_rendering_values(processing_values)
    if self.provider_code != 'qr_bo':
        return res
    provider = self.provider_id
    res.update({
        'provider_sudo':    provider,
        'transaction_sudo': self,
        'qr_image_url': (
            '/web/image/payment.provider/%d/qr_bo_image' % provider.id
            if provider.qr_bo_image else False
        ),
    })
    return res
```

---

## 🔴 Error 4: XPath `//div[hasclass('oe_website_sale')]` no localizable

**Fecha**: 2026-05-08  
**Severidad**: 🔴 HIGH — módulo no instala, RPC_ERROR en `ir.module.module`  
**Síntoma**:
```
odoo.tools.convert.ParseError: while parsing None:227
El elemento "<xpath expr="//div[hasclass('oe_website_sale')]">" no se puede
localizar en la vista principal
```

### Causa raíz

Odoo valida el XPath de herencia **solo contra el DOM literal del template padre**,
no contra el HTML renderizado. `website_sale.confirmation` en Odoo 18 es:

```xml
<template id="confirmation">
    <t t-call="website_sale.checkout_layout">  <!-- ← no genera DOM propio -->
        ...
        <div class="oe_structure" id="oe_structure_website_sale_confirmation_1"/>
        <div class="oe_structure" id="oe_structure_website_sale_confirmation_2"/>
        ...
    </t>
</template>
```

`div.oe_website_sale` vive dentro de `checkout_layout`, no en el template
`confirmation` directamente. Tampoco existe `div#wrap` en su DOM literal.

### Intentos fallidos

| XPath intentado | Por qué falla |
|---|---|
| `//div[hasclass('oe_website_sale')]` | Está en `checkout_layout`, no en `confirmation` |
| `//div[@id='wrap']` | Está en `website.layout`, no en `confirmation` |

### Solución

Usar un elemento que **sí existe literalmente** en el DOM de `confirmation`:

```xml
<template id="qr_bo_inject_confirmation"
          inherit_id="website_sale.confirmation"
          name="QR Bolivia - Inyeccion en confirmacion">
    <!-- ✅ Este div SÍ existe literalmente en website_sale.confirmation -->
    <xpath expr="//div[@id='oe_structure_website_sale_confirmation_1']" position="before">
        <t t-if="qr_bo_tx">
            <t t-call="payment_qr_bo.qr_bo_confirmation_block"/>
        </t>
    </xpath>
</template>
```

### Regla general

> Si el template padre usa `<t t-call="otro.template">`, **ningún elemento
> de ese otro template es seleccionable por XPath**. Solo los elementos
> definidos directamente en el XML del template padre son válidos como anchor.

**Para diagnosticar el DOM literal de cualquier template**:
```bash
python3 -c "
import odoo; odoo.tools.config.parse_config(['--config=/etc/odoo/odoo.conf'])
from odoo import api, registry
with registry('mi_db').cursor() as cr:
    env = api.Environment(cr, 1, {})
    print(env.ref('website_sale.confirmation').arch)
"
```

---

## 🔴 Error 5: HTTP 403 en endpoint AJAX `type='http'` con `fetch + FormData`

**Fecha**: 2026-05-08  
**Severidad**: 🔴 HIGH — upload silenciosamente rechazado  
**Síntoma**: `fetch()` recibe HTTP 403, `Failed to load resource: 403`

### Causa raíz

Dos problemas combinados:
1. Con `csrf=True`, Odoo valida el token estrictamente. Un `fetch()` con
   `FormData` no envía el token de la misma forma que un `<form>` submit
   nativo → rechazo.
2. Al intentar solucionar con `csrf=False` + `request.validate_csrf()` manual,
   ese método **no existe en Odoo 18** como API pública → `AttributeError`
   que el framework convierte silenciosamente en `403`.

### Intento fallido

```python
# ❌ MAL — validate_csrf() no existe en Odoo 18
csrf_token = post.get('csrf_token')
if not request.validate_csrf(csrf_token):  # AttributeError → 403
    return _json(False, 'CSRF inválido', 403)
```

### Solución

```python
@http.route(
    '/payment/qr_bo/upload_voucher',
    type='http',
    methods=['POST'],
    auth='public',
    website=True,
    csrf=False,  # ✅ Sin validación CSRF — seguro por validación de referencia
)
def qr_bo_upload_voucher(self, **post):
    # Seguridad garantizada por:
    # - reference debe matchear una tx real con provider_code='qr_bo'
    # - idempotente: bloquea si ya tiene voucher
    ...
```

### Cuándo es seguro `csrf=False`

| Condición | ¿Seguro sin CSRF? |
|---|---|
| El endpoint valida una referencia única en BD | ✅ Sí |
| La acción es idempotente (no se puede repetir) | ✅ Sí |
| El endpoint hace acciones financieras irreversibles | ❌ No |
| Solo lectura | ✅ Siempre |

---

## 🔴 Error 6: Adjunto no aparece en la orden de venta

**Fecha**: 2026-05-08  
**Severidad**: 🟡 MEDIUM — funcionalidad incompleta, no bloquea el flujo  
**Síntoma**: El comprobante se guarda en `tx.qr_bo_voucher` pero **no aparece
en los adjuntos ni en el chatter de la orden de venta**.

### Causa raíz

En Odoo 18, `payment.transaction.sale_order_ids` es un Many2many que se puebla
en `_reconcile_after_done()`. Este método **solo corre cuando la tx pasa a
`done`**. En estado `pending`, `sale_order_ids` está **vacío**:

```python
# ❌ MAL — sale_order_ids vacío en estado pending
for order in tx_sudo.sale_order_ids:  # no itera nunca en pending
    env['ir.attachment'].sudo().create({...})
```

### Solución — `message_post` con adjunto

Usar `message_post` es el patrón nativo de Odoo para comprobantes de pago
(idéntico a lo que hace la transferencia bancaria manual):

```python
order.sudo().message_post(
    body=(
        '<p><strong>Comprobante de pago QR Bolivia recibido</strong></p>'
        '<p>Referencia: <code>%s</code></p>'
        '<p>Verifica el comprobante adjunto y confirma el pago.</p>'
        % reference
    ),
    subject='Comprobante QR Bolivia - %s' % reference,
    message_type='comment',
    subtype_xmlid='mail.mt_comment',
    attachments=[(filename, raw_bytes)],  # ← tupla (nombre, bytes crudos)
)
```

**`message_post` vs `ir.attachment.create` directo**:

| | `ir.attachment.create` | `message_post` |
|---|---|---|
| Visible en chatter | ❌ No | ✅ Sí |
| Notifica seguidores | ❌ No | ✅ Sí |
| Requiere `sale_order_ids` poblado | ✅ Sí | ❌ No |
| Patrón nativo Odoo | ❌ No | ✅ Sí |

### Cadena de fallbacks para resolver la sale.order en pending

```python
def _resolve_sale_order(self, tx_sudo):
    # 1. M2M estándar (funciona post-reconciliación / estado done)
    if tx_sudo.sale_order_ids:
        return tx_sudo.sale_order_ids[0]

    # 2. Campo inverso transaction_ids en sale.order (funciona desde pending)
    order = request.env['sale.order'].sudo().search(
        [('transaction_ids', 'in', tx_sudo.ids)], limit=1, order='id desc'
    )
    if order:
        return order

    # 3. Última orden de la sesión web activa
    last_order_id = request.session.get('sale_last_order_id')
    if last_order_id:
        order = request.env['sale.order'].sudo().browse(last_order_id)
        if order.exists():
            return order

    return None
```

---

## 💡 Reglas de oro para payment providers en Odoo 18

1. **Contexto QWeb** = exactamente lo que devuelve `_get_specific_rendering_values()`. Nada más.
2. **XPath en herencia**: solo elementos que existen en el DOM **literal** del template padre, no en sus `t-call`.
3. **`request.validate_csrf()` no existe en Odoo 18** — usar `csrf=False` directamente para endpoints AJAX multipart.
4. **`sale_order_ids` vacío en pending** — usar `sale.order.transaction_ids` (campo inverso) o `session['sale_last_order_id']`.
5. **Adjuntos de comprobante** — siempre usar `message_post(attachments=[(filename, raw)])`, no `ir.attachment.create` directo.

---

## 📋 Checklist para crear un payment provider manual en Odoo 18

- [ ] `_should_build_inline_form()` → retorna `True`
- [ ] `_get_inline_form_view()` → retorna `self.env.ref('mi_modulo.mi_template')`
- [ ] `_get_specific_rendering_values()` inyecta `provider_sudo` y `transaction_sudo`
- [ ] `_get_specific_rendering_values()` solo devuelve valores JSON-serializables (no Binary)
- [ ] Template usa variables directamente (no `rendering_values.get('...')`)
- [ ] Form hace POST a ruta propia, redirige a `/shop/payment/validate`
- [ ] XPath de herencia apunta a elemento que existe en el DOM **literal** del padre
- [ ] Endpoint de upload usa `csrf=False` (no `request.validate_csrf()`)
- [ ] Adjunto al comprobante via `order.message_post(attachments=[(filename, raw)])`
- [ ] Resolver `sale.order` via `transaction_ids` (no `sale_order_ids`) en estado pending

---

## 🔗 Referencias

- [payment/models/payment_transaction.py Odoo 18](https://github.com/odoo/odoo/blob/18.0/addons/payment/models/payment_transaction.py)
- [website_sale/views/templates.xml Odoo 18](https://github.com/odoo/odoo/blob/18.0/addons/website_sale/views/templates.xml)
- [Repositorio largotekodoo — payment_qr_bo](https://github.com/odoolargotek/largotekodoo)
- Errores documentados en staging: `odoolargotek-largotekodoo-staging-31918364.dev.odoo.com` (2026-05-08)

---

**Documentado por**: Juan Luis Garvía Ossio / Largotek SRL  
**Última actualización**: 2026-05-08  
**Estado**: ✅ Resuelto en staging
