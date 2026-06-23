---
name: oca-audit
description: Audits all installed OCA modules, comparing local commits against upstream OCA repositories. Checks only commits relevant to the specific module subdirectory and the active Odoo version branch. Also lists open PRs per module. Flags critical and warning-level keywords in commit messages and PR titles. Outputs a foldable markdown report.
invocation: user
allowed-tools: Bash, WebFetch
argument-hint: "[odoo-addons-path]"
keywords:
  critical:
    - traceback
    - security
    - vulnerability
    - CVE
    - exploit
    - injection
    - data loss
    - corruption
  warning:
    - bugfix
    - bug fix
    - regression
    - deprecat
    - breaking
    - backward compat
    - migration required
    - hotfix
    - critical fix
---

# OCA Module Audit

You are auditing OCA modules installed in this Odoo instance and writing results to a markdown report file.

## Addons path
User-provided path: $ARGUMENTS
If none given, check these in order and use the first that exists:
`./addons`, `../oca`, `./oca`, `/opt/odoo/addons`

---

## Step 1 — Detect the active Odoo version

Determine the Odoo version branch to use for all comparisons. Try in order:

```bash
# From the odoo-bin version string if available
python3 -c "import odoo; print(odoo.release.version)" 2>/dev/null | cut -d. -f1-2

# From a nearby version file
cat /opt/odoo/odoo/release.py 2>/dev/null | grep "version = " | head -1

# From the submodule branch tracking
git -C <addons-path> submodule foreach --quiet 'git branch -r | grep -oP "origin/\K[0-9]+\.[0-9]+" | sort -V | tail -1' 2>/dev/null | head -1
```

Store the result as `<odoo_version>` (e.g. `17.0`). If detection fails, ask the user.

---

## Step 2 — Discover OCA module directories

Find all git repositories under the addons path:

```bash
find <addons-path> -maxdepth 3 -name ".git" -type d
```

For each `.git` directory, verify it is OCA:
```bash
git -C <repo-dir> remote get-url origin 2>/dev/null
```

Only proceed if the URL contains `github.com/OCA/`.

Extract:
- `<oca_repo>`: repo name from URL (e.g. `sale-workflow`)
- `<repo_dir>`: the local clone directory (e.g. `/opt/odoo/addons/sale-workflow`)

Within each `<repo_dir>`, find individual Odoo module directories (they contain a `__manifest__.py`):
```bash
find <repo-dir> -maxdepth 2 -name "__manifest__.py" | sed 's|/__manifest__.py||'
```

Each of these paths is a `<module_dir>` with a `<module_name>` (the basename).

---

## Step 3 — Gather local data per module

For each `<module_dir>`:

```bash
# Last commit date and hash that touched this specific subdirectory
git -C <repo-dir> log -1 --format="%ci | %h | %s" -- <module_name>/ 2>/dev/null

# Short module path relative to repo root (for git log path filter)
# e.g. "sale_order_backorder_policy"
basename <module_dir>
```

For install date from the Odoo DB (if $DATABASE_URL is set):
```bash
psql $DATABASE_URL -t -c \
  "SELECT TO_CHAR(write_date, 'YYYY-MM-DD') FROM ir_module_module WHERE name='<module_name>' AND state='installed' LIMIT 1;" \
  2>/dev/null | xargs
```
Use `N/A` if unavailable.

---

## Step 4 — Fetch upstream and find commits relevant to this module and version

For each `<repo_dir>`, fetch the version branch (do this once per repo, not once per module):

```bash
git -C <repo-dir> fetch origin <odoo_version> --quiet 2>/dev/null
```

Then for each `<module_name>` within that repo, get all upstream commits that:
1. Are on `origin/<odoo_version>` but not in the local HEAD
2. Touch the `<module_name>/` subdirectory path specifically
3. Are not noise commits

```bash
git -C <repo-dir> log HEAD..origin/<odoo_version> \
  --format="%ci | %H | %h | %an | %s" \
  --no-merges \
  -- <module_name>/ \
  2>/dev/null \
  | grep -v -iE "(weblate|translated using|translation update)" \
  | grep -v -iE "\[bot\]"
```

