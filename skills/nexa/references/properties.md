---
name: properties
description: Common property schema and index for object property references
---

Use this file as the common entry point for property definitions and type semantics

---

# TYPE SEMANTICS
- `boolean`: Flag value
- `string`: Text value
- `integer`: Numeric value without decimals
- `number`: Numeric value with optional decimals
- `enum{…}`: Closed list of literal values
- `ref(…)`: References an existing object of provided types

---

# REFERENCES
Explore object and preferences properties
- Run `python scripts/genexus-catalog.py -h` for full usage
- Relevant catalogs: `objects`, `preferences`, `properties`