# 📚 Odoo Lessons Learned

Comprehensive knowledge base documenting Odoo version-specific issues, solutions, breaking changes, and best practices across multiple Odoo versions.

## 🎯 Purpose

This repository serves as a **centralized reference** for:
- ✅ **Broken functionality** across Odoo versions
- ✅ **Breaking changes** from one version to another
- ✅ **Solutions & workarounds** to common issues
- ✅ **Best practices** per Odoo version
- ✅ **Performance tips** and optimization patterns
- ✅ **Security considerations** version-specific

## 📑 Available Documentation

### By Odoo Version

#### Odoo 19 🆕
- [**odoo-19-lessons.md**](./odoo-19-lessons.md) - Complete migration guide from Odoo 18 to 19 with breaking changes and solutions

#### Odoo 18
- [**odoo-18-lessons.md**](./odoo-18-lessons.md) - Issues, breaking changes, and solutions for Odoo 18
- [**odoo-18-payment-provider-lessons.md**](./odoo-18-payment-provider-lessons.md) - Custom payment provider: QWeb context, inline form, rendering_values
- [**odoo-18-portal-ecommerce.md**](./odoo-18-portal-ecommerce.md) - Portal and eCommerce customization patterns
- [**odoo-18-stock-cancel-lessons.md**](./odoo-18-stock-cancel-lessons.md) - Stock cancellation and inventory flow issues

#### Odoo 17
- *Planned*

#### Odoo 16
- *Planned*

#### Odoo 15 ✅
- [**odoo-15-lessons.md**](./odoo-15-lessons.md) - `crm.lead` computed fields, `order_ids` vs `sale_ids`, kanban views, bulk data population

#### Odoo 14 (Legacy)
- *Planned*

### By Topic (Cross-Version)

#### Reports & Views 🖨️
- [**qweb-reports-lessons.md**](./qweb-reports-lessons.md) - QWeb report issues: blank PDFs, template caching, field access errors

#### Payments 💳
- [**odoo-18-payment-provider-lessons.md**](./odoo-18-payment-provider-lessons.md) - Custom payment provider inline form: QWeb context, rendering_values, provider_sudo

#### Quick Fixes & Patches ⚡
- [**quick-fixes.md**](./quick-fixes.md) - Fast solutions for common Odoo issues across modules, database, views, and APIs

#### Performance (Coming Soon)
- *Planned*

#### Security (Coming Soon)
- *Planned*

## 🚀 Quick Links

