---
name: jobdanmark-search
description: Search for jobs on jobdanmark.dk. Use when the user wants to find jobs on Jobdanmark, search Danish job listings. Supports filtering by keyword, location, job type, and category.
allowed-tools: Bash(playwright-cli:*)
argument-hint: [search query]
---

# Jobdanmark.dk Job Search

This skill searches for jobs on jobdanmark.dk using playwright-cli automation.

## Prerequisites

**You MUST have an open browser session before running any commands.**

```bash
# Headless (default, supports parallel execution)
playwright-cli -s=jobdanmark open "https://jobdanmark.dk"

# Headed (visible browser, for debugging)
playwright-cli -s=jobdanmark open "https://jobdanmark.dk" --headed
```

**Named session `-s=jobdanmark`** allows this skill to run in parallel with other job search skills. Use `-s=jobdanmark` on all subsequent commands.

Wait a few seconds for the page to load before proceeding.

## Workflow

### Step 1: Dismiss Cookie Dialog

On fresh sessions, dismiss the Didomi cookie consent dialog:

```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  try {
    await page.waitForTimeout(1500);
    await page.getByRole("button", { name: /Afvis og luk/ }).click({ timeout: 3000 });
    await page.waitForTimeout(1000);
  } catch (e) {
    console.log("No cookie dialog or already dismissed");
  }
}'
```

**Note:** Use try/catch because the dialog only appears on first visit. The button name matches the pattern "Afvis og luk" (Decline and close).

### Step 2: Navigate to Search Results

Build the search URL using query parameters and navigate:

**Basic search (keyword only):**
```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  const searchUrl = "https://jobdanmark.dk/jobsoeger/find-job?freetext=data%20analyst";
  await page.goto(searchUrl, { waitUntil: "domcontentloaded" });
  await page.waitForTimeout(2000);
}'
```

**Search with filters (location, type, category):**
```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  const params = new URLSearchParams({
    freetext: "softwareudvikler",
    location: "København",
    jobtype: "fuldtid",
    category: "227978"
  });
  const searchUrl = "https://jobdanmark.dk/jobsoeger/find-job?" + params.toString();
  await page.goto(searchUrl, { waitUntil: "domcontentloaded" });
  await page.waitForTimeout(2000);
}'
```

**CRITICAL:** Always construct URLs inside JavaScript using `URLSearchParams` or unicode escapes. Shell interprets `&` as background operator. DO NOT use raw `&` characters in bash strings.

**Alternative (unicode escape):**
```bash
# Replace & with \u0026 in the URL string
playwright-cli -s=jobdanmark open "https://jobdanmark.dk/jobsoeger/find-job?freetext=udvikler\u0026location=Odense"
```

### Step 3: Extract Job Listings

Extract job data from the Vue.js app layer using page.evaluate():

```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  await page.waitForTimeout(2000);
  const jobs = await page.evaluate(() => {
    const jobLinks = document.querySelectorAll("a.job-list");
    return Array.from(jobLinks).map(el => {
      const title = el.querySelector("h6, h5")?.textContent?.trim() || "";
      const company = el.querySelector("strong")?.textContent?.trim() || "";
      const location = el.querySelectorAll("li");
      const locText = location.length > 1 ? location[1]?.textContent?.trim() : "";
      const deadlineMatch = el.textContent?.trim().match(/Ansøgningsfrist:\s*([\d-]+)/);
      const deadline = deadlineMatch ? deadlineMatch[1] : "";
      return {
        url: "https://jobdanmark.dk" + el.getAttribute("href"),
        title,
        company,
        location: locText,
        deadline
      };
    });
  });
  return JSON.stringify(jobs, null, 2);
}'
```

**Output format:**
```json
[
  {
    "url": "https://jobdanmark.dk/job/some-slug",
    "title": "Softwareudvikler",
    "company": "IT Company ApS",
    "location": "København K",
    "deadline": "2026-03-15"
  }
]
```

**Typical page load:** 30 results per load. Use "Indlæs flere job" button for more results.

### Step 4: Present Results

Format the extracted JSON into a readable summary for the user:
- Total number of jobs found
- List each job with title, company, location, deadline, and URL
- Note if there are more results available (button "Indlæs flere job" present)

**Load more results (if needed):**
```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  try {
    await page.getByRole("button", { name: /Indlæs flere job/ }).click({ timeout: 3000 });
    await page.waitForTimeout(2000);
    console.log("Loaded more jobs");
  } catch (e) {
    console.log("No more jobs to load");
  }
}'
```

Then repeat Step 3 to extract the updated job list.

## URL Patterns

**Base search URL:**
```
https://jobdanmark.dk/jobsoeger/find-job
```

**Query parameters:**

| Parameter | Type | Values | Example |
|-----------|------|--------|---------|
| `freetext` | string | Job title, keyword | `data%20analyst` |
| `location` | string | City name or postal code | `Odense` or `5000` |
| `jobtype` | string | fuldtid, deltid, elev, studiejob, praktik, fleksjob | `fuldtid` |
| `category` | number | Category ID (see table below) | `227978` |

**Job types:**
- `fuldtid` — Full-time
- `deltid` — Part-time
- `elev` — Apprentice
- `studiejob` — Student job
- `praktik` — Internship
- `fleksjob` — Flex job