Each line contains: `date | full_hash | short_hash | author | subject`

Use `full_hash` to construct the commit URL: `https://github.com/OCA/<oca_repo>/commit/<full_hash>`
Use `short_hash` as the visible link label in the markdown table.

**Additionally, exclude commits that only affect `.pot` files.** For each candidate commit, check which files it touches within the module path:

```bash
git -C <repo-dir> diff-tree --no-commit-id -r --name-only <full_hash> -- <module_name>/
```

If every file listed ends in `.pot`, skip this commit entirely — do not include it in the results or count it toward "commits behind".

This is the definitive "commits behind" list for that module. Collect all of them (no limit).

---

## Step 5 — Check open pull requests for each module

For each module, query the GitHub API to find open PRs targeting `<odoo_version>` that touch the `<module_name>/` directory.

Use WebFetch to call:
```
https://api.github.com/repos/OCA/<oca_repo>/pulls?state=open&base=<odoo_version>&per_page=100
```

From the response JSON, filter PRs where the `title` or `body` references `<module_name>`, OR fetch the PR files list to verify:
```
https://api.github.com/repos/OCA/<oca_repo>/pulls/<pr_number>/files
```
and check if any `filename` starts with `<module_name>/`.

For each matching open PR, record:
- PR number
- Title
- Author (login)
- Created date
- URL (`html_url`)
- Label names if any (e.g. `needs review`, `wip`)

Note: GitHub API allows 60 unauthenticated requests/hour. If you hit rate limits or receive 403, record "PR check skipped (rate limit)" for that module and continue.

---

## Step 6 — Keyword scanning per module

After collecting commits and PRs for a module, scan all commit subject lines and PR titles against the keyword lists defined in the frontmatter.

**Critical keywords** (case-insensitive, partial match):
traceback, security, vulnerability, CVE, exploit, injection, data loss, corruption

**Warning keywords** (case-insensitive, partial match):
bugfix, bug fix, regression, deprecat, breaking, backward compat, migration required, hotfix, critical fix

For each module, produce:
- `<critical_matches>`: list of `keyword → "commit/PR subject"` for every critical hit
- `<warning_matches>`: list of `keyword → "commit/PR subject"` for every warning hit
- `<alert_level>`: `critical` if any critical match exists, `warning` if only warning matches exist, `none` otherwise

Scan both commit subjects AND PR titles. Record which keyword triggered the match and which commit/PR it came from.

---

## Step 7 — Write the output markdown file

Write `oca-audit-report.md` in the current working directory.

### Anchor ID convention

Every module `<details>` block in the Module Details section must have a unique HTML `id` attribute on the `<details>` tag itself, using the module name as the ID:

```html
<details id="module-sale_order_backorder_policy">
```

In the Summary section, every module name cell must be a link to that anchor:

```markdown
[`sale_order_backorder_policy`](#module-sale_order_backorder_policy)
```

### Alert badge rules

| alert_level | `<details>` tag | Summary prefix | Summary table badge |
|-------------|----------------|----------------|---------------------|
| `critical`  | `<details id="module-<name>" open>` | `🚨` | `🚨 CRITICAL` |
| `warning`   | `<details id="module-<name>">` | `⚠️` | `⚠️ N behind` |
| `none` + behind | `<details id="module-<name>">` | `⚠️` | `⚠️ N behind` |
| `none` + up to date | `<details id="module-<name>">` | `✅` | `✅ 0` |

Note: critical modules still use `open` (expanded by default) in addition to having the `id`.

### JavaScript auto-opener

Embed this script block once, immediately after the report title and metadata, before the Summary section. It runs on page load, reads the URL hash, finds the matching `<details>` element by ID, opens it, opens any parent `<details>` elements above it, and scrolls it into view:

