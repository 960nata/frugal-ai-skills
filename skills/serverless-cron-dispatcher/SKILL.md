---
name: serverless-cron-dispatcher
description: Run many fine-grained scheduled jobs on a platform with few/limited cron slots by using one hourly dispatcher, a job_schedule table, a pure timezone-aware "is due" function, and idempotent batch leases in Postgres. Use when cron limits are hit (Vercel Hobby, free tiers), jobs run twice or never, jobs need per-timezone hours or trading-day rules, or a crashed batch must be safely retried.
---

# Serverless cron dispatcher

Instead of one platform cron per job, run **one dispatcher every hour** (from the
platform cron and/or a GitHub Actions `schedule`). The dispatcher reads a
`job_schedule` table and enqueues whatever is due. Being idempotent per calendar hour,
two triggers in the same hour still produce one dispatch.

## Tables

```sql
create table job_schedule (
  job_name text primary key,
  hours_of_day int[] not null,          -- local hours, e.g. {17}
  timezone text not null,               -- 'Asia/Jakarta', 'America/New_York', 'UTC'
  trading_days_only boolean default false,
  last_run_at timestamptz
);
create table job_run (
  id serial primary key,
  job_name text not null,
  batch_key text not null,              -- e.g. '2026-09-18T16' (local hour slot)
  status text not null,                 -- running | success | partial | failed
  started_at timestamptz, finished_at timestamptz,
  unique (job_name, batch_key)
);
```

## Pure "is due" function (no I/O, no global clock — inject `now`)
- Convert `now` to the job's timezone with `Intl.DateTimeFormat(..., { timeZone })`.
- Build `slot = YYYY-MM-DDTHH` in that zone, plus `hour` and `weekday`.
- Due when `hours_of_day` includes `hour`, (not `trading_days_only` or weekday 1–5),
  and `last_run_at` is not already in this slot.
- Unit-test with fixed dates across DST and midnight boundaries.

## Idempotent lease (one statement)
```sql
insert into job_run (job_name, batch_key, status, started_at)
values ($1, $2, 'running', now())
on conflict (job_name, batch_key) do update
  set status = 'running', started_at = now(), finished_at = null
  where job_run.status in ('failed', 'partial')
     or (job_run.status = 'running' and job_run.started_at < now() - interval '15 minutes')
returning id;
```
No row returned → someone else owns it, or it already succeeded. The stale-lease
window (15 min) must **outlive** the function's max duration.

Dependent jobs are released only after confirmed `success`:
```ts
async function recoverBatch(ops) {
  if (!(await ops.claim())) return (await ops.status()) === 'success'
  return (await ops.run()).status === 'success'
}
```

## Triggers
- GitHub Actions (`on.schedule: cron: '5 * * * *'`) calling `/api/cron/dispatcher`
  with `Authorization: Bearer $CRON_SECRET`; skip gracefully if secrets are unset.
- Heavy jobs (rendering, long scraping) run **in Actions itself**, not in the function.
- Optional queue (QStash/SQS) for fan-out with retries; retries hit the same lease.

## Checklist
- [ ] Endpoint rejects requests without the cron secret
- [ ] Double trigger in one hour → one run (test it)
- [ ] Crash mid-run → next hour reclaims after lease expiry
- [ ] Timezone logic is pure and tested
