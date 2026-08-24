# Case Study: Financial Data Warehouse on AWS

**A cloud-provisioned data pipeline tracking company fundamentals and stock performance, built with Terraform, AWS Lambda, and dbt snapshots**

---

## Problem

Most portfolio ETL projects run entirely on a laptop — real, useful practice, but it skips the part of the job that separates "I can write a pipeline" from "I can provision and operate infrastructure a team would trust in production." This project was built to close that gap: provision real AWS infrastructure with Terraform, ingest real regulatory and market data, and implement a genuine dimensional-modeling pattern (Slowly Changing Dimension Type 2) that most portfolio projects never touch because it requires data that actually changes over time.

The specific question this project answers: *how has each AI/tech company's stock performed recently, broken down by industry sector — and what happens to that analysis when a company's sector classification itself changes?*

## Architecture

```
SEC EDGAR API (company facts + SIC)        Alpha Vantage (daily OHLCV JSON)
         │                                          │
         ▼                                          ▼
  AWS Lambda: sec_edgar_ingest             AWS Lambda: alpha_vantage_ingest
         │                                          │
         └───────────────────┬──────────────────────┘
                              ▼
                  Amazon S3 (raw / bronze landing zone)
                              │
                              ▼
          Python bridge script (S3 → local Postgres "raw" schema)
                              │
                              ▼
      dbt: staging → snapshot (SCD Type 2) → intermediate → marts
                              │
                              ▼
          PostgreSQL analytics-ready tables (queryable via psql)
```

Triggered daily by Amazon EventBridge Scheduler. The entire AWS footprint — S3, both Lambda functions, IAM roles, CloudWatch log groups, the EventBridge schedules, and an AWS Budget alert — is provisioned and torn down with a single `terraform apply` / `terraform destroy`.

## Tech Stack & Reasoning

| Layer | Choice | Why |
|---|---|---|
| Infrastructure as Code | Terraform (AWS provider) | Reproducible, reviewable (`plan` before `apply`), fully tear-down-able |
| Compute | AWS Lambda (Python 3.12, stdlib only) | Serverless, permanently free at this usage volume, no dependency layer needed since both handlers use only `urllib` + `boto3` |
| Scheduling | Amazon EventBridge Scheduler | Modern replacement for EventBridge Rules cron; triggers both Lambdas daily |
| Storage | S3 (raw/bronze landing zone) | Versioned, lifecycle-expiring after 45 days, public access fully blocked |
| Warehouse | Local PostgreSQL (Docker) | Deliberately not RDS/Redshift — see Key Decisions |
| Transformation | dbt-core + dbt-postgres | Staging → snapshot (SCD Type 2) → intermediate → marts |
| Data sources | SEC EDGAR (free, no key) + Alpha Vantage (free tier, 25 req/day, using 8) | See the Stooq pivot story below |
| CI/CD | GitHub Actions | `terraform plan` against real scoped credentials, `dbt parse`, `pytest`, `ruff` |
| Cost control | AWS Budget alert ($5), S3 lifecycle rules, CloudWatch log retention (14 days) | No RDS/Redshift; Lambda's 1M-request/month tier is permanently free |

## Key Decisions

**Terraform over the console.** Every AWS resource in this project — down to the IAM role Lambda assumes and the Budget alert itself — is defined in code. This means the entire cloud footprint can be destroyed and rebuilt from nothing in about a minute, which matters directly for a side project's actual bill: infrastructure that only exists while you're actively demoing it is infrastructure that costs almost nothing the rest of the time.

**Local Postgres over RDS or Redshift.** Redshift Serverless has no permanent free allowance, and RDS free-tier hours eventually run out or draw down a credit balance — both are the kind of service that quietly starts billing once a trial window closes. A local Dockerized Postgres instance costs $0 forever, and its SQL dialect is close enough to Redshift's that the modeling skill transfers directly. The honest trade-off: this doesn't demonstrate operating a *cloud-hosted* database specifically, which is a reasonable next step for a future project once cost tolerance is different.

