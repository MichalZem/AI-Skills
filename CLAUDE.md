# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Co tento repozitář je

Toto **není aplikační kód** — je to **Claude Code plugin marketplace**. Repo definuje jeden plugin (`mz`) a hostí kolekci skillů, které si uživatelé instalují do svého Claude Code přes:

```
/plugin marketplace add michalzemek/AI-Skills
/plugin install mz@AI-Skills
```

Po instalaci jsou skilly volatelné jako `/mz:<skill-name>` nebo se aktivují automaticky podle `description` z frontmatteru SKILL.md.

## Architektura

Tři vrstvy konfigurace, které spolu musí ladit:

1. **`.claude-plugin/marketplace.json`** — definice marketplace. `name: "AI-Skills"`, registruje jeden plugin `mz` se `source: "./"` (plugin žije v rootu repa).
2. **`.claude-plugin/plugin.json`** — manifest pluginu `mz`. Pole `name` zde **musí přesně odpovídat** `plugins[].name` v marketplace.json — jinak instalace selže nebo se skilly načtou pod jiným namespace.
3. **`skills/<skill-name>/SKILL.md`** — jednotlivé skilly. Claude Code je auto-discoveruje na základě YAML frontmatteru (`name`, `description`). Bez frontmatteru skill nebude správně rozpoznán.

Jméno složky pod `skills/` se používá jako součást slash příkazu (`/mz:<složka>`) a má matchovat pole `name` ve frontmatteru.

## Práce v tomto repu

### Přidání nového skillu

1. `skills/<nazev>/SKILL.md` s YAML frontmatterem:
   ```yaml
   ---
   name: nazev
   description: Konkrétní popis kdy skill použít — Claude podle něj rozhoduje o auto-invocation, takže má být specifický k situaci, ne ke schopnosti.
   ---
   ```
2. Tělo skillu pod frontmatter (markdown — instrukce pro Claude když je skill aktivován).
3. Přidat řádek do tabulky „Dostupné skilly" v `readme.md`.
4. Otestovat lokálně (viz níže).

### Lokální test změn

Po úpravě SKILL.md, plugin.json nebo marketplace.json:

```
/plugin marketplace add P:\GITHUB-MICHALZEMEK\AI-Skills   # poprvé
/plugin install mz@AI-Skills                              # poprvé
/reload-plugins                                           # po každé změně
```

Pak vyvolej skill manuálně (`/mz:<nazev>`) nebo zkus přirozený dotaz, který odpovídá `description`, abys ověřil auto-invocation.

### Validace JSON

```powershell
Get-Content .claude-plugin/marketplace.json -Raw | ConvertFrom-Json
Get-Content .claude-plugin/plugin.json      -Raw | ConvertFrom-Json
```

Oba musí projít bez chyby — Claude Code marketplace odmítá nevalidní manifesty bez detailní chybové zprávy.

## Konvence

- **Jazyk obsahu:** SKILL.md jsou psané česky (instrukce + výstupní formát). Frontmatter `description` taky česky, protože uživatelé i auto-invocation prompty jsou česky.
- **Názvy skillů:** kebab-case (`analyze-error-log`, ne `analyzeErrorLog`).
- **Plugin namespace:** `mz` (lowercase). Plugin name v `plugin.json` se nemění bez koordinace s `marketplace.json` a všemi odkazy v `readme.md`.
- **Žádný build/test runtime** — repo obsahuje pouze markdown a JSON. Žádný `package.json`, žádné CI. Verifikace = manuální install + vyvolání skillu.

## Co existující skilly dělají

- **`analyze-error-log`** — analýza Serilog souborů (`[dd-MM-yyyy HH:mm:ss LEVEL]` formát), Grep-based extrakce ERR/FTL/WRN, klasifikace priorit, výstup ve fixed-format reportu s Unicode tabulkami. Kdykoli upravuješ tento skill, zachovej formát výstupu — uživatelé na něj mohou parsovat.
- **`check-task-specificity`** — gating skill před implementací. Hodnotí zadání dle 5 dimenzí z arxiv 2504.20196 (User Intent, Specifics, Operationalization, Localization, Codebase Context), skóre 0–10, klade jednu cílenou otázku na nejslabší dimenzi. Pravidlo „jen jedna otázka najednou" je záměrné — neporušovat.
