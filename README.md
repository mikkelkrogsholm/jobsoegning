# Jobsøgning

Automatiseret jobsøgning på danske jobportaler med Claude Code og Playwright.

## Hvad er det her?

Et sæt Claude Code skills der bruger `playwright-cli` til at navigere danske jobsider, søge efter stillinger og udtrække jobopslag struktureret.

Idéen er simpel: i stedet for manuelt at besøge 4-5 jobportaler hver dag, lad Claude Code gøre det for dig.

## Understøttede jobportaler

| Portal | Status | Skill |
|--------|--------|-------|
| [Jobindex.dk](https://www.jobindex.dk) | Klar | `/jobindex-search` |
| [Jobbank.dk](https://jobbank.dk) | Klar | `/jobbank-search` |
| [Jobdanmark.dk](https://jobdanmark.dk) | Klar | `/jobdanmark-search` |
| [Jobnet.dk](https://jobnet.dk) | Klar | `/jobnet-search` |

## Forudsætninger

- [Claude Code](https://claude.com/claude-code) CLI
- [Playwright CLI](https://github.com/anthropics/claude-code/tree/main/packages/playwright-cli) (`playwright-cli`)

## Sådan bruger du det

### Installér skills

Clone dette repo og link skills ind i dit Claude Code setup:

```bash
git clone git@github.com:mikkelkrogsholm/jobsoegning.git
```

Claude Code finder automatisk skills i `.claude/skills/` mappen.

### Søg efter jobs

Start Claude Code i repo-mappen og brug en skill:

```
/jobindex-search data engineer københavn
```

Eller bed Claude om det i naturligt sprog:

```
Find mig administrative jobs i København oprettet inden for de sidste 7 dage
```

### Hvad får du ud af det?

Strukturerede jobopslag med:
- Jobtitel og link
- Virksomhedsnavn
- Lokation
- Oprettelsesdato
- Kort beskrivelse

## Projektstruktur

```
.
├── .claude/
│   └── skills/
│       ├── jobbank-search/     # Jobbank søgeskill
│       ├── jobdanmark-search/  # Jobdanmark søgeskill
│       ├── jobindex-search/    # Jobindex søgeskill
│       ├── jobnet-search/      # Jobnet søgeskill
│       └── playwright-cli/     # Playwright reference docs
├── CLAUDE.md                   # Instruktioner til Claude Code
├── CONTRIBUTING.md             # Bidragsretningslinjer
├── LICENSE                     # MIT
└── README.md                   # Denne fil
```

## Sådan bidrager du

Bidrag er velkomne! Se [CONTRIBUTING.md](CONTRIBUTING.md) for retningslinjer.

Kort version:
1. Fork repo
2. Opret en branch (`git checkout -b ny-jobportal`)
3. Udforsk en ny jobportal og dokumentér den
4. Opret en skill i `.claude/skills/`
5. Åbn en pull request

## Licens

MIT — se [LICENSE](LICENSE).
