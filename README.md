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

### Odoo 19 🆕
- [**odoo-19-lessons.md**](./odoo-19-lessons.md) - Complete migration guide from Odoo 18 to 19 with breaking changes and solutions

### Odoo 18
- [**odoo-18-lessons.md**](./odoo-18-lessons.md) - Issues, breaking changes, and solutions for Odoo 18

### Odoo 17
- *Planned*

### Odoo 16
- *Planned*

### Odoo 15
- *Planned*

### Odoo 14 (Legacy)
- *Planned*

## 🚀 Quick Links

**Most Recent Issues:**
1. [Odoo 19: HTTP Route Type Deprecation](./odoo-19-lessons.md#1-http-route-type-deprecation-typejson--typejsonrpc) - `type='json'` → `type='jsonrpc'`
2. [Odoo 19: View Target Changes](./odoo-19-lessons.md#2-view-target-deprecation-targetinline-no-longer-valid) - `target='inline'` removed
3. [Odoo 19: Model Description Required](./odoo-19-lessons.md#3-model-_description-attribute-now-required) - All models need `_description`
4. [Odoo 18: Chatter Widget Migration](./odoo-18-lessons.md#chatter-widget-migration) - Form rendering breaks with legacy chatter syntax
5. [Odoo 18: Activity Field Naming Conflict](./odoo-18-lessons.md#activity-field-naming-conflict) - `activity_ids` conflicts with mail.activity.mixin

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
4. **Adding new lessons**: Follow the document structure and submit a PR

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

4. **Submit a pull request** to the appropriate version file

## 📊 Statistics

| Version | Issues Documented | Last Updated |
|---------|-------------------|---------------|
| Odoo 19 | 5 | 2026-02-22 |
| Odoo 18 | 3 | 2026-01-22 |
| Odoo 17 | - | - |
| Odoo 16 | - | - |
| Odoo 15 | - | - |
| Odoo 14 | - | - |

## 🔗 Related Repositories

- [pignora_server](https://github.com/odoolargotek/pignora_server) - Pignora crypto-collateralized loans (Odoo 19)
- [largotekodoo](https://github.com/odoolargotek/largotekodoo) - Main Largotek ERP implementation
- [Odoo Official](https://github.com/odoo/odoo) - Official Odoo repository

## 📧 Questions?

For questions about specific lessons or to suggest new topics, open an issue in this repository.

---

**Last Updated**: 2026-02-22  
**Maintained by**: Largotek SRL Development Team  
**License**: CC-BY-4.0 (Knowledge sharing)
