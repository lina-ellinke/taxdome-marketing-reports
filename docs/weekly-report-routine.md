# Weekly Marketing Report Routine

> **Methodology reconciliation (May 17, 2026):** Field names and filters in this spec
> have been patched to match the W17-W20 weekly reports. Key changes from the original
> draft: icp__nonicp replaced with marketing_icp__nonicp_enriched, MQL/SQL date fields
> updated to hs_v2 properties, SQL definition requires demo registration, all queries
> filtered to list 6894, closed-won scoped to Sales pipelines only. Any future
> methodology changes should be documented here with the date and reason.

**Schedule:** Every Tuesday at 1:00 PM America/Panama (UTC-5, no DST)

---

## OVERVIEW

You are generating TaxDome's weekly marketing report. This runs automatically every Tuesday. Your job is to:

1. Read the monthly and quarterly targets from CSV files in this repo
2. Pull live data from HubSpot for three time windows
3. Compute WoW and YoY deltas for every metric
4. Update the live dashboard and create a frozen weekly snapshot
5. Create a Confluence page with the dashboard embedded
6. Notify Lina on Slack that the draft is ready

Do NOT ask questions. Execute all steps autonomously. If a HubSpot query fails, note the failure in the dashboard card as "data unavailable" and continue with the remaining metrics.

---

## STEP 1: DETERMINE DATES AND WEEK NUMBER

Calculate the following using today's date (Tuesday):

- **Current week**: Monday through Sunday of the week just ended (the 7 days ending yesterday, Monday)
- **Prior week**: the 7 days before current week (for WoW comparison)
- **Same week last year**: the equivalent 7-day window from the prior year (for YoY comparison)
- **Month-to-date**: first day of current month through yesterday
- **Q2 dates**: April 1 through June 30, 2026 (91 days)
- **Q2 day count**: number of days elapsed in Q2 as of yesterday
- **Current month day count**: number of days elapsed in the current month as of yesterday
- **Days in current month**: total days in the current month
- **ISO week number**: for the snapshot folder name (format: YYYY-WNN)

---

## STEP 2: READ TARGETS

Read `targets/monthly-plan.csv` and `targets/quarterly-plan.csv` from this repo.

From the monthly plan, extract the current month's column. The metrics you need:

| Metric | Row in CSV |
|--------|-----------|
| Revenue target | Revenue target ($) |
| MQLs total | MQLs needed total |
| MQLs Non-ICP | MQLs (Non-ICP) |
| MQLs ICP | MQLs (ICP) |
| SQLs total | SQLs needed (exact) |
| SQLs Non-ICP | SQLs (Non-ICP) |
| SQLs ICP | SQLs (ICP) |
| Classified Leads total | Leads needed total |
| Leads Non-ICP | Leads (Non-ICP) |
| Leads ICP | Leads (ICP) |

From the quarterly plan, extract the current quarter's column for Q2 OKR cumulative cards:

| Metric | Row in CSV |
|--------|-----------|
| Q2 Revenue target | Revenue target ($) |
| Q2 MQLs | MQLs needed total |
| Q2 SQLs | SQLs needed (exact) |
| Q2 Leads | Leads needed total |

**ACV override**: The blended ACV in the CSV is $2,620. The operational target is $3,000. Use $3,000 for the ACV KPI card and OKR card. Do not use $2,620.

**TD Payments target**: 30% activation rate. This is not in the CSV.

---

## STEP 3: PULL HUBSPOT DATA

Query HubSpot for each of the three time windows (current week, prior week, same week last year). Use inclusive date boundary logic: `date_field >= start AND date_field < day after end`.

All field names and filters below match the W17-W20 weekly report methodology exactly. Do not substitute.

### 3A: Classified Leads and ICP Share

Search the `companies` object with these filters:

- `ilsListIds` contains `6894` (marketing-source companies only — all-market list)
- `createdate` within the time window
- `marketing_icp__nonicp_enriched` = `"ICP"` OR `"Non-ICP"` (case-sensitive, hyphen in Non-ICP)