**Most Recent Issues:**
1. [Odoo 15: `sale_ids` → `order_ids` en `crm.lead`](./odoo-15-lessons.md#sale_ids-no-existe-en-crmlead----usar-order_ids) - AttributeError en computed fields del CRM
2. [Odoo 15: Campo `related` no recalcula en asignación manual](./odoo-15-lessons.md#issue-1-campo-related-con-storetrue-no-se-recalcula-en-asignación-manual-desde-shell) - Forzar recalculo después de poblar masivo
3. [Odoo 15: Vistas kanban no actualizan sin `-u`](./odoo-15-lessons.md#issue-2-vista-kanban-no-se-actualiza-hasta-hacer--u-del-módulo) - En Odoo.sh usar Upgrade desde Apps
4. [Odoo 18: Payment Provider — KeyError rendering_values](./odoo-18-payment-provider-lessons.md#-error-2-keyerror-rendering_values-en-el-template-qweb) - `rendering_values` no es variable en el template
5. [Odoo 18: Payment Provider — KeyError provider_sudo](./odoo-18-payment-provider-lessons.md#-error-3-keyerror-provider_sudo-en-el-template-qweb) - Hay que inyectarlo manualmente
6. [Odoo 18: Payment Provider — Pantalla gris](./odoo-18-payment-provider-lessons.md#-error-1-pantalla-gris-en-el-checkout-overlay-bloqueante) - No devolver binarios en rendering_values
7. [QWeb Reports: Blank PDF Files](./qweb-reports-lessons.md#-blank-pdf-reports) - Report generates but shows no content
8. [QWeb Reports: Template Caching](./qweb-reports-lessons.md#-template-caching-issues) - Changes not appearing after update
9. [Odoo 19: HTTP Route Type Deprecation](./odoo-19-lessons.md#1-http-route-type-deprecation-typejson--typejsonrpc) - `type='json'` → `type='jsonrpc'`
10. [Odoo 19: View Target Changes](./odoo-19-lessons.md#2-view-target-deprecation-targetinline-no-longer-valid) - `target='inline'` removed

**Quick Fixes:**
- [Module Won't Install](./quick-fixes.md#module-wont-install---dependency-error)
- [Database Connection Pool Exhausted](./quick-fixes.md#database-connection-pool-exhausted)
- [View Not Refreshing](./quick-fixes.md#view-not-refreshing-after-changes)
- [Slow Query - Missing Index](./quick-fixes.md#slow-query---missing-index)

## 📝 Document Structure

Each version document follows this structure:

```
# Odoo X Lessons Learned

## Breaking Changes
- List of backward-incompatible changes

## Common Issues
- Issue 1: Problem description
  - Symptoms
  - Root cause
  - Solution
  - References

## Deprecated Features
- Feature list with migration path

## Performance Optimizations
- Optimization techniques specific to this version

## Security Updates
- Important security considerations

## Widget & API Changes
- Widget changes
- API deprecations
- New features

## References & Resources
- Official documentation
- GitHub issues
- Blog posts
- Stack Overflow threads
```

## 💡 How to Use This Repository

1. **Finding solutions**: Use Ctrl+F to search within documents
2. **Looking up breaking changes**: Check the "Breaking Changes" section
3. **Version comparison**: Compare sections across version files
4. **Quick fixes**: Check [quick-fixes.md](./quick-fixes.md) for immediate solutions
5. **Topic-specific**: Browse by topic (Reports, Performance, Security) for cross-version issues
6. **Adding new lessons**: Follow the document structure and submit a PR

## 🔄 Contributing

When you encounter a new Odoo issue:

1. **Document the problem** with:
   - Version of Odoo
   - Exact symptoms
   - Error messages (if any)
   - Code snippets

2. **Document the solution** with:
   - Root cause analysis
   - Step-by-step fix
   - Code changes
   - Testing verification

3. **Add references** with:
   - GitHub links
   - Blog posts
   - Official docs
   - Stack Overflow

4. **Submit a pull request** to the appropriate version file or topic file

## 📊 Statistics

| Document | Issues Documented | Last Updated |
|----------|-------------------|---------------|
| **By Version** | | |
| Odoo 19 | 5 | 2026-02-22 |
| Odoo 18 | 3 | 2026-01-22 |
| Odoo 18 Payment Providers | 3 | 2026-05-08 |
| Odoo 17 | - | - |
| Odoo 16 | - | - |
| **Odoo 15** | **5** | **2026-06-10** |
| **By Topic** | | |
| QWeb Reports | 4 | 2026-02-26 |
| Quick Fixes | 25+ | 2026-02-23 |

## 🔗 Related Repositories

- [odessa](https://github.com/odoolargotek/odessa) - Odessa logistics ERP (Odoo 15)
- [pignora_server](https://github.com/odoolargotek/pignora_server) - Pignora crypto-collateralized loans (Odoo 19)
- [largotekodoo](https://github.com/odoolargotek/largotekodoo) - Main Largotek ERP implementation
- [Odoo Official](https://github.com/odoo/odoo) - Official Odoo repository

## 📧 Questions?

For questions about specific lessons or to suggest new topics, open an issue in this repository.

---

**Last Updated**: 2026-06-10  
**Maintained by**: Largotek SRL Development Team  
**License**: CC-BY-4.0 (Knowledge sharing)