**SEC EDGAR directly, not a wrapped API.** Pulling straight from `data.sec.gov` instead of a third-party convenience wrapper meant handling real API quirks first-hand — a mandatory identifying `User-Agent` header enforced by SEC's fair-use policy, zero-padded 10-digit CIK formatting, and a JSON shape that wasn't designed to be beginner-friendly. It also meant free, unlimited access to a genuine Slowly Changing Dimension case: a company's SIC industry classification and legal name are attributes that actually change over time in SEC's own records.

**`check` strategy over `timestamp` for the dbt snapshot.** SEC's data has no trustworthy "last modified" field — the `fetched_at` timestamp only reflects when *this pipeline* pulled the data, not when SEC's underlying record changed. The `check` strategy compares actual column values (`sic_code`, `sic_description`, `company_name`) on every run instead, which is the correct choice whenever no reliable source-side timestamp exists.

**Lambda + EventBridge Scheduler over reusing Airflow.** An earlier project in this series already demonstrated Airflow-based orchestration. Repeating that pattern here would have added portfolio breadth without adding depth — serverless scheduling is a genuinely different, equally production-valid pattern, and a better fit for two small, independent, once-daily jobs that don't need DAG-level dependency management.

**A resource-scoped IAM wildcard, arrived at iteratively, not by default.** The Terraform-managing IAM policy started fully enumerated — every individual action spelled out. Across Day 1 and Day 2, applying infrastructure repeatedly surfaced missing read permissions the AWS provider needs after every create (bucket ACLs, CORS, tags, attached policies, log group tags, and more) that aren't obvious in advance. After the fourth or fifth individually-discovered missing permission on the S3 and IAM statements, the policy was deliberately switched to `s3:*` / `iam:*` — but still tightly scoped by resource name (`financial-data-warehouse-*` only). The real security boundary was always *which resources* this credential could touch, not the exhaustiveness of the action list — recognizing when that trade-off is the right call, versus when a team's security posture would require the fully enumerated version, was itself part of the exercise.

## Engineering Challenges (the interview-worthy parts)

**The Stooq pivot.** The original plan called for daily price data from Stooq's free CSV endpoint. Partway through building the ingestion Lambda, invocations started returning a 200 response containing an HTML page instead of CSV — Stooq had deployed a JavaScript proof-of-work bot-verification challenge (an Anubis-style check), which a headless Lambda function using `urllib` has no way to solve, since it requires executing JavaScript in a real browser context. This wasn't a bug in the ingestion code; it was a third-party dependency changing behavior out from under the project mid-build — a genuinely realistic production scenario, not a contrived exercise.

The response mattered as much as the diagnosis: rather than attempting to defeat the bot-verification challenge programmatically (fragile, likely a terms-of-service violation, and a bad look in a commit history), the fix was switching to a different, legitimate data source. Alpha Vantage — initially set aside during planning for its 25-requests/day free-tier cap — turned out to fit perfectly once the real requirement was understood: 8 tickers, once daily, is 8 requests, comfortably inside that limit. The full pivot included renaming the Lambda, its IAM permissions, the raw Postgres table, and the bridge script's parsing logic — and cleaning up every trace of the abandoned path, including dead S3 objects, stale IAM policy entries, and a table that was created but never populated.

**Watching SCD Type 2 actually work.** Real SEC reclassifications don't happen on a one-week timeline, so the snapshot mechanism was verified deliberately: manually updating one company's `sic_description` directly in the raw source table, then re-running `dbt snapshot`. The result was two rows for the same company — the original version with `dbt_valid_to` now populated with a real timestamp, and a new version starting exactly where the old one closed, with `dbt_valid_to` back to `NULL` as the current record. That's a materially different, more credible way to answer "how does SCD Type 2 work" in an interview than reciting a definition — and it visibly changed downstream analytics too: `mart_sector_performance` split one company into a new sector bucket automatically, with no manual query changes required.

**Iterative least-privilege IAM debugging.** Standing up the S3 bucket, IAM role, and budget alert on Day 1 alone took roughly a dozen apply-fail-diagnose-fix cycles, each surfacing one specific missing read permission (`s3:GetBucketPolicy`, `s3:GetBucketAcl`, `s3:GetBucketCors`, `iam:ListRolePolicies`, `iam:ListAttachedRolePolicies`, `logs:ListTagsForResource`, `logs:DeleteLogGroup`, and more) that the AWS provider needs to refresh Terraform's state after creating a resource — none of which are obvious from Terraform's documentation in advance. Several resources ended up marked "tainted" from failed read-backs on otherwise-successful creates, requiring `terraform untaint` rather than letting Terraform destroy and recreate resources that were actually fine. This was slow in the moment and is exactly the kind of experience that produces a real, specific answer to "how do you approach least-privilege IAM" instead of a textbook one.