Output:
- Total classified leads (ICP + Non-ICP count)
- ICP count
- Non-ICP count
- ICP share % (ICP / total classified)

Do NOT use the older `icp__nonicp` property — it was populated by bulk imports and spikes unpredictably. RevOps moved off it in April 2026.

### 3B: MQLs

Search the `companies` object with these filters:

- `ilsListIds` contains `6894`
- `hs_v2_date_entered_marketingqualifiedlead` within the time window
- `lifecyclestage` HAS_PROPERTY (sanity check that the company has progressed)
- Split by `marketing_icp__nonicp_enriched` (ICP / Non-ICP / unset for No Value)

Output:
- Total MQLs
- MQLs where `marketing_icp__nonicp_enriched` = `"ICP"`
- MQLs where `marketing_icp__nonicp_enriched` = `"Non-ICP"`
- No Value count (where the property is unset) — informational

Do NOT use `hs_lifecyclestage_marketingqualifiedlead_date` — it was retired when RevOps standardized on v2 lifecycle properties.

### 3C: SQLs (Companies Demo Booked)

Search the `companies` object with these filters:

- `ilsListIds` contains `6894`
- `hs_v2_date_entered_opportunity` within the time window
- `n1_1_demo_registered_date` HAS_PROPERTY (company must have a demo registered)
- Split by `marketing_icp__nonicp_enriched`

Output:
- Total SQLs (= Companies Demo Booked)
- SQLs where `marketing_icp__nonicp_enriched` = `"ICP"`
- SQLs where `marketing_icp__nonicp_enriched` = `"Non-ICP"`

The demo-registered filter is essential. Without it, counts will be 4-6x too high because every Opportunity-stage company is included, not just demo-booked ones. RevOps "SQL" definition = Companies Demo Booked.

**Known limitation**: Pipeline filter (Sales Inbound 135241873 + Sales International 134917174) on associated deals is not directly queryable via HubSpot search API. Numbers are approximated; flag with footnote on the SQL chart that this is "approximate — pipeline filter pending."

### 3D: MQL-to-SQL Conversion Rate

Calculate from the MQL and SQL data above (period-aggregate ratios, not cohort):

- Overall conversion: SQLs / MQLs for the current week
- ICP conversion: ICP SQLs / ICP MQLs
- Non-ICP conversion: Non-ICP SQLs / Non-ICP MQLs

### 3E: ACV (Average Contract Value)

Search the `deals` object with these filters (two filterGroups combined with OR for the two pipelines):

- `hs_is_closed_won` = `"true"`
- `closedate` within the time window (use Apr 1 through end of current period for full Q2 view; or use month-bounded windows per the report context)
- `amount` >= `10` (exclude test/junk deals)
- `pipeline` = `135241873` (Sales Inbound) — first filterGroup
- `pipeline` = `134917174` (Sales International) — second filterGroup

Output:
- Total closed-won deal count
- Total deal amount
- Average deal amount (ACV)

Do NOT use "Deal Stage 7" alone or `dealstage` = `"closedwon"` without the pipeline filter — that includes CS Retention/Expansion and other CS pipelines that inflate ACV and don't represent new logos.

### 3F: TD Payments Activation

Point-in-time metric, not time-windowed. Pull once for current state.

Search the `companies` object:

- Numerator: `td_payments_enabled` = `"true"` AND `csam` HAS_PROPERTY AND `license_type` = `"paid"`
- Denominator: `csam` HAS_PROPERTY AND `license_type` = `"paid"`
- Activation rate: numerator / denominator
- Target: 30%

US-only scope is enforced by the CSAM filter (denominator is paid accounts with CSAM, not all paid accounts).

### 3G: Marketing-Sourced Pipeline ARR

Search the `deals` object for open (not closed) deals in the current pipeline, marketing-attributed, created within each time window. Sum the `amount` field.

Output:
- Total marketing-sourced pipeline ARR for current week
- Same for prior week (WoW)
- Same for same week last year (YoY)

---

## STEP 4: COMPUTE DELTAS

For every metric that has three time windows, compute:

**WoW (Week over Week):**
- Volume metrics (leads, MQLs, SQLs, ACV dollar, pipeline): show as percentage change. Format: "+X% WoW" or "-X% WoW" with green/red coloring.
- Share metrics (ICP share %, conversion rates, activation rate): show as percentage point change. Format: "+X pp WoW" or "-X pp WoW".

**YoY (Year over Year):**
- Same formatting rules as WoW but labeled "YoY".
- For metrics where the methodology changed mid-year (e.g. anything filtered by list 6894 starting April 2026), flag YoY as "methodology change" instead of computing a number, to avoid misleading comparisons.

**Month-to-date pace:**
- For monthly KPI cards: (MTD actual / monthly target) vs (days elapsed / days in month)
- If actual pace >= 100% of target pace: green (on track)
- If actual pace is 80-99% of target pace: orange (at risk)
- If actual pace < 80% of target pace: red (not on pace)

**Q2 cumulative pace:**
- For Q2 OKR cards: (Q2 cumulative actual / Q2 target) vs (Q2 days elapsed / 91)
- Same green/orange/red logic

---

## STEP 5: UPDATE DASHBOARD

### 5A: Update live dashboard

Open `dashboard/index.html`. Update all data values, delta indicators, pace colors, and date references:

- Subtitle: "Last updated: [today's date] | Week of [current week start] - [current week end]"
- All KPI cards with fresh values, WoW deltas, YoY deltas, and pace indicators
- All charts with current week data points added (drop oldest week, append current)
- Q2 OKR progress bars with cumulative actuals vs quarterly targets
- Update Q2 pace line (day N of 91)

Use TaxDome brand colors:
- Primary blue: `#4C6EF5`
- Green (on track): `#2F9E44`
- Orange (at risk): `#E8590C`
- Red (off pace): `#E03131`

### 5B: Create frozen weekly snapshot

Copy the updated dashboard to `dashboard/weekly/[YYYY-WNN]/index.html` where `[YYYY-WNN]` is the ISO week number (e.g. `2026-W21`).

Title the snapshot: "TaxDome Marketing — Weekly Overview — Week [NN] ([week start] - [week end], [year])"

Update the snapshot's relative paths for the campaign-reports footer: links should use `../../../campaigns/` (three levels up from `dashboard/weekly/YYYY-WNN/`).

Also update `dashboard/weekly/index.html` to add the new week's snapshot to the list.

### 5C: Commit to GitHub

**Use `mcp__github__push_files` for all commits.** Do NOT use local `git push` — it 403s through the proxy in this environment.

Push all changed files in a single commit:
- `dashboard/index.html` (live dashboard)
- `dashboard/weekly/YYYY-WNN/index.html` (frozen snapshot)
- `dashboard/weekly/index.html` (updated index)
- Any campaign-report updates if applicable

Commit message: `Weekly report: [YYYY-WNN] | MQLs: [value] ([WoW delta]) | SQLs: [value] ([WoW delta]) | ACV: $[value]`

---

## STEP 6: CREATE CONFLUENCE PAGE

Create a new child page under the Marketing Weekly Summary parent page.

**Confluence details:**
- Cloud ID: `39336249-9666-44dc-aa6f-abfd48a357ce`
- Space: Operations (key: `1298661388`)
- Parent page ID: `2697592850`
- Page title: `[YYYY-MM-DD] Marketing Weekly Report`

**Page content structure:**

```
## Executive Summary
[LEAVE BLANK, Lina fills this in]

## Dashboard
[Embed iframe: https://lina-horner.github.io/taxdome-marketing-reports/dashboard/weekly/YYYY-WNN/index.html]

## Key Changes This Week
- MQLs: [value] ([WoW] WoW, [YoY] YoY) vs [monthly target] monthly target
- SQLs: [value] ([WoW] WoW, [YoY] YoY) vs [monthly target] monthly target
- ICP Lead Share: [value]% ([WoW delta] pp WoW)
- ACV: $[value] ([WoW] WoW)
- TD Payments Activation: [value]% vs 30% target
- Pipeline ARR: $[value] ([WoW] WoW)

## Attention Required
[List any metrics that are red or orange on pace, with context]

## Focus Next Week
[LEAVE BLANK, Lina fills this in]
```

