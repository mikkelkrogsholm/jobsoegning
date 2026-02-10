---
name: jobbank-search
description: Search for jobs on jobbank.dk (Akademikernes Jobbank). Use when the user wants to find academic/highly educated positions, search for jobs on Jobbank. Supports filtering by keyword, location, education, work area, industry, job type, remote work, and pagination.
allowed-tools: Bash(playwright-cli:*)
argument-hint: [search query]
---

# Jobbank.dk Job Search

Search for academic and highly educated positions on Akademikernes Jobbank using playwright-cli.

## Prerequisites

Before searching, ensure you have an open browser session:

```bash
# Headless (default, supports parallel execution)
playwright-cli -s=jobbank open https://jobbank.dk/job/

# Headed (visible browser, for debugging)
playwright-cli -s=jobbank open https://jobbank.dk/job/ --headed
```

**Named session `-s=jobbank`** allows this skill to run in parallel with other job search skills. Use `-s=jobbank` on all subsequent commands.

## Workflow

### Step 1: Dismiss Cookie Consent (Fresh Sessions Only)

On first visit, dismiss the cookie dialog:

```bash
playwright-cli -s=jobbank run-code 'async page => {
  try {
    await page.waitForTimeout(1500);
    await page.getByRole("button", { name: "Accepter valgte" }).click({ timeout: 3000 });
    await page.waitForTimeout(1000);
  } catch (e) {
    console.log("No cookie dialog or already dismissed");
  }
}'
```

**Note:** Use try/catch because the dialog only appears on first visit.

### Step 2: Navigate to Search Results

**Most efficient approach:** Construct the URL directly with all desired filters and navigate to it.

Example searches:

```bash
# Simple keyword search
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=data+scientist"

# With location filter (Copenhagen area)
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=python&amt=2"

# Full-time jobs in IT education area
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=developer&cvtype=3&udd=24"

# Multiple locations (repeat param)
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=engineer&amt=2&amt=8"

# Remote work positions
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=consultant&fjernarbejde=helt"

# Paginated results (page 2)
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=project+manager&page=2"
```

See `url-reference.md` for complete filter documentation.

### Step 3: Extract Job Listings

Use the following JavaScript code to extract all job listings from the current page:

```bash
playwright-cli -s=jobbank run-code 'async page => {
  const jobs = await page.evaluate(() => {
    return Array.from(document.querySelectorAll("div.job-item")).map(item => {
      const id = item.getAttribute("name");
      const link = item.querySelector("a[href^=\"/job/\"]");
      const url = link ? "https://jobbank.dk" + link.getAttribute("href") : null;
      const header = item.querySelector(".job-header");
      const teaser = item.querySelector(".job-teaser");
      const desc = item.querySelector(".job-description");
      const dateUpdated = item.querySelector(".job-date-updated");
      const deadline = item.querySelector(".job-date-application");
      return {
        id,
        title: header ? header.textContent.trim() : null,
        teaser: teaser ? teaser.textContent.trim() : null,
        description: desc ? desc.textContent.trim() : null,
        url,
        dateUpdated: dateUpdated ? dateUpdated.textContent.trim() : null,
        deadline: deadline ? deadline.textContent.trim() : null,
      };
    });
  });
  return JSON.stringify(jobs, null, 2);
}'
```

**Expected output:** JSON array with ~20 job objects per page, each containing:
- `id`: Job identifier
- `title`: Job title
- `teaser`: Job type, company, and location
- `description`: Brief job description excerpt
- `url`: Full URL to job detail page
- `dateUpdated`: Date job was last updated
- `deadline`: Application deadline (if available)

### Step 4: Present Results to User

Format the extracted jobs into a readable summary for the user. Include:
- Job title (with link)
- Company and location from teaser
- Brief description
- Application deadline (if present)
- Page navigation hints if more results exist

## Using Filters

Jobbank.dk supports extensive filtering through URL parameters. The most efficient approach is to construct URLs directly rather than clicking through the UI.

**Common filter patterns:**

```bash
# Job type: Full-time positions only
cvtype=3

# Education area: IT positions
udd=24

# Location: Copenhagen area
amt=2

# Work area: Software development
erf=31

# Industry: IT/Telecom
branche=10331

# Remote work: Fully remote
fjernarbejde=helt

# Suitable for: New graduates
andet=2

# Multiple filters combined
cvtype=3&udd=24&amt=2&erf=31&fjernarbejde=delvist
```

**Multiple values for same filter:**

Repeat the parameter name:
```bash
# Both Copenhagen and Aarhus
amt=2&amt=8

# Full-time OR graduate/trainee positions
cvtype=3&cvtype=6
```

See `url-reference.md` for complete filter ID tables.

## Tips

1. **Construct URLs directly** — Most reliable method. Build the full URL with all desired parameters and navigate to it in one command.

2. **Pagination** — Jobbank shows ~20 results per page. Use `&page=2`, `&page=3`, etc. to browse multiple pages. Page numbers are 1-indexed.

3. **RSS feeds** — Every search has an RSS equivalent: `https://jobbank.dk/job/rss?{same_query_params}`. Useful for monitoring new jobs.

4. **Cookie consent** — Must be dismissed once per session. Look for "Accepter valgte" button on first page load.

5. **Keyword spacing** — Use `+` or `%20` for spaces in keywords: `data+scientist` or `data%20scientist`.

6. **Exclude keywords** — Use `antikey` parameter to exclude terms: `?key=developer&antikey=senior`.

7. **Date filtering** — Use `oprettet=YYYY-MM-DD` to find jobs posted since a specific date.

8. **Company filtering** — Use `virk={company_name}` to search within specific companies.

## Known Issues and Workarounds

### Filter Dropdown Overlap

**Problem:** When using the advanced filter UI (not recommended), dropdown menus can overlap and block interactions.

**Workaround:**
- Use `{ force: true }` option in clicks: `.click({ force: true })`
- Or close previous dropdown before opening next one
- Or (better) construct URLs directly and skip the UI entirely

### Filter Checkboxes Don't Update URL

**Problem:** Clicking filter checkboxes in the UI doesn't immediately update the browser URL, making it hard to track selected filters.

**Workaround:**
- Always click the search button after selecting filters
- Or (recommended) construct URLs directly with desired parameters

### Cookie Dialog Blocks Interaction

**Problem:** On fresh sessions, the cookie consent dialog blocks all page interactions.

**Workaround:** Always dismiss the cookie dialog first by clicking "Accepter valgte" before attempting any other actions.

## Examples

### Find Data Science Jobs in Copenhagen

```bash
playwright-cli -s=jobbank open https://jobbank.dk/job/ --headed
# Dismiss cookie dialog (see Step 1)
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=data+scientist&amt=2"
playwright-cli -s=jobbank run-code 'async page => { /* extraction code */ }'
```

### Find Remote IT Jobs for New Graduates

```bash
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?udd=24&fjernarbejde=helt&andet=2"
playwright-cli -s=jobbank run-code 'async page => { /* extraction code */ }'
```

### Browse Multiple Pages of Project Manager Jobs

```bash
# Page 1
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=project+manager"
playwright-cli -s=jobbank run-code 'async page => { /* extraction code */ }'

# Page 2
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=project+manager&page=2"
playwright-cli -s=jobbank run-code 'async page => { /* extraction code */ }'
```

### Find Full-Time Software Jobs in Multiple Cities

```bash
playwright-cli -s=jobbank goto "https://jobbank.dk/job/?key=software&cvtype=3&erf=31&amt=2&amt=8"
playwright-cli -s=jobbank run-code 'async page => { /* extraction code */ }'
```
