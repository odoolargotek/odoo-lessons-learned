# 💳 Odoo 18 — Payment Provider Custom: Lecciones Aprendidas

**Proyecto**: payment_qr_bo (QR Bolivia)  
**Fecha**: 2026-05-08  
**Versión Odoo**: 18.0 (Odoo.sh)  
**Módulo afectado**: `payment_qr_bo`

---

## 🎯 Resumen Ejecutivo

Al desarrollar un payment provider manual (QR estático) para el checkout de
Odoo 18 Website, se encontraron tres errores consecutivos derivados de un
malentendido sobre cómo Odoo 18 renderiza el formulario inline del proveedor
de pago.

**Total de errores documentados**: 3  
**Raíz común**: cómo `_get_processing_values()` pasa el contexto a `ir.qweb._render()`

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
    res['qr_image'] = self.provider_id.qr_bo_image  # <-- campo Binary, binario!
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
    rendering_values  # <-- este dict ES el contexto QWeb
)
```

`rendering_values` (el dict retornado por `_get_specific_rendering_values`)
se pasa como el **contexto completo** de QWeb. No existe una variable llamada
`rendering_values` dentro del template — sus **claves** son las variables.

Por eso `rendering_values.get('key')` dentro del template lanza `KeyError`.

### Solución

En el template, usar directamente las claves del dict como variables:

```xml
<!-- ❌ MAL — rendering_values no existe como variable dentro del template -->
<t t-out="rendering_values.get('bank_name') or 'Banco'"/>
<t t-out="rendering_values.get('qr_image_url')"/>

<!-- ✅ BIEN — usar la clave directamente como variable -->
<t t-out="bank_name or 'Banco'"/>
<img t-att-src="qr_image_url"/>
```

---

## 🔴 Error 3: `KeyError: 'provider_sudo'` en el template QWeb

**Severidad**: 🔴 HIGH — RPC_ERROR, el servidor aborta el render  
**Odoo version**: 18.0  
**Traceback clave**:
```
KeyError: 'provider_sudo'
Path: /t/div/div[1]/div/small/t[1]
Node: <t t-out="provider_sudo.qr_bo_bank_name or 'Transferencia bancaria'"/>
```

### Causa raíz

Confusión natural: en otros contextos de Odoo (portal, website controllers),
`provider_sudo` y `transaction_sudo` son variables estándar del contexto.
PERO en el render del formulario de pago vía `_get_processing_values()`,
el contexto QWeb contiene **únicamente** lo que devuelve
`_get_specific_rendering_values()`.

`provider_sudo` y `transaction_sudo` **no se inyectan automáticamente** —
deben agregarse manualmente al dict.

### Solución

Inyectar los recordsets en `_get_specific_rendering_values()`:

```python
def _get_specific_rendering_values(self, processing_values):
    res = super()._get_specific_rendering_values(processing_values)
    if self.provider_code != 'qr_bo':
        return res

    provider = self.provider_id
    res.update({
        # Inyectar explicitamente para usarlos en el template
        'provider_sudo':    provider,
        'transaction_sudo': self,
        # Valores simples adicionales
        'qr_image_url': (
            '/web/image/payment.provider/%d/qr_bo_image' % provider.id
            if provider.qr_bo_image else False
        ),
        'bank_name':    provider.qr_bo_bank_name or '',
        'account_name': provider.qr_bo_account_name or '',
        'instructions': provider.qr_bo_instructions or '',
        'tx_reference': self.reference,
    })
    return res
```

---

## 💡 Regla de oro para templates de payment providers en Odoo 18

> **El contexto QWeb del template de un payment provider es exactamente y solo
> el dict devuelto por `_get_specific_rendering_values()`.**

No asumas que existen variables como `provider_sudo`, `transaction_sudo`,
`rendering_values`, `request`, etc. Debes inyectarlas tú mismo si las
necesitas en el template.

### Resumen de variables disponibles en el template

| Variable | ¿Disponible automáticamente? | Cómo obtenerla |
|----------|-------------------------------|----------------|
| `provider_sudo` | ❌ No | Agregar en `_get_specific_rendering_values()` |
| `transaction_sudo` | ❌ No | Agregar en `_get_specific_rendering_values()` |
| `rendering_values` | ❌ No (es el contexto, no una var) | No aplica |
| Tus claves custom | ✅ Sí | Las que devuelvas en el dict |

### Template mínimo correcto para Odoo 18

```xml
<template id="payment_miprovider_form" name="Mi Provider - Inline Form">
    <div>
        <!-- provider_sudo y transaction_sudo disponibles porque
             los inyectamos en _get_specific_rendering_values() -->
        <p>Banco: <t t-out="provider_sudo.mi_campo_banco"/></p>
        <p>Referencia: <t t-out="transaction_sudo.reference"/></p>

        <form action="/payment/miprovider/submit" method="post"
              enctype="multipart/form-data">
            <input type="hidden" name="csrf_token"
                   t-att-value="request.csrf_token()"
                   t-nocache="The csrf token must always be up to date."/>
            <input type="hidden" name="reference"
                   t-att-value="transaction_sudo.reference"/>
            <!-- ... campos del formulario ... -->
            <button type="submit">Confirmar pago</button>
        </form>
    </div>
</template>
```

---

## 📋 Checklist para crear un payment provider manual en Odoo 18

- [ ] `_should_build_inline_form(self, is_validation=False)` → retorna `True`
- [ ] `_get_inline_form_view()` → retorna `self.env.ref('mi_modulo.mi_template')`
- [ ] `_get_redirect_form_view(self, is_validation=False)` → mismo template (fallback)
- [ ] `_get_specific_rendering_values()` inyecta `provider_sudo` y `transaction_sudo`
- [ ] `_get_specific_rendering_values()` solo devuelve valores JSON-serializables
- [ ] Template usa variables directamente (no `rendering_values.get('...')`)
- [ ] Form del template hace POST a ruta propia, no depende del bus OWL
- [ ] Ruta del controller redirige a `/shop/payment/validate` al final

---

## 🔗 Referencias

- [payment/models/payment_transaction.py L455](https://github.com/odoo/odoo/blob/18.0/addons/payment/models/payment_transaction.py)
- [Repositorio largotekodoo — payment_qr_bo](https://github.com/odoolargotek/largotekodoo)
- Errores documentados en staging: `odoolargotek-largotekodoo-staging-31918364.dev.odoo.com` (2026-05-08)

---

**Documentado por**: Juan Luis Garvía Ossio / Largotek SRL  
**Fecha**: 2026-05-08  
**Estado**: 🔧 En corrección (staging)
