---
name: analyze-error-log
description: Analyzuje aplikační log soubor (Serilog formát) a vrací strukturovaný report chyb, výjimek a varování s prioritami a doporučeními. Použij když uživatel poskytne cestu k log souboru a chce přehled problémů.
---

# Skill: analyze-error-log

Analyzuje aplikační log soubor a poskytne strukturovaný přehled všech chyb, výjimek a varování.

## Argumenty
- `$ARGUMENTS` — cesta k log souboru (povinná)

## Instrukce

### 1. Ověření vstupu
- Pokud `$ARGUMENTS` je prázdný, vypiš: "Použití: /analyze-error-log <cesta_k_log_souboru>" a skonči.
- Ověř existenci souboru pomocí Read tool. Pokud neexistuje, informuj uživatele a skonči.

### 2. Vyhledání chyb pomocí Grep
Spusť **paralelně** tyto Grep vyhledávání na souboru z `$ARGUMENTS`:

1. **` ERR\]`** — hlavní chybové záznamy (Serilog formát: `[dd-MM-yyyy HH:mm:ss ERR]`)
2. **` FTL\]`** — fatální chyby
3. **`Exception`** — výjimky ve stack traces (output_mode: "count" pro odhad rozsahu)
4. **` WRN\]`** — varování

**Poznámka k formátu:** Mindtra používá Serilog s formátem `[dd-MM-yyyy HH:mm:ss LEVEL]`. Pattern ` ERR]` (s mezerou před) spolehlivě matchuje úroveň uvnitř závorky s datumem. Alternativně pokud log používá formát `[ERR]` samostatně, hledej `\[ERR\]`.

Nejprve spusť všechny 4 vyhledávání s `output_mode: "count"` pro rychlý přehled počtů.
Poté spusť ERR a FTL s `output_mode: "content"`, `-A: 15`, `-B: 2` pro zachycení stack traces.

### 3. Načtení kontextu stack traces
Pro každý nalezený `[ERR]` a `[FTL]` záznam:
- Použij Grep s parametrem `-A: 15` a `-B: 2` pro zachycení celého stack trace za chybovou zprávou.
- Identifikuj typ výjimky (např. `System.ArgumentException`, `System.NullReferenceException`).
- Identifikuj místo vzniku — hledej pattern `at Namespace.Class.Method(...)` nebo `in FilePath:line N`.

### 4. Seskupení a klasifikace
Seskup nalezené chyby podle:
- **Typ výjimky** (celý qualified name)
- **Místo vzniku** (metoda + řádek)

Klasifikuj prioritu:
| Priorita | Kritéria |
|----------|----------|
| KRITICKÁ | `[FTL]`, unhandled exceptions, `NullReferenceException`, `StackOverflowException`, `OutOfMemoryException`, databázové chyby |
| STŘEDNÍ | `[ERR]` s opakovaným výskytem (3+), timeout chyby, chyby externích služeb |
| NÍZKÁ | Jednorázové `[ERR]`, validační chyby, expected exceptions |
| INFO | `[WRN]`, bezpečnostní události (Access Denied, PermissionDenied, Unauthorized) |

### 5. Oddělení bezpečnostních událostí
Zvlášť vyčleň záznamy obsahující:
- `Access Denied`, `AccessDenied`
- `PermissionDenied`, `Forbidden`
- `Unauthorized`, `401`, `403`
- `AuthenticationException`, `AuthorizationException`

Tyto NEJSOU bugy — reportuj je v samostatné sekci "Bezpečnostní události".

### 6. Výstupní report

Vypiš report v **přesně tomto formátu** — bez markdown code blocků kolem celého reportu, přímo jako text s nadpisy a tabulkami:

