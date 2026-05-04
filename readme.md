# AI-Skills (`mz` plugin)

Osobní kolekce skillů pro [Claude Code](https://claude.com/claude-code) od Michala Zemka. Tento repozitář funguje jako Claude Code **plugin marketplace** — můžeš si ho přidat a získat všechny skilly jednou instalací.

## Instalace

V Claude Code:

```
/plugin marketplace add michalzemek/AI-Skills
/plugin install mz@AI-Skills
```

Po instalaci jsou skilly dostupné automaticky (auto-invocation) nebo přes prefix `/mz:`.

## Dostupné skilly

| Skill | Co dělá |
|-------|---------|
| [`/mz:check-task-specificity`](skills/check-task-specificity/SKILL.md) | Před implementací ohodnotí specifičnost zadání dle 5 dimenzí (viz: arxiv 2504.20196 https://arxiv.org/abs/2504.20196 ) a klade cílené otázky pokud je vágní. |
| [`/mz:analyze-error-log`](skills/analyze-error-log/SKILL.md) | Analyzuje aplikační log (Serilog formát), seskupí chyby dle priority, vrátí strukturovaný report s doporučeními. |

## Vyvolání

**Automaticky** — Claude Code skill aktivuje sám podle popisu z frontmatteru SKILL.md, když matchuje uživatelův dotaz.

**Manuálně** — slash příkazem:

```
/mz:analyze-error-log C:\logs\app-2026-05-04.log
/mz:check-task-specificity
```

## Struktura repa

```
AI-Skills/
├── .claude-plugin/
│   ├── marketplace.json     # marketplace definice
│   └── plugin.json          # manifest pluginu mz
├── skills/
│   ├── analyze-error-log/
│   │   └── SKILL.md
│   └── check-task-specificity/
│       └── SKILL.md
└── readme.md
```

## Přidání nového skillu

1. Vytvoř `skills/<nazev-skillu>/SKILL.md`.
2. Na začátek dej YAML frontmatter:
   ```yaml
   ---
   name: nazev-skillu
   description: Stručný popis kdy a proč skill použít — Claude podle něj rozhoduje o auto-invocation.
   ---
   ```
3. Pod frontmatter napiš instrukce pro skill (markdown).
4. Přidej řádek do tabulky v této README a otestuj lokálně:
   ```
   /plugin marketplace add P:\GITHUB-MICHALZEMEK\AI-Skills
   /plugin install mz@AI-Skills
   /reload-plugins
   ```

## Licence

MIT — viz [LICENSE](LICENSE).