**CI debugging, layer by layer.** Getting GitHub Actions fully green surfaced a chain of small, realistic issues: `terraform fmt -check` failing on whitespace drift accumulated across sessions; a `dbt compile` step that turned out to need a live database connection it would never have in CI, fixed by switching to `dbt parse` (which validates structure without connecting); a one-character typo in a CI-only `profiles.yml` (`financial warehouse` instead of `financial_warehouse`) that produced a "profile not found" error despite the file existing right there; and — the most instructive one — a fix that was correctly written locally but never actually committed, caught only by running `git status` before assuming a push had happened. That last one is a good discipline to name directly: verify the state of the repository rather than trusting memory of what you meant to do.

**The versioned-bucket teardown gotcha.** Tearing down the AWS infrastructure at the end of the project hit `BucketNotEmpty` on `terraform destroy`, even though the bucket looked empty in a normal listing. S3 versioning — deliberately enabled from Day 1 as a durability best practice — means deleted objects don't actually disappear; they get a delete marker while every prior version remains. Fully emptying a versioned bucket requires explicitly deleting every object version *and* every delete marker before AWS will allow the bucket itself to be deleted. This is a common, real trade-off: some Terraform configurations set `force_destroy = true` specifically to skip this friction, at the cost of a `destroy` command being able to silently wipe historical data without confirmation. This project kept the safer default and paid for it with one extra manual cleanup step at the end — a defensible trade worth being able to explain.

## Results

- Real regulatory and market data flowing end-to-end, daily, with zero manual intervention: SEC EDGAR fundamentals and Alpha Vantage price history for an 8-company AI/tech basket (NVDA, MSFT, GOOGL, META, AMD, AVGO, PLTR, TSLA)
- A working, tested, and *proven* SCD Type 2 implementation — not just modeled, but demonstrated with a real before/after change
- Sector-performance analytics that correctly reflect dimensional history rather than a static snapshot
- Full CI/CD: infrastructure changes are plan-checked against real AWS credentials, dbt models are structurally validated, Python is linted and unit-tested — all on every push
- A fully reproducible AWS footprint: provisioned from nothing and torn down to nothing, verified both directions
- Effectively $0 in AWS spend across the entire build, independent of free-tier/credit eligibility, by design rather than by luck

## What I Learned

- A dependency you didn't choose can change out from under you mid-project — the right response is diagnosing it properly and finding a legitimate alternative, not trying to force the original plan to keep working
- Least-privilege IAM is discovered iteratively against real error messages, not written correctly from a blog post on the first try — and knowing when to stop enumerating individual actions in favor of resource-scoped wildcards is itself a judgment call worth being able to defend
- Reading about `dbt_valid_from`/`dbt_valid_to` is nothing like watching a row actually split into two after a real source change — the concept only fully clicks once you've caused it and observed it yourself
- "Did I actually commit that?" is worth checking with `git status` rather than assuming — a fix that exists only in a text editor isn't a fix
- Terraform's plan/apply discipline (read the diff before you execute) is the same instinct as reviewing a code diff before merging, applied to infrastructure — and it caught real problems (tainted resources, unexpected recreates) multiple times over the course of this build

## Future Improvements

- Remote Terraform state (S3 backend + DynamoDB locking) so multiple engineers could safely run `apply` without conflicting
- A Schema Registry-style contract for the S3 bronze layer, so downstream consumers have a guaranteed schema instead of an implicit one
- A move to Redshift or Athena/Glue once data volume justified leaving a single local Postgres instance
- AWS Secrets Manager instead of Terraform variables/environment variables for the Alpha Vantage key and database credentials
- Airflow-managed orchestration once there are enough independent ingestion jobs to justify DAG-level dependency management
- A second snapshot using the `timestamp` strategy, for direct comparison against the `check` strategy used here

## Code

GitHub: `github.com/YOUR-USERNAME/financial-data-warehouse-aws`