```
Analýza logu: <název_souboru>

  Souhrn

  - Celkem [ERR] záznamů: X
  - Celkem [FTL] záznamů: X
  - Celkem [WRN] záznamů: X
  - Unikátních typů výjimek: X
  - Analyzovaný soubor: <cesta>

  ---
  Kritické chyby

  <Pokud žádné:> Žádné [FTL] záznamy. Žádné NullReferenceException, OutOfMemoryException ani unhandled exceptions.

  <Pokud existují: použij stejný formát jako u "Chyby – střední priorita" níže>

  ---
  Chyby – střední priorita

  <Pro každou skupinu chyb se stejným typem výjimky a místem vzniku:>

  X. <Stručný název problému> — <Typ výjimky> (Nx ERR)

  ┌──────────┬──────────────┬─────────────┬────────────────────┬─────────────┐
  │ <Entita> │    Název     │ Výskytů ERR │   <Kontext job>    │    Časy     │
  ├──────────┼──────────────┼─────────────┼────────────────────┼─────────────┤
  │ <id>     │ '<název>'    │ <n>×        │ <job info>         │ <čas–čas>   │
  └──────────┴──────────────┴─────────────┴────────────────────┴─────────────┘

  <Typ výjimky>: <Zpráva výjimky>
    at <Namespace.Class.Method()>
       in <Soubor.cs>:line <N>
    at <Namespace.Class.Method()>
       in <Soubor.cs>:line <N>

  Popis: <Lidsky srozumitelný popis co se stalo, proč k chybě došlo, jak se projevila>

  <Vedlejší efekt: pokud existuje, popiš dopad na uživatele nebo systém>

  ---
  Chyby – nízká priorita

  ┌─────┬───────────────────────────┬──────────────────────────────────────────┬─────────┬────────────────────┐
  │  #  │         Typ chyby         │                  Zpráva                  │ Výskytů │        Čas         │
  ├─────┼───────────────────────────┼──────────────────────────────────────────┼─────────┼────────────────────┤
  │ 1   │ <Typ výjimky/chyby>       │ "<Zpráva>"                               │ <n>×    │ <časy>             │
  └─────┴───────────────────────────┴──────────────────────────────────────────┴─────────┴────────────────────┘

  Popis: <Popis každé chyby s nízkou prioritou>

  ---
  Bezpečnostní události

  <Pokud žádné:> Žádné záznamy s Access Denied, Unauthorized, 403, AuthorizationException apod.

  <Pokud existují: tabulka s časem, typem události, uživatelem/IP>

  ---
  Varování [WRN] — přehled

  ┌───────────────────────────────────────────┬───────────────┬──────────────────────────────────────────────┐
  │               Typ varování                │ Počet výskytů │                    Zdroj                    │
  ├───────────────────────────────────────────┼───────────────┼──────────────────────────────────────────────┤
  │ <Popis varování>                          │ <n>×          │ <Třída/Namespace>                            │
  └───────────────────────────────────────────┴───────────────┴──────────────────────────────────────────────┘

  ---
  Doporučení

  1. <Název problému> (PRIORITA: KRITICKÁ/STŘEDNÍ/NÍZKÁ)

  <Podrobný popis problému a konkrétní doporučení k opravě. Uveď konkrétní soubory a řádky kde je třeba opravit.>
  - <Konkrétní krok 1>
  - <Konkrétní krok 2>
  - <Konkrétní krok 3>

  2. <Název dalšího problému> (NÍZKÁ)

  <Popis a doporučení>
```

### 7. Pravidla formátování tabulek
- Tabulky vykresluj pomocí Unicode box-drawing znaků: `┌ ┬ ┐ ├ ┼ ┤ └ ┴ ┘ │ ─`
- Šířku sloupců přizpůsob nejdelšímu obsahu v daném sloupci
- Každou skupinu chyb odděluj `---` oddělovačem
- Stack traces vypisuj bez code blocku, pouze s odsazením mezerami
- Názvy entit v tabulkách obaluj do apostrofů: `'název'`
- Časové rozsahy formátuj jako `HH:mm–HH:mm`
- Pro chyby se stejným stack trace ale různými instancemi (různé ID entity) vytvoř jeden nadpis skupiny + tabulku instancí

### 8. Pravidla analýzy
- **NIKDY nečti celý log soubor** pomocí Read — vždy použij Grep pro cílené vyhledávání.
- Pokud je log velmi velký (tisíce chyb), omez výstup na top 20 nejčastějších chyb a uveď celkový počet.
- Časové údaje extrahuj z řádků logu (Mindtra formát: `[dd-MM-yyyy HH:mm:ss LEVEL]`, např. `[11-02-2026 13:23:26 ERR]`).
- Report piš česky, ale technické názvy (typy výjimek, metody, soubory) ponech v originále.
- Pokud se stejný typ výjimky vyskytuje na více místech, seskup je do jedné položky se seznamem výskytů.
- V sekci "Popis" vždy vysvětli **proč** k chybě došlo (příčina), **jak** se projevila (symptom) a **co to znamená** pro systém/uživatele.
- V sekci "Doporučení" uveď vždy konkrétní soubor a řádek kde je třeba opravit, ne jen obecné rady.