**Sorting:**
- Bedste match — Best match
- Oprettelsesdato — Creation date (default)
- Ansøgningsfrist — Application deadline

**Job detail URL pattern:**
```
https://jobdanmark.dk/job/{slug}
```

**Pagination:** No URL-based pagination. Use "Indlæs flere job" button to load 30 more results at a time.

## Categories

| Category ID | Name (Danish) |
|-------------|---------------|
| 227972 | Pædagogik, Uddannelse og Forskning |
| 227973 | Håndværk, Industri, Transport og Landbrug |
| 227974 | Salg, Kommunikation, Marketing, og Design |
| 227975 | Pleje, Social og Sundhed |
| 227976 | Hotel, Service, Restauration og Sikkerhed |
| 227977 | Kontor, Finans og Økonomi |
| 227978 | IT, Ingeniør og Energi |
| 227979 | Ledelse, HR og projektstyring |
| 543415 | Kirke, Kultur og Underholdning |
| 227980 | Øvrige job |

**English reference:**
- 227972: Education, Research
- 227973: Crafts, Industry, Transport, Agriculture
- 227974: Sales, Communication, Marketing, Design
- 227975: Care, Social, Health
- 227976: Hotel, Service, Restaurant, Security
- 227977: Office, Finance, Economy
- 227978: IT, Engineering, Energy
- 227979: Management, HR, Project Management
- 543415: Church, Culture, Entertainment
- 227980: Other jobs

## Tips

### URL Construction
**ALWAYS construct search URLs inside JavaScript** using `URLSearchParams` or use unicode escape `\u0026` for ampersands in bash strings. The shell interprets `&` as a background operator.

**Good:**
```javascript
const params = new URLSearchParams({ freetext: "udvikler", location: "Aarhus" });
const url = "https://jobdanmark.dk/jobsoeger/find-job?" + params.toString();
```

**Bad:**
```bash
# Shell treats & as background operator - NEVER do this
playwright-cli -s=jobdanmark open "https://jobdanmark.dk/jobsoeger/find-job?freetext=udvikler&location=Aarhus"
```

### Vue.js App Layer
Jobdanmark.dk uses a Vue.js app on top of Jobnet SSR. The snapshot shows SSR content but JavaScript extraction finds Vue app content. **Always use `page.evaluate()` for extraction** — snapshots are misleading.

### Job Card Selector
Use `a.job-list` to select job cards. Each card contains:
- `h6` or `h5` — Job title
- `strong` — Company name
- Second `li` element — Location
- Deadline text pattern: `Ansøgningsfrist: YYYY-MM-DD`

### Wait Times
Allow 2 seconds after navigation for Vue.js hydration:
```javascript
await page.waitForTimeout(2000);
```

### Cookie Dialog
The Didomi consent dialog only appears once per browser session. Always wrap dismissal in try/catch to avoid errors on subsequent searches.

### Search Button Disambiguation
On the homepage, two buttons match "Søg". Use exact match when needed:
```javascript
await page.getByRole("button", { name: "Søg", exact: true }).click();
```

### Location Input
Location accepts both city names and postal codes:
- City: `København`, `Aarhus`, `Odense`
- Postal code: `5000`, `8000`, `2100`

### Empty Results
If no jobs match, the page shows "Ingen job matcher din søgning" (No jobs match your search). Check for this message to avoid showing empty results as errors.

## Known Issues

1. **Shell ampersand interpretation:** Raw `&` in URLs causes the shell to background the process. Always use `URLSearchParams` in JavaScript or unicode escape `\u0026`.

2. **SSR vs Vue layer mismatch:** Snapshots show Jobnet SSR content but the Vue app renders different content. Never trust snapshot selectors — always verify with `page.evaluate()`.

3. **No URL pagination:** Unlike jobindex.dk, jobdanmark.dk doesn't support `?page=2` URLs. Must use "Indlæs flere job" button for pagination.

4. **Cookie dialog blocking:** The Didomi dialog blocks all interactions until dismissed. Always handle it first with try/catch.

5. **Deadline extraction fragility:** Deadline format is text-based (`Ansøgningsfrist: 2026-03-15`), not structured data. The regex may need adjustment if the format changes.

6. **Load more timing:** After clicking "Indlæs flere job", wait at least 2 seconds before extracting jobs. Vue needs time to append new results.

## Example Searches

**Search for full-time IT jobs in Copenhagen:**
```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  const params = new URLSearchParams({
    freetext: "softwareudvikler",
    location: "København",
    jobtype: "fuldtid",
    category: "227978"
  });
  await page.goto("https://jobdanmark.dk/jobsoeger/find-job?" + params.toString());
  await page.waitForTimeout(2000);
}'
```

**Search for part-time jobs in Odense (postal code):**
```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  const params = new URLSearchParams({
    freetext: "butiksassistent",
    location: "5000",
    jobtype: "deltid"
  });
  await page.goto("https://jobdanmark.dk/jobsoeger/find-job?" + params.toString());
  await page.waitForTimeout(2000);
}'
```

**Broad keyword search (no filters):**
```bash
playwright-cli -s=jobdanmark run-code 'async page => {
  await page.goto("https://jobdanmark.dk/jobsoeger/find-job?freetext=data%20analyst");
  await page.waitForTimeout(2000);
}'
```
