# GeneXus OpenCode Skills

Skills for using GeneXus expertise with OpenCode.

## Included

- **nexa** — GeneXus expert skill, version 1.1.4.
- **gam** — GeneXus Access Manager expert skill, version 1.1.0. GAM declares Nexa as a dependency.

## Structure

```
skills/
├── nexa/
│   └── SKILL.md
└── gam/
    └── SKILL.md
```

> The SKILL.md files reference additional `references/` documentation. This repository currently contains the two supplied skill definitions; reference packs can be added separately when available.

## Usage with OpenCode

Clone this repository on each development machine and expose the `skills` directory to OpenCode according to the OpenCode Skills configuration.

## Versioning

Skill versions are declared in each SKILL.md. Use Git tags/releases or the included status tooling (to be added) to keep multiple machines synchronized.
