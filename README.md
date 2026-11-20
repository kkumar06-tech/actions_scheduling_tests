# GitHub Actions scheduler delay test

14 test workflows to measure how reliably GitHub Actions starts scheduled runs,
compared with runs triggered by an external scheduler via `workflow_dispatch`.

## Structure

```
.github/workflows/
  _light.yml  _medium.yml  _heavy.yml  _control.yml   reusable workloads
  n01 … n10                                           native (schedule:)
  e11 … e14                                           external (workflow_dispatch)
workload/
  light.py            ~seconds of hashing / JSON work
  medium/             pytest suite, CPU-bound, scaled by WORK_FACTOR
  heavy.sh            generate, compress, verify a backup-like archive
```

| # | Workflow | Cron (UTC) | Workload |
|---|---|---|---|
| 1 | n01-native-5min | `*/5 * * * *` | light |
| 2 | n02-native-30min | `19,49 * * * *` | medium |
| 3 | n03-native-hourly-00 | `0 * * * *` | medium |
| 4 | n04-native-hourly-05 | `5 * * * *` | medium |
| 5 | n05-native-hourly-19 | `19 * * * *` | medium |
| 6 | n06-native-hourly-37 | `37 * * * *` | medium |
| 7 | n07-native-hourly-51 | `51 * * * *` | medium |
| 8 | n08-native-twice-daily | `19 */12 * * *` | heavy |
| 9 | n09-native-daily | `0 0 * * *` | heavy |
| 10 | n10-native-control-37 | `37 * * * *` | control |
| 11 | e11-external-5min | external, every 5 min | light |
| 12 | e12-external-hourly-00 | external, hourly :00 | medium |
| 13 | e13-external-hourly-19 | external, hourly :19 | medium |
| 14 | e14-external-daily | external, daily 00:00 | heavy |

## Repository variables (Settings → Secrets and variables → Actions → Variables)

| Variable | Default | Purpose |
|---|---|---|
| `TEST_RUNNER` | `ubuntu-latest` | runner label for all jobs, e.g. a self-hosted label |
| `MEDIUM_WORK_FACTOR` | `150` | amount of work in the medium test suite |
| `HEAVY_DATA_MB` | `500` | data size for the heavy workload |

Tune `MEDIUM_WORK_FACTOR` and `HEAVY_DATA_MB` once on the real runner, then
keep all values fixed for the 14 days.

## Rules

- Scheduled workflows only run from the default branch (`main`).
- Each schedule starts firing as soon as its file is on `main`.
- No `concurrency` groups: they cancel pending runs and would hide missed ones.
- All native workflows also have `workflow_dispatch` for manual testing.
  Manual runs have `event=workflow_dispatch` and are filtered out in the analysis.
- External workflows take a `dispatch_id` input, shown in the run name, so each
  run can be matched to the scheduler's log entry.

## Triggering an external workflow

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/actions/workflows/e11-external-5min.yml/dispatches \
  -d "{\"ref\":\"main\",\"inputs\":{\"dispatch_id\":\"$(date -u +%Y%m%dT%H%M%SZ)\"}}"
```