---

## STEP 7: NOTIFY ON SLACK

Send a Slack DM to Lina with:

```
Weekly report draft ready for review: [Confluence page URL]

Quick numbers: MQLs [value] ([WoW delta] WoW), SQLs [value] ([WoW delta] WoW), ACV $[value], ICP share [value]%

[Count] metrics on pace, [count] at risk, [count] off pace.
```

---

## MCP DEPENDENCIES

The routine requires these MCP connections at runtime:

- **HubSpot MCP** — for all data pulls (steps 3A-3G)
- **Confluence MCP** — cloud ID `39336249-9666-44dc-aa6f-abfd48a357ce`, space `1298661388`, parent page `2697592850` (step 6)
- **Slack MCP** — for notification to Lina (step 7)
- **GitHub MCP** — for commits via `mcp__github__push_files` (step 5C)

If Confluence or Slack MCP is unavailable at runtime, continue with all other steps and note the failure in the commit message. The dashboard and snapshot are the source of truth; Confluence and Slack are notification surfaces.

---

## ERROR HANDLING

- **HubSpot query fails**: mark the KPI card as "Data unavailable" in gray, continue with all other metrics, note the failure in the Slack message.
- **GitHub commit fails**: retry once with `mcp__github__push_files`. If it still fails, notify on Slack that the dashboard update failed and include the error.
- **Confluence page creation fails**: include the dashboard URL directly in the Slack message instead.
- **Slack notification fails**: the dashboard and Confluence page are still updated. Lina will see them on review.

Do NOT fall back to local `git push` — it 403s through the proxy.

---

## IMPORTANT NOTES

- **SALs are retired.** Do NOT query for or display SAL data. The funnel is Leads > MQLs > SQLs > Customers.
- **Classified leads = ICP + Non-ICP only.** Exclude unclassified (no value in `marketing_icp__nonicp_enriched`) from the "classified" count, but show the No Value segment separately in stacked charts for transparency.
- **ACV target is $3,000**, not the $2,620 from the CSV. This was an explicit operational override; do not "fix" it back to the CSV value.
- **TD Payments is US-only.** Denominator is paid accounts with CSAM, not all paid accounts.
- **All timestamps in America/Panama** timezone (UTC-5, no DST).
- **Monthly KPI cards** pace against monthly targets from `targets/monthly-plan.csv`.
- **Q2 OKR cumulative cards** pace against quarterly targets from `targets/quarterly-plan.csv`.
- **Quarterly target totals are intentionally less than the sum of monthly targets.** Use each as-is; do not recalculate one from the other.
- **Field-name guardrail:** the W17-W20 reports use the property names listed in step 3. Any future change to a different field name must be documented in the reconciliation block at the top of this file with date and reason.

---

## REFERENCE: METHODOLOGY HISTORY

| Date | Change | Reason |
|---|---|---|
| Apr 2026 | Switched from `icp__nonicp` to `marketing_icp__nonicp_enriched` | Old field spiked on bulk imports; RevOps standardized on enriched field |
| Apr 2026 | Switched lead methodology from list 9736 (US/CA) to list 6894 (all-market, marketing-source) | Aligned with Mikhail's "Total Leads MoM" RevOps report |
| Apr 2026 | Switched from `hs_v2_date_entered_lead` to `createdate` for lead date filter | Matches RevOps definition |
| Apr 2026 | SQL definition changed to require `n1_1_demo_registered_date HAS_PROPERTY` | RevOps definition = "Companies Demo Booked", not raw Opportunity stage entry |
| Apr 2026 | ACV scoped to Sales Inbound + Sales International pipelines, amount ≥ $10 | Excludes CS pipelines (Retention, Expansion) and test deals that distort new-logo ACV |
| Apr 2026 | MQL date field updated to `hs_v2_date_entered_marketingqualifiedlead` | v2 lifecycle property standardization |
| May 17, 2026 | Routine spec reconciled to match W17-W20 published reports | Original routine draft used outdated field names that would produce divergent numbers |
