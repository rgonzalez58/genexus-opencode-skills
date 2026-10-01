# GeneXus OpenCode Skills

Repositorio central de Agent Skills de GeneXus para OpenCode.

## Skills incluidos

- **nexa** — GeneXus expert skill, version 1.1.4.
- **gam** — GeneXus Access Manager expert skill, version 1.1.0.

## Estructura

```text
skills/
├── nexa/
│   ├── SKILL.md
│   ├── references/
│   └── scripts/
└── gam/
    ├── SKILL.md
    └── references/
```

Las referencias y scripts forman parte de los Skills y deben mantenerse junto con sus respectivos `SKILL.md`.

## Sincronización entre máquinas

La idea es mantener este repositorio como fuente central y sincronizarlo con Git en cada instalación de OpenCode.

## Versiones

Las versiones se mantienen en el metadata de cada `SKILL.md`.
