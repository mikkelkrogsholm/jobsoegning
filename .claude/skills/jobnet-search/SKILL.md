---
name: jobnet-search
description: Search for jobs on jobnet.dk (Denmark's official public job portal by STAR). Use when the user wants to find jobs on Jobnet, search government job listings. Supports filtering by keyword, region, occupation area, work hours, employment duration, and more.
allowed-tools: Bash(playwright-cli:*)
argument-hint: [search query]
---

# Jobnet.dk Job Search

This skill searches for jobs on **jobnet.dk**, Denmark's official public job portal operated by STAR (Styrelsen for Arbejdsmarked og Rekruttering). Jobnet aggregates public sector jobs and many private sector positions, making it a comprehensive resource for job seekers in Denmark.

## Prerequisites

Before running a search, ensure you have an open browser session:

```bash
# Headless (default, supports parallel execution)
playwright-cli -s=jobnet open https://jobnet.dk/find-job

# Headed (visible browser, for debugging)
playwright-cli -s=jobnet open https://jobnet.dk/find-job --headed
```

**Named session `-s=jobnet`** allows this skill to run in parallel with other job search skills. Use `-s=jobnet` on all subsequent commands.

## Workflow

### Step 1: Dismiss Cookie Banner

On first visit, dismiss the cookie banner:

```bash
playwright-cli -s=jobnet run-code 'async page => {
  try {
    await page.waitForTimeout(1500);
    await page.getByRole("button", { name: "Kun nødvendige cookies" }).click({ timeout: 3000 });
    await page.waitForTimeout(1000);
  } catch (e) {
    console.log("No cookie dialog or already dismissed");
  }
}'
```

**Note:** Use try/catch because the dialog only appears on first visit.

### Step 2: Navigate to Search Results

Build the search URL with appropriate filters and navigate directly. This is more reliable than using the UI filters, which use React components that can be difficult to automate.

**Example URL construction:**

```
https://jobnet.dk/find-job?searchString=systemudvikler&regions=HovedstadenOgBornholm&occupationAreas=110000&workHoursType=FullTime&employmentDurationType=Permanent&orderType=CreationDate
```

Navigate using playwright-cli:

```bash
playwright-cli -s=jobnet goto "https://jobnet.dk/find-job?searchString=YOUR_QUERY&regions=REGION_CODE"
```

Wait for results to load (usually 1-2 seconds).

### Step 3: Extract Job Listings

Use the following JavaScript code to extract job data from the page:

```bash
playwright-cli -s=jobnet run-code '
async page => {
  await page.waitForTimeout(2000);
  const jobs = await page.evaluate(() => {
    return Array.from(document.querySelectorAll("article")).map(a => {
    const title = a.querySelector("h3 p")?.textContent?.trim();
    const titleLink = a.querySelector("h3")?.closest("a");
    const company = titleLink?.querySelectorAll("p")?.[1]?.textContent?.trim();
    const url = titleLink?.getAttribute("href");
    const times = a.querySelectorAll("time");
    const deadline = times[0]?.textContent?.trim() || "";
    const published = times[1]?.textContent?.trim() || "";
    const infoList = a.querySelector(".jobadcard_job-info-list__aA30l");
    const spans = infoList?.querySelectorAll("span") || [];
    const workHours = spans[0]?.textContent?.trim() || "";
    const location = spans[1]?.textContent?.trim() || "";
    const snippet = a.querySelector(".jobadcard_description-paragraph__VTOnj")?.textContent?.trim() || "";
    return {
      title,
      company,
      url: "https://jobnet.dk" + url,
      deadline,
      published,
      workHours,
      location,
      snippet
    };
  });
  });
  return JSON.stringify(jobs, null, 2);
}
'
```

**Note on pagination:** Jobnet uses infinite scroll with a "Indlæs flere job" (Load more jobs) button. The initial page load shows 10 results. To load more:

```bash
playwright-cli -s=jobnet run-code 'async page => {
  try {
    await page.getByRole("button", { name: /Indlæs flere job/ }).click({ timeout: 3000 });
    await page.waitForTimeout(2000);
    console.log("Loaded more jobs");
  } catch (e) {
    console.log("No more jobs to load");
  }
}'
```

Repeat as needed, then re-extract jobs. The list grows cumulatively (10 → 20 → 30).

### Step 4: Present Results

Format the extracted JSON into a readable summary for the user, including:
- Job title
- Company name
- Location
- Work hours (Fuldtid/Deltid)
- Application deadline
- Published date
- Brief snippet
- Direct URL to full job posting

Present 5-10 jobs per batch unless the user requests more.

## URL Reference

### Base URL
```
https://jobnet.dk/find-job
```

### Full URL Pattern
```
https://jobnet.dk/find-job?searchString={query}&regions={region}&occupationAreas={code}&occupationGroups={code}&employmentDurationType={type}&workHoursType={type}&jobAnnouncementType={type}&orderType={type}&postalCode={code}&kmRadius={km}&workplaceFilter=NonFixed
```

### Query Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `searchString` | Free-text search query | `systemudvikler` |
| `regions` | Region filter (comma-separated for multiple) | `HovedstadenOgBornholm,Midtjylland` |
| `postalCode` | Postal code for geographic search | `2100` |
| `kmRadius` | Distance radius from postal code (km) | `50` (default) |
| `workplaceFilter` | No fixed workplace filter | `NonFixed` |
| `occupationAreas` | Fagområde code (comma-separated) | `110000` |
| `occupationGroups` | Kategori code (comma-separated) | `110050` |
| `workHoursType` | Working hours type | `FullTime`, `PartTime`, `PartTimeHourly` |
| `employmentDurationType` | Contract duration | `Permanent`, `Temporary` |
| `jobAnnouncementType` | Employment conditions | `Fleksjob`, `HotJob`, `HandicapNoObstacle`, `SuitableForEarlyRetirees`, `SuitableForDisabilityPensioners` |
| `orderType` | Sort order | `BestMatch`, `CreationDate` (default), `ApplicationDate` |

## Filter Reference

### Regions (regions)

| Display Name | Code |
|--------------|------|
| Hovedstaden og Bornholm | `HovedstadenOgBornholm` |
| Midtjylland | `Midtjylland` |
| Syddanmark | `Syddanmark` |
| Øvrige Sjælland | `OevrigeSjaelland` |
| Nordjylland | `Nordjylland` |

**Note:** Multiple regions can be selected by comma-separating codes: `regions=HovedstadenOgBornholm,Midtjylland`

### Geographic Search Modes

Jobnet supports **three mutually exclusive** geographic filter modes:

1. **Region-based** — Use `regions` parameter for broad regional filtering
2. **Postal code + radius** — Use `postalCode` + `kmRadius` for precise location search
3. **No fixed workplace** — Use `workplaceFilter=NonFixed` for remote/flexible jobs

Do not combine these modes in a single search.

### Fagområde (occupationAreas)

**Confirmed codes:**
| Display Name | Code |
|--------------|------|
| It og teleteknik | `110000` |

**Other fagområder** (codes need discovery):
- Akademisk arbejde
- Ledelse
- Kontor/administration/regnskab/finans
- Sundhed/omsorg/personlig pleje
- Elever
- Pædagogisk/socialt/kirkeligt arbejde
- Salg/indkøb/markedsføring
- Undervisning/vejledning
- Hotel/restauration/køkken/kantine
- Rengøring/ejendomsservice/renovation
- Bygge og anlæg
- Transport/post/lager/maskinførerr
- Landbrug/skovbrug/gartneri/fiskeri/dyrepleje
- Jern/metal/auto
- Vagt/sikkerhed/overvågning
- Industriel produktion
- Medie/kultur/turisme/idræt/underholdning
- Design/formgivning/grafisk arbejde
- Nærings-/nydelsesmiddel
- Tekstil/beklædning
- Træ/møbel/glas/keramik

**Kategori (occupationGroups):**
- `110050` — Systemudvikling (under It og teleteknik)
- Additional codes need discovery

**Note:** `occupationGroups` is hierarchical and depends on `occupationAreas`. To discover more codes, inspect the network requests when using the jobnet.dk filter UI.

### Work Hours (workHoursType)

| Display Name | Code |
|--------------|------|
| Fuldtid | `FullTime` |
| Deltid | `PartTime` |
| Timelønnet | `PartTimeHourly` |

### Employment Duration (employmentDurationType)

| Display Name | Code |
|--------------|------|
| Fast | `Permanent` |
| Tidsbegrænset | `Temporary` |

### Job Announcement Type (jobAnnouncementType)

| Display Name | Code |
|--------------|------|
| Fleksjob | `Fleksjob` |
| Hotteste job | `HotJob` |
| Handicap er ikke et problem | `HandicapNoObstacle` |
| Egnet til efterlønsmodtagere | `SuitableForEarlyRetirees` |
| Egnet til førtidspensionister | `SuitableForDisabilityPensioners` |

### Sort Order (orderType)

| Display Name | Code |
|--------------|------|
| Relevans | `BestMatch` |
| Oprettelsesdato | `CreationDate` (default) |
| Ansøgningsfrist | `ApplicationDate` |

## Tips

### Building Effective Search URLs

1. **Start simple:** Begin with just `searchString` and `regions`
2. **Add filters incrementally:** Layer on `workHoursType`, `employmentDurationType`, etc. as needed
3. **Use postal code for precise location:** Combine `postalCode=2100&kmRadius=25` for jobs near Østerbro within 25km
4. **Combine filters:** Multiple filters narrow results: `searchString=udvikler&regions=HovedstadenOgBornholm&workHoursType=FullTime&employmentDurationType=Permanent`

### Search Query Tips

- **No quotes needed:** Unlike Jobindex, Jobnet doesn't require quotes for exact matches
- **Use Danish terms:** "systemudvikler" works better than "system developer"
- **Try variations:** Search both "IT-konsulent" and "konsulent" separately if needed

### Handling Pagination

Initial load shows 10 jobs. To load more, use the same pattern as Step 3:

```bash
playwright-cli -s=jobnet run-code 'async page => {
  try {
    await page.getByRole("button", { name: /Indlæs flere job/ }).click({ timeout: 3000 });
    await page.waitForTimeout(2000);
  } catch (e) {
    console.log("No more jobs to load");
  }
}'
```

Then re-run the extraction code. The article list grows cumulatively (10 → 20 → 30+).

### Extracting Full Job Details

Job detail pages have URL pattern: `https://jobnet.dk/find-job/{uuid}`

To extract full job descriptions:

1. Click a job card link from the search results
2. Wait for detail page to load
3. Extract content using appropriate selectors

**Example detail extraction** (selectors may need adjustment):

```bash
playwright-cli -s=jobnet run-code '
async page => {
  const title = document.querySelector("h1")?.textContent?.trim();
  const description = document.querySelector("[class*=description]")?.innerHTML;
  return JSON.stringify({ title, description }, null, 2);
}
'
```

## Known Issues and Pitfalls

### 1. CSS Class Names May Change

CSS class names include hash suffixes (e.g., `jobadcard_job-info-list__aA30l`) that may change between deployments. If extraction fails:

- Take a fresh snapshot and inspect the HTML structure
- Update selectors to use semantic queries where possible
- Test the extraction code before relying on results

### 2. React-Select Components

Region and occupation filters use React-select components, which are difficult to automate reliably. **Always prefer URL parameters over UI interactions** for setting filters.

### 3. Fagområde/Kategori Codes

Most occupation area and group codes are undocumented. To discover codes:

1. Open browser DevTools (F12)
2. Go to Network tab
3. Select filters in the jobnet.dk UI
4. Inspect the `find-job` request URL to see the codes
5. Document new codes in this skill file

### 4. Hierarchical Filter Dependencies

The "Kategori" (occupationGroups) filter is disabled until a "Fagområde" (occupationAreas) is selected. They form a hierarchy. When using URL parameters, you can include both, but ensure consistency.

### 5. No URL-Based Pagination

Unlike Jobindex, Jobnet does not use URL parameters for pagination (no `?page=2`). You must:
- Click "Indlæs flere job" button repeatedly
- Re-extract the full article list after each click
- The list grows cumulatively (10 → 20 → 30 jobs)

### 6. Geographic Filter Conflicts

Do not mix geographic filter types:
- ❌ Bad: `regions=HovedstadenOgBornholm&postalCode=2100`
- ✅ Good: `regions=HovedstadenOgBornholm` OR `postalCode=2100&kmRadius=50`

### 7. Initial Page Load Time

Search results may take 1-3 seconds to load. After navigating to a search URL, wait briefly before extracting jobs. If extraction returns empty results, wait longer and retry.

## Example Searches

### IT Developer Jobs in Copenhagen Area

```bash
playwright-cli -s=jobnet goto "https://jobnet.dk/find-job?searchString=systemudvikler&regions=HovedstadenOgBornholm&workHoursType=FullTime&employmentDurationType=Permanent"
```

### Jobs Near Postal Code 2100 (Østerbro) Within 25km

```bash
playwright-cli -s=jobnet goto "https://jobnet.dk/find-job?searchString=udvikler&postalCode=2100&kmRadius=25&workHoursType=FullTime"
```

### Remote/Flexible Jobs (No Fixed Workplace)

```bash
playwright-cli -s=jobnet goto "https://jobnet.dk/find-job?searchString=konsulent&workplaceFilter=NonFixed&workHoursType=FullTime"
```

### IT Jobs Sorted by Application Deadline

```bash
playwright-cli -s=jobnet goto "https://jobnet.dk/find-job?searchString=IT&occupationAreas=110000&orderType=ApplicationDate"
```


