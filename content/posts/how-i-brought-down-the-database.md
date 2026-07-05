---
title: "How I Brought Down the Database (Twice), So You Don't Have To"
date: 2026-07-05T09:00:00+10:00
draft: false
tags: ["postgres", "rails", "backfills", "databases", "talks"]
summary: "A BrisRails talk write-up: backfilling enormous amounts of data at Tanda, causing a failover (or two), and everything we built afterwards so it can't happen again."
---

This is the written-up version of a talk I gave at **BrisRails** about backfilling
enormous amounts of data at Tanda — how I managed to take production down, and
everything we built afterwards so it can't happen again.

If there's one thing to take away up front, it's the line I kept coming back to:

> If there's an issue, it's an issue with the *system*, not the reviewer.

Everything below is me building systems so that the next person doesn't get to
repeat my mistake.

## A little context

A quick tour of the setup, because the incident only lands once you know the shape
of things:

- **Rails:** edge. **Ruby:** 3.4.2 (mostly). **Postgres:** 15.13 at the time, 17.9 now.
- ECS containers on top of **Amazon Aurora**.
- We commit to master something like **67 times a working day**.
- Production gets scrubbed and cloned down into staging and development.
- We use `structure.sql`, not `schema.rb` — and we're past **9,000 migrations**.
- There's an org id on almost everything, which lets us flip the whole app into
  read-only mode via Flipper.
- The monolith now runs in **three regions / accounts**. Shipping a migration means
  thinking about *where* it runs.

That last point is the whole reason this story exists. We split the monolith's
database by region — and after a copy-paste-style split, you're left with each
region carrying a full copy of everyone's data. APAC data sitting in the US
database. US data sitting in APAC. All of it billable, all of it slowing down the
neighbours.

## Why delete data at all?

Two reasons.

**Cost.** Aurora storage is cheap per gigabyte — cents per million I/O requests —
until you're sitting on a **9.3 TB** database. That was roughly **$4,500 USD a
month for production storage alone**. No compute, no snapshots, no staging. Just
the volume.

**Noisy neighbours.** Fewer records in the database means less work for the
database. Around **80% of the records in one cluster were APAC** data that didn't
belong there — dead weight that every query had to step around. Deleting it meant
lower latency for the customers who actually lived in that region.

So: delete a *lot* of data, in a live database, without taking anyone down.

You can already see where this is going.

## The evolution of a backfill

Backfilling at Tanda has been on a long journey, and the incident sits right in the
middle of it. The spine of the story:

1. **Rails console, straight on the database.** (We were young.)
2. **Migration PRs that update data,** checked into git.
3. **Migrations holding a transaction lock for far too long.**
4. **Queue async jobs from a migration** (or rake tasks).
5. → a proper `BackfillRecordsJob`.

The job was a huge step up. But it had a flaw that only shows itself at scale, and
it's the flaw that bit me.

### The offset trap

The old job paginated with Kaminari — `page` and `per`. Clean, familiar, and a
performance landmine on a big table:

```ruby
records.order(:id).page(page).per(self.class.batch_size)
# → LIMIT 200 OFFSET 19_999_800  ← Postgres scans 19.9M rows just to throw them away
```

Every batch got slower than the last, because `OFFSET` makes Postgres walk and
discard every row it's skipping. Batch 1 is instant. Batch 100,000 is scanning
twenty million rows to hand you two hundred. The in-memory operations for the big
jobs would OOM before they even got that far.

### The incident

Here's what actually happened. I kicked off a cleanup — a big one. Postgres was
happily powering along in the quiet hours (quiet for the US, at least). Then the
records I'd deleted pushed the table past its **autovacuum threshold**, and off went
the autovacuum on a table that, with its indexes, was **864 GB**.

Who would win: my tidy little deletion job, or an autovacuum on a 864 GB table on a
database already under pressure?

Neither, it turns out. I caused a **failover**. I'd heard of failovers before; this
was the first time I'd *caused* one. And because it happened **twice**, the reader
and writer in the Aurora cluster ended up swapped back exactly where they started.
A perfect facepalm.

I'd deleted maybe **a billion records** at that point — and this was only the
*second* of the tables that needed cleaning. Long way to go.

### "Have you tried upgrading the database?"

We opened a case with AWS support, who were genuinely helpful. The best they could
offer, after all the back-and-forth, was:

> Just upgrade the database.

Ah yes. So simple. Why didn't I think of that.

To which we replied: already doing it. So we upgraded first (15 → 17), *then* went
back to deleting data — this time with a comically safe batch size of **200
records**. As I gave this talk, that job was still ticking away in the background,
quietly cleaning up.

## What we built so it can't happen again

The fix wasn't "be more careful." The fix was a job that *can't* fall into the
offset trap and *can* be stopped instantly. Enter `BackfillRecordsKeysetJob`.

Keyset pagination uses the primary key to jump through records instead of counting
past them. And a primary key isn't just a label — Postgres enforces it with a real,
unique B-tree index. So "use the index, Luke": walk the PK directly.

```ruby
scope = records.reorder(:id)
scope = scope.where("#{table}.id > ?", last_id) if last_id
scope.limit(self.class.batch_size)
# → WHERE id > 19_999_800 LIMIT 200  ← index seek, constant cost
```

The difference, framed as a question to the terminal:

```
> show me batch 100,000

OLD                                  NEW
─────────────────────────────────    ─────────────────────────────────
LIMIT 200 OFFSET 19_999_800          WHERE id > 19_999_800 LIMIT 200
Plan: Seq Scan + Limit               Plan: Index Scan
Rows scanned: 20,000,000             Rows scanned: 200
Time: gets worse every batch         Time: constant
```

And, just as importantly, a kill switch — the thing the old job never had:

```
> the backfill is killing prod, what do I do?

OLD                                  NEW
─────────────────────────────────    ─────────────────────────────────
1. Page the on-call                  Flipper.enable(:my_backfill_job)
2. Find the job in DelayedJob
3. Delete each queued run by ID
4. Hope no orgs were mid-batch
```

The full comparison:

| | **Old: `BackfillRecordsActiveJob`** | **New: `BackfillRecordsKeysetJob`** |
|---|---|---|
| Pagination | Offset (`page` + `per`) via Kaminari | Keyset (`WHERE id > last_id`) |
| Per-batch cost | Grows with page number — Postgres scans every skipped row | Constant — uses the PK index |
| `WHERE` in `#records` | Forbidden — the result set shifts between pages | Safe — we track the last actual ID processed |
| Pause / kill switch | None — once queued, runs to completion | Flipper flag per subclass |
| Max pause | n/a | 3 days, then gives up cleanly |
| Direction | Forward only | Forward or reverse — pairs with a trigger filling new rows |
| `handle_records` gets | `Array` (materialized) | `ActiveRecord::Relation` — can `update_all` directly |
| Sorbet | Required boilerplate | Dropped |

The deletion side ended up nice and clean too — once I'd triple-checked I hadn't
been quietly betrayed by a default scope. Then I merged.

### Did it work?

It did. We chose the copy-paste split, and I've been clawing back the cost from the
peak ever since. The US database is now edging under **6 TB**, saving roughly
**$1,300 USD a month** — and that's with only three tables cleaned up so far.

## The second act: everything else that was lurking

Once you go spelunking in a database this size, you find neighbours.

### Our integers are running out

Anyone remember the pre-Rails-5.1 days, when `integer` was the default primary key?
Some of our oldest tables are getting dangerously close to the **2.1 billion** ceiling.

The emergency hotfix, if you ever hit it: integer columns can go *negative*, so you
flip the sequence and buy yourself another 2.1 billion numbers. The real fix is a
zero-downtime **bigint upgrade** — Buildkite and a few large Rails shops have
published playbooks for this. It's "easy," as in about eight gnarly PRs each.

The nice part: because we were *already* cleaning up data in these tables, we get to
do the bigint upgrade, reshuffle the columns, and land the records into fresh tables
with none of the old bloat — all in one pass.

### Column ordering actually matters

Postgres pads columns for alignment, so the *order* you declare them in changes how
much space each row takes. We wrote a RuboCop rule to enforce column ordering in
`create_table`, following [GitLab's guidance][gitlab-ordering]. Fatkodima has noted
that reordering the columns on their PaperTrail versions table could save *hundreds
of GB*.

The caveat: when you're a hammer, everything looks like a nail. There's no point
retroactively reshuffling every table — you'd never keep up. (At least until PG19
ships a repack that reorders columns for you.)

### Columns Rails forgot but the database didn't

Someone at Intercom noticed they were shipping **terabytes** of data a day because
of `ignored_columns` left behind. Rails may give up on a column, but the database
still serializes and sends it over the wire on every read.

So I built a feature-flag component into the standard Rails `ignored_columns` DSL:
a developer gets *assigned* an ignored column with an expiry date, and CI fails when
that date passes — forcing the column to actually get dropped. Fatkodima has a gem
for this too; mine just adds the flag-and-expiry piece.

### Cleaning up migrations

With 9,000+ migrations and `structure.sql` spinning up every test database, the
migration folder is its own liability. Every month, Dependabot kicks off a script
that removes stale migrations older than 90 days. I've open-sourced it:

**→ [rails-migration-cleanup](https://github.com/benaitcheson/rails-migration-cleanup)**

## What's next

- **Schema rectification.** We don't run pganalyze in every region, so drift
  between the three regional databases is invisible today. The plan: diff the
  `pg_dump` of each prod DB against the others and catch divergence before it bites.
- **A system-aware backfill.** The next generation of the job decides *for itself*:
  what batch size to use based on current DB load, whether the records it's about to
  touch are caught up in an in-flight vacuum (and backs off if so), and auto-throttles
  when replication lag, CPU, or lock waits cross a threshold. Today the kill switch is
  a binary Flipper flag. GitHub-style auto-throttling is where the big shops have
  landed instead of manually paging a human at 2am.
- **The settings table.** It's 48 columns, and it's read tens of thousands of times a
  minute. We can't just `add_column ... default:` to it — the `add_column` is safe in
  itself, but it grabs an `AccessExclusiveLock`, and then every SELECT piles up behind
  it. We're weighing caching, shadow tables, and out-of-hours migrations. If your shop
  has fought this one, come find me.

## Closing thought

We chose the copy-paste option, took the database down a couple of times getting the
data back under control, and came out the other side with better tools than we went
in with. Whatever DHH and Intercom do, we'll probably do six months later.

And the moral, as always:

> Don't optimise before you need to.

Thanks to everyone who helped me dig out of this — Cam for pointing me at keyset in
the first place, and the folks whose open-source work I keep leaning on. See you at
Ruby Retreat.

[gitlab-ordering]: https://docs.gitlab.com/ee/development/database/ordering_table_columns.html
