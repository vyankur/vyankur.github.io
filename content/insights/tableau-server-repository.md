# Querying the Tableau Server Repository: A Practical Guide to PostgreSQL Metadata

How to pull governance-grade answers - permissions, view activity, RLS coverage - straight from Tableau's own repository database, without relying on the Metadata API.

---

## Overview

Every Tableau Server (and Tableau Cloud, behind the scenes) is backed by a PostgreSQL repository database that logs almost everything: who owns which workbook, who can see which view, when a dashboard was last opened, and which entitlements a user actually inherited once group and project permissions are flattened out. The Metadata API gets you part of the way there, but for audit-grade questions - "which workbooks does this ex-employee still have Explorer access to?" - going straight to the repository is often the more direct route.

This is a walkthrough of that approach: which tables actually matter, the constraints I built under on a recent enterprise deployment, and the patterns that kept the queries both correct and safe to run against a production repository.

---

## Why go around the Metadata API at all

The Metadata API is the supported, forward-compatible way to ask Tableau about content relationships - and it's the right default. But it has gaps that matter for governance work: historical view-count trends, raw permission-capability rows (not just resolved "can view" booleans), and anything that needs to be joined against an external entitlement source in a single SQL statement. When the question is inherently relational - "join our HR org chart to Tableau's permission grants" - a direct repository query is often simpler than stitching together several API calls.

> [!IMPORTANT]
> The repository schema is officially documented by Tableau as the *Repository Data Dictionary*, refreshed per release. Treat it as the source of truth for column meanings - table and view names can shift slightly between versions, and undocumented internal tables should be assumed unstable.

---

## The tables that actually matter

Most governance questions only touch a handful of the repository's tables and admin views:

* **workbooks, views, projects** - the content hierarchy
* **users, groups, groupusers** - identity and group membership
* **Permission-capability tables** - the raw grants before Tableau resolves them into an effective "can view / can edit" decision
* **historical_events** and the admin views built on top of it - the closest thing to an activity log for view opens, extract refreshes, and subscription sends
* **sites** - essential the moment you're working across more than one site, since almost every other table is site-scoped

> [!TIP]
> The admin views (the ones Tableau ships specifically for querying, rather than the raw internal tables) are worth defaulting to first - they already handle a lot of the join logic and are more stable across upgrades than reaching into internal tables directly.

---

## Working under real constraints: no CTEs, no recursion

On one deployment, the review process I was working under ruled out common table expressions (CTEs) and recursive queries entirely - reviewers wanted every query plan predictable and every join visible inline, not hidden behind a WITH clause. That's a reasonable constraint for a shared production database, but it does mean rewriting the patterns most people reach for automatically.

In practice, this comes down to pushing the "staging" step CTEs usually do into a scalar or correlated subquery instead. For example, resolving "the most recent view timestamp per workbook" without a CTE:

```sql
SELECT
    w.name AS workbook_name,
    (
        SELECT MAX(he.created_at)
        FROM historical_events he
        JOIN views v ON v.id = he.view_id
        WHERE v.workbook_id = w.id
    ) AS last_viewed_at
FROM workbooks w
WHERE w.project_id = (
    SELECT id FROM projects WHERE name = 'Insurance Analytics'
);
```

It reads less elegantly than the CTE version, but it kept every intermediate result inline and auditable, which was the actual point of the constraint.

> [!WARNING]
> Correlated subqueries like this can execute once per outer row, so they need to be scoped tightly (as above, to a single project) rather than run unfiltered across the whole repository.

---

## Verifying against the data dictionary before trusting a result

Repository columns aren't always named the way you'd guess. A field that looks like a boolean "is active" flag might actually track something narrower, like whether a user's license was provisioned versus whether they've ever logged in. Before shipping any query that governance or compliance would act on, I cross-check every non-obvious column against Tableau's published data dictionary rather than inferring meaning from the column name alone - it's a five-minute check that avoids reporting a false positive on something like license reclamation or access review.

---

## Putting it together: an entitlement-based RLS coverage check

A common ask: "show me every workbook that contains PII fields but does not have row-level security applied." That's a natural fit for this approach, since it needs to join content metadata against permission grants in one pass:

```sql
SELECT w.name AS workbook_name, p.name AS project_name
FROM workbooks w
JOIN projects p ON p.id = w.project_id
WHERE w.id NOT IN (
    SELECT DISTINCT workbook_id
    FROM workbook_rls_flags   - example: an internal tagging view
    WHERE has_entitlement_rls = true
)
AND p.name IN ('HNW Client Reporting', 'Product Performance');
```

The specific tagging mechanism will differ per environment (some teams tag RLS coverage in a naming convention, others in a metadata table maintained alongside deployment), but the shape of the query - anti-join content against a coverage source, scoped to sensitive projects - is reusable anywhere entitlement-based RLS matters.

---

## Key learnings

* **Default to Tableau's admin views** over raw tables; they're the more upgrade-stable surface.
* **Losing CTEs isn't just a syntax inconvenience** - it forces every intermediate step to stay visible, which is often exactly what a security reviewer wants from a query touching a production repository.
* **Never trust a column name at face value.** Verify against the current data dictionary, since it's revised per Tableau release.
* **historical_events retention is configurable server-side** - confirm the retention window before treating "no events found" as "never viewed."
