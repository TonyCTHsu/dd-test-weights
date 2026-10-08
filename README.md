# dd-test-weights

Duration weights for dd-trace-rb unit test batch scheduling. Kept out of the
tracer repo so the generated data does not pollute its diffs, and so the
refresh can run on a schedule instead of relying on a manual commit.

## Data flow

```
dd-trace-rb Unit Tests run          this repo                     dd-trace-rb
  uploads timings-*     cron job downloads artifacts,  matrix generation
  artifacts on master    aggregates medians, commits     fetches ci_task_timings.json
                         ci_task_timings.json            from raw.githubusercontent.com
```

## Refresh process

Automated by `.github/workflows/update-weights.yml` (daily cron, or manual
via `workflow_dispatch`). The job:

1. Lists recent successful `Unit Tests` runs on `DataDog/dd-trace-rb@master`.
2. Downloads their `timings-*` artifacts (requires `PAT_TOKEN` secret, see below).
3. Runs dd-trace-rb's own `github:update_task_timings` rake task from a
   checkout of `DataDog/dd-trace-rb` (no code is duplicated here).
4. Commits the new `ci_task_timings.json` if it changed.

New test tasks run on fallback estimates in dd-trace-rb until the next
refresh records real durations. Deleted tasks leave stale entries; they are
looked up by key, so they are harmless bloat.

## Secrets

`PAT_TOKEN`: a GitHub token that can read workflow artifacts from the public
`DataDog/dd-trace-rb` repository (a classic PAT with `repo` scope works; the
repo is public so no write access is needed).

## Consumer contract

dd-trace-rb fetches `ci_task_timings.json` from this repo's default branch
at matrix-generation time. Consumers must tolerate a fetch failure and fall
back to their own estimate defaults; CI must not break if this repo is
unavailable or stale.

## Upgrade path

This file-based store is a stand-in for a service. When Datadog CI Visibility
or Test Optimization ships a distribution API, replace the fetch URL on the
consumer side and retire this repo. The interface is deliberately one GET
request.

## Ownership

Owner: tony.hsu. If the cron job stops running, weights silently go stale;
check the latest commit date on `ci_task_timings.json` before debugging
batch balance.
