# Bidrag til Jobsøgning

Tak fordi du vil bidrage!

## Sådan tilføjer du en ny jobportal

### 1. Udforsk portalen

Brug `playwright-cli` til at udforske portalen systematisk:

```bash
playwright-cli -s=mitsite open "https://example-jobsite.dk" --headed
```

Dokumentér:
- URL-mønstre (base URL, query params, path segments, paginering)
- Søgeformular-struktur
- Filtertyper og hvordan de påvirker URL
- CSS-selektorer for jobopslag
- JavaScript-kode til dataudtræk
- Popups/cookie-consent og hvordan de lukkes
- Kendte problemer og workarounds

### 2. Skriv udforskningsnoter

Opret `notes/<portal>-exploration.md` efter samme format som `notes/jobindex-exploration.md`.

Tag screenshots og gem dem i `notes/screenshots/<portal>-*.png`.

### 3. Opret en skill

Opret `.claude/skills/<portal>-search/SKILL.md` med:
- Frontmatter (name, description, allowed-tools)
- Workflow: popup-håndtering, URL-navigation, dataudtræk
- Kendte faldgruber og workarounds

Se `.claude/skills/jobindex-search/SKILL.md` som reference.

### 4. Test

Kør din skill via Claude Code og verificér at:
- Popups lukkes korrekt
- Søgeresultater udtrækkes rigtigt
- Paginering virker
- Filtre fungerer

## Code style

- Skills skrives i Markdown (SKILL.md)
- Udforskningsnoter i Markdown
- JavaScript-uddrag skal være korrekte og kørbare via `playwright-cli run-code`

## Pull requests

- Beskriv hvad du har tilføjet/ændret
- Inkludér screenshots fra din udforskning
- Sørg for at eksisterende skills stadig virker
