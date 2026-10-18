# MyVibingSkills

Colección de skills y agentes de Claude Code: los que ya tengo y los que vaya creando. Sirve para compartirlos y mantenerlos actualizados.

## Estructura

```
MyVibingSkills/
├── skills/                 una carpeta por skill
│   ├── planning/
│   │   ├── SKILL.md        (obligatorio)
│   │   ├── scripts/        (opcional)
│   │   └── resources/      (opcional)
│   └── ...
├── agents/                 subagentes que usan algunas skills (un .md por agente)
└── templates/
    └── skill-template/     plantilla para crear nuevas skills
```

`SKILL.md` (con esa mayúscula exacta) empieza con este encabezado:

```markdown
---
name: mi-skill
description: Cuándo y para qué usar esta skill
---
```

- `description` es lo que Claude lee para decidir si usa la skill: explica bien cuándo aplica.
- El nombre de la carpeta debe coincidir con `name` (minúsculas y guiones).

## Contenido

### Skills

| Skill | Qué hace | Modelo |
| --- | --- | --- |
| `/planning` | Planifica un cambio y espera confirmación antes de implementarlo | Opus 5.5 · medium |
| `/brainstorming` | Discute y pule una idea contigo, sin implementar nada | Opus 5.5 · medium |
| `/autosync` | Commits semánticos, push, PR y auditoría de una vez; para ante cualquier duda | Sonnet 5.5 · medium |
| `/merger` | Mergea las PR listas y borra las ramas ya mergeadas | Sonnet 5.5 · medium |
| `/audit` | Auditoría profunda de solo lectura: bugs, rendimiento, seguridad | Opus 5.5 · high |
| `/improve-ui` | Revisa y mejora una UI con reglas UX concretas | Sonnet 5.5 · medium |
| `/fix` | Bucle plan → implementación → PR → auditoría hasta dejarlo limpio | Sonnet 5.5 · medium |

### Agentes

| Agente | Modelo | Lo usan |
| --- | --- | --- |
| `fix-planner` | Opus 5.5 · medium | `/fix` |
| `fix-implementer` | Sonnet 5.5 · medium | `/fix` |
| `pr-auditor` | Opus 5.5 · medium | `/fix`, `/autosync` |
| `test-runner` | Sonnet 5.5 · medium | `/planning` |

## Instalación

Claude Code carga las skills de usuario desde `~/.claude/skills/` y los agentes desde `~/.claude/agents/`. Clona el repo en cualquier sitio y copia ambas carpetas:

```bash
git clone <url-del-repo> ~/MyVibingSkills
```

```bash
mkdir -p ~/.claude/skills ~/.claude/agents
```

```bash
cp -r ~/MyVibingSkills/skills/* ~/.claude/skills/
```

```bash
cp ~/MyVibingSkills/agents/*.md ~/.claude/agents/
```

Para actualizar: `git pull` en el repo y repetir las dos copias.

Una sola skill en un proyecto: copiarla a `.claude/skills/<nombre>/SKILL.md` dentro de ese proyecto (y sus agentes a `.claude/agents/`).

## Añadir una skill nueva

1. Copia `templates/skill-template/` a `skills/<nombre-de-la-skill>/`.
2. Edita `name` y `description` en su `SKILL.md`.
3. Si necesita un agente, añádelo en `agents/` y a la tabla de arriba.
4. Commit y push.
