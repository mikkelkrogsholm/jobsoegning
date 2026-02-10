---
name: jobindex-search
description: Search for jobs on jobindex.dk. Use when the user wants to find jobs, search for positions, look for work, or browse job listings on Jobindex. Supports filtering by keyword, location, category, recency, and pagination.
allowed-tools: Bash(playwright-cli:*)
argument-hint: [search query]
---

# Jobindex.dk Job Search

Search for jobs on jobindex.dk using playwright-cli for browser automation.

## Prerequisites

A browser session must be open. If not, open one:

```bash
playwright-cli open https://www.jobindex.dk --headed
```

For headless operation (no visible browser):

```bash
playwright-cli open https://www.jobindex.dk
```

## Workflow

### Step 1: Dismiss popups

On first visit, up to three popups appear. Dismiss them in order:

```bash
# Cookie consent — click "Afvis" (reject)
playwright-cli snapshot
# Find and click the "Afvis" button by ref
playwright-cli click <ref-for-Afvis>

# AI assistant popup — click "Luk" (close)
playwright-cli click <ref-for-Luk>

# Jobagent popup (appears after first search) — click "Luk" inside dialog
playwright-cli click <ref-for-dialog-Luk>
```

**Important:** Refs change per session. Always take a snapshot first and find buttons by name:
- Cookie: button named "Afvis"
- AI popup: button named "Luk"
- Jobagent: button named "Luk" inside a `dialog` element

Some popups may not appear. That's fine — just continue.

### Step 2: Navigate to search results via URL

The most efficient approach is constructing the URL directly:

```bash
playwright-cli goto "https://www.jobindex.dk/jobsoegning?q=data+engineer"
```

For detailed URL patterns, see [url-reference.md](url-reference.md).

### Step 3: Extract job listings

```bash
playwright-cli run-code 'async page => {
  const jobs = await page.$$eval("div.PaidJob, div.jix_robotjob", cards => cards.map(card => {
    const title = card.querySelector("h4 a")?.textContent?.trim() || "";
    const url = card.querySelector("h4 a")?.href || "";
    const company = card.querySelector("a[href]")?.textContent?.trim() || "";
    const locationEl = card.querySelector("h4")?.parentElement?.querySelector("h4 ~ div");
    const location = locationEl?.textContent?.replace(/Se rejsetid/g, "").trim() || "";
    const date = card.querySelector("time")?.textContent?.trim() || "";
    const desc = Array.from(card.querySelectorAll("p")).map(p => p.textContent.trim()).join(" ");
    return { title, company, location, date, url, desc: desc.substring(0, 300) };
  }));
  const countEl = await page.$("h2");
  const count = countEl ? await countEl.textContent() : "";
  return JSON.stringify({ total: count.trim(), jobs }, null, 2);
}'
```

### Step 4: Present results to the user

Format the extracted jobs as a readable list with:
- Job title (linked to URL)
- Company name
- Location
- Posted date
- Short description

## Using the Geografi (km radius) Filter

When the user wants jobs near a specific address/city, use the Geografi filter UI. This cannot be done via URL alone.

### Procedure

```bash
# 1. Navigate to search results first
playwright-cli goto "https://www.jobindex.dk/jobsoegning?q=administrativ"

# 2. Dismiss any popups (see Step 1 above)

# 3. Take snapshot, find and click Geografi button
playwright-cli snapshot
playwright-cli click <ref-for-Geografi-button>

# 4. Fill km radius (spinbutton "Hvor langt vil du have til arbejde?")
playwright-cli fill <ref-for-km-spinbutton> "30"

# 5. Fill address (textbox "Fra hvilken adresse?")
playwright-cli fill <ref-for-address-textbox> "Skævinge"

# 6. Take snapshot to see autocomplete suggestions
playwright-cli snapshot

# 7. Click the desired address suggestion (role: option)
playwright-cli click <ref-for-address-option>

# 8. CRITICAL: Dismiss the autocomplete dropdown before clicking submit
#    Press Tab to move focus away (do NOT press Escape — it closes the whole panel)
playwright-cli press Tab

# 9. Take snapshot, find and click "Vis X job" button
playwright-cli snapshot
playwright-cli click <ref-for-vis-job-button>
```

### Known pitfalls

- **NEVER press Escape** in filter panels — it closes the entire panel and loses unsaved values
- **Autocomplete dropdown blocks clicks** — after selecting an address option, press `Tab` to dismiss the dropdown before clicking "Vis X job"
- **Both km AND address required** — setting only one won't filter
- **Filter values are lost if panel closes** — always click "Vis X job" before doing anything else
- **Refs change between snapshots** — always take a fresh snapshot before clicking

If "Vis X job" is blocked by overlay, use force click as last resort:
```bash
playwright-cli run-code 'async page => { await page.getByRole("button", { name: /Vis.*job/ }).click({ force: true }); }'
```

## Using the Kategorier Filter

Works similarly to Geografi but with category autocomplete:

```bash
# 1. Click Kategorier button
# 2. Type category in search field
# 3. Select from autocomplete (treeitem)
# 4. Press Tab to dismiss
# 5. Click "Vis X job"
```

## Tips

- **Broad vs exact search:** `q=data+engineer` (broad, many results) vs `q=%27data+engineer%27` (exact match with quotes, fewer results)
- **Pagination:** Add `&page=2`, `&page=3` etc. ~20 results per page.
- **Multiple pages:** To get more results, loop through pages by incrementing the page parameter.
- **Popups:** On a fresh session you MUST dismiss popups before extracting data. After the first dismissal in a session, they won't reappear.
- **Browser reuse:** If a browser is already open from a previous command, just use `playwright-cli goto` — no need to reopen.
- **Always snapshot before interacting** — refs are session-specific and change with every snapshot.
- **Prefer URL construction** over UI interaction when possible — it's faster and avoids filter panel pitfalls.