```html
<script>
(function () {
  function openAndReveal(id) {
    const el = document.getElementById(id);
    if (!el) return;
    // Open this details element
    if (el.tagName === 'DETAILS') el.open = true;
    // Walk up and open any ancestor details elements
    let parent = el.parentElement;
    while (parent) {
      if (parent.tagName === 'DETAILS') parent.open = true;
      parent = parent.parentElement;
    }
    // Scroll into view after a short delay to allow rendering
    setTimeout(() => el.scrollIntoView({ behavior: 'smooth', block: 'start' }), 100);
  }

  // Handle hash present on initial load
  if (window.location.hash) {
    const id = window.location.hash.slice(1);
    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', () => openAndReveal(id));
    } else {
      openAndReveal(id);
    }
  }

  // Handle hash changes while the page is open (clicking another summary link)
  window.addEventListener('hashchange', () => {
    const id = window.location.hash.slice(1);
    openAndReveal(id);
  });
})();
</script>
```

### File structure

```markdown
# OCA Module Audit Report

**Generated:** <datetime>
**Addons path:** <path>
**Odoo version:** <odoo_version>

> **Note on filtering:** This report excludes all translation-related commits and PRs from its analysis.
> The following are silently ignored: Weblate commits, commits whose subject matches "translated using" or "translation update", commits that only modify `.pot` files, any commit or PR authored by a `[bot]` account, and PRs from bot accounts.
> Counts and "commits behind" figures reflect functional changes only.

<script>
(function () {
  function openAndReveal(id) {
    const el = document.getElementById(id);
    if (!el) return;
    if (el.tagName === 'DETAILS') el.open = true;
    let parent = el.parentElement;
    while (parent) {
      if (parent.tagName === 'DETAILS') parent.open = true;
      parent = parent.parentElement;
    }
    setTimeout(() => el.scrollIntoView({ behavior: 'smooth', block: 'start' }), 100);
  }
  if (window.location.hash) {
    const id = window.location.hash.slice(1);
    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', () => openAndReveal(id));
    } else {
      openAndReveal(id);
    }
  }
  window.addEventListener('hashchange', () => openAndReveal(window.location.hash.slice(1)));
})();
</script>

---

## Summary

Group modules by OCA submodule. Each submodule group is a foldable `<details>` block,
collapsed by default. Inside each group is a table of its modules.

The highest alert_level among a submodule's modules determines the submodule summary badge:
- Any critical module → 🚨 in the submodule summary line
- Any warning module (no critical) → ⚠️
- All up to date → ✅

<details>
<summary>⚠️ sale-workflow — 2 modules · 1 critical · 1 behind</summary>

| Module | Local last commit | Installed | Commits behind | Open PRs | Alert |
|--------|-------------------|-----------|---------------|----------|-------|
| [`sale_order_backorder_policy`](#module-sale_order_backorder_policy) | 2024-11-03 | 2024-12-01 | ⚠️ 2 | 🔀 1 | 🚨 CRITICAL |
| [`sale_stock_picking_blocking`](#module-sale_stock_picking_blocking) | 2024-10-01 | 2024-10-15 | ✅ 0 | — | — |

</details>

<details>
<summary>⚠️ stock-logistics-workflow — 1 module · 1 behind</summary>

| Module | Local last commit | Installed | Commits behind | Open PRs | Alert |
|--------|-------------------|-----------|---------------|----------|-------|
| [`stock_move_full_reservation`](#module-stock_move_full_reservation) | 2024-10-15 | N/A | ⚠️ 3 | — | ⚠️ warning |

</details>

<details>
<summary>✅ account-invoicing — 1 module · all up to date</summary>

| Module | Local last commit | Installed | Commits behind | Open PRs | Alert |
|--------|-------------------|-----------|---------------|----------|-------|
| [`account_invoice_helper`](#module-account_invoice_helper) | 2024-09-01 | 2024-09-15 | ✅ 0 | — | — |

</details>

---

## Module Details

### 🚨 Critical alerts

<details id="module-sale_order_backorder_policy" open>
<summary>🚨 `sale_order_backorder_policy` — OCA/sale-workflow — 2 commits behind · 1 open PR</summary>

**OCA repo:** https://github.com/OCA/sale-workflow/tree/<odoo_version>/sale_order_backorder_policy
**Local last commit:** 2024-11-03 14:22 `a3f9c12` — Fix backorder confirmation dialog
**Installed:** 2024-12-01

> 🚨 **Critical:** `traceback` matched in commit [`b7d3e21`](https://github.com/OCA/sale-workflow/commit/b7d3e21full) — "Fix traceback on empty picking"
> ⚠️ **Warning:** `bugfix` matched in PR [#1234](https://github.com/OCA/sale-workflow/pull/1234) — "Bugfix: backorder reason field not saved"

#### Upstream commits not yet local

| Date | Hash | Author | Message |
|------|------|--------|---------|
| 2025-01-15 09:41 | [`b7d3e21`](https://github.com/OCA/sale-workflow/commit/b7d3e21full) | John Dev | Fix traceback on empty picking 🚨 |
| 2025-01-10 16:03 | [`c4a1f09`](https://github.com/OCA/sale-workflow/commit/c4a1f09full) | Jane Dev | Add partner-level override |

#### Open Pull Requests targeting <odoo_version>

| PR | Title | Author | Created | Labels |
|----|-------|--------|---------|--------|
| [#1234](https://github.com/OCA/sale-workflow/pull/1234) | Bugfix: backorder reason field not saved ⚠️ | contributor1 | 2025-01-20 | needs review |

</details>

### ⚠️ Warnings / commits behind

<details id="module-stock_move_full_reservation">
<summary>⚠️ `stock_move_full_reservation` — OCA/stock-logistics-workflow — 3 commits behind</summary>

**OCA repo:** https://github.com/OCA/stock-logistics-workflow/tree/<odoo_version>/stock_move_full_reservation
**Local last commit:** 2024-10-15 11:05 `d9b2a44` — Initial release
**Installed:** N/A

> ⚠️ **Warning:** `regression` matched in commit [`e5c3b12`](https://github.com/OCA/stock-logistics-workflow/commit/e5c3b12full) — "Fix regression in full reservation logic"

#### Upstream commits not yet local

| Date | Hash | Author | Message |
|------|------|--------|---------|
| 2025-01-12 10:00 | [`e5c3b12`](https://github.com/OCA/stock-logistics-workflow/commit/e5c3b12full) | Dev Name | Fix regression in full reservation logic ⚠️ |

</details>

### ✅ Up to date

<details id="module-account_invoice_helper">
<summary>✅ `account_invoice_helper` — OCA/account-invoicing — up to date</summary>

**OCA repo:** https://github.com/OCA/account-invoicing/tree/<odoo_version>/account_invoice_helper
**Local last commit:** 2024-09-01 08:30 `f1a2b3c` — Initial release
**Installed:** 2024-09-15

</details>

---

## Skipped

| Path | Reason |
|------|--------|
| `/opt/odoo/addons/some-dir` | Not an OCA remote |
```

### Inline alert badges in commit/PR tables

When a commit message or PR title matched a keyword, append the appropriate badge at the end of the Message/Title cell:
- `🚨` for critical keyword match
- `⚠️` for warning keyword match

This makes matches scannable inside expanded detail sections without having to re-read the alert block.

---

## Step 8 — Print terminal summary only

After writing the file, output only:
```
OCA Audit complete → oca-audit-report.md
  Odoo version : 17.0
  Modules      : 23 scanned, 7 behind, 16 up to date, 2 skipped
  Open PRs     : 4 found across 3 modules
  Alerts       : 2 critical 🚨, 3 warning ⚠️
  Rate limits  : 0 modules skipped
```

Do NOT print the full report content to the terminal.

---

## Filtering rules (apply in Steps 4 and 5)

Exclude any commit where subject line OR author name matches (case-insensitive):
- `weblate`
- `translated using`
- `translation update`
- `[bot]` (in subject or author)
- Author login ending in `-bot` or `[bot]`

Additionally exclude any commit where **all files touched within the module path end in `.pot`**.
Check this with `git diff-tree` per commit after the initial log filter.

These must not appear in commit tables or count toward "commits behind".
