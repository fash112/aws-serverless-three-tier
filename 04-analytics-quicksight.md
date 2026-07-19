# Analytics Layer — Data Visualization with Amazon QuickSight

**Standalone build** · Services: Amazon S3, Amazon QuickSight
**Build time:** ~1 hour

---

## Goal

Connect a dataset sitting in S3 to a business intelligence tool and build visualizations from it — the analytics path over cloud storage, separate from the three-tier application.

---

## Architecture

```
   S3 bucket
   ├─ dataset (Netflix titles: type, release_year, listed_in, …)
   └─ manifest.json  ── describes location + structure
         │
         │  S3 URI to the manifest
         ▼
   Amazon QuickSight
   ├─ S3 data source
   ├─ analysis
   └─ visualizations + filters
```

---

## The manifest file is the whole trick

QuickSight does not simply point at a bucket. It reads a `manifest.json`, which acts as a map: it tells QuickSight where the files are and how they are organised, describing what each piece of data looks like so the service knows how to interpret it.

The manifest had to be edited to carry the correct S3 URI for the dataset. An outdated URI in the manifest means QuickSight is directed at the wrong address and the connection fails — the file is a pointer, and a stale pointer is worse than no pointer.

Two objects therefore live in the bucket: the dataset itself and the manifest describing it.

**Connecting the source.** From the QuickSight datasets page, a new S3 data source was created and the manifest's S3 URI supplied. QuickSight also needs explicit permission to access that specific bucket — enabling S3 access at the account level is not enough, the bucket has to be authorised.

Account creation itself was quick and cost nothing, running on the 30-day free trial.

---

## Visualizations built

**Records by release year and type.** Built by dragging fields into position — release year on the y-axis, with `type` (movie or TV show) as the grouping variable. The result is a breakdown of movies versus TV shows across release years.

**Genre distribution with a filter applied.** A filter excluding titles released before 2015 narrowed the set to 171 records, visualised as a donut showing distribution across categories including Thrillers, TV Comedies, and Action.

Filters matter here because they specify the exact subset of data worth analysing and exclude everything irrelevant. The pre-2015 exclusion turned a full-catalogue view into a recent-content view, which is a different and more answerable question.

---

## Honest reflection

This build was fast and error-free, which is worth being clear-eyed about: the dataset was complete and well-formed before it reached S3. Real analytics work rarely starts from that position. The failure modes that dominate day to day — inconsistent schemas across source files, nulls in the columns you need to group by, duplicate rows inflating counts — never appeared, because there was nothing wrong with the data.

The transferable piece is the connection pattern rather than the charting: object storage, a manifest describing structure, a BI tool reading through explicit permissions. That pattern recurs across cloud data platforms regardless of which vendor's BI layer sits on top.

---

## What I'd add next

- **Data quality checks before the visualisation layer** — null and completeness checks, duplicate detection against a natural key, and schema validation on ingest, so the dashboard cannot silently render bad data
- **SPICE ingestion with a refresh schedule**, rather than direct query
- **Calculated fields** for derived measures, kept in the dataset rather than duplicated across analyses
- **A dataset with real problems in it**, to make the exercise representative
