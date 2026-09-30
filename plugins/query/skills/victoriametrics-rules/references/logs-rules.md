# vlogs rules

A rule which queries VictoriaLogs sets `type: vlogs` on its group, and its `expr` is a LogsQL
stats query, because vmalert calls `/select/logsql/stats_query`.

```yaml
groups:
  - name: log-errors
    type: vlogs
    interval: 5m
    rules:
      - alert: ErrorSpike
        expr: 'level:error | stats count() as errors | filter errors:>50'
```

Put the threshold in the expression with a `filter` pipe. The rule fires on the rows the query
returns, so a bare `stats count()` always returns one row and always fires.

**Reference the count as `{{ $value }}`, never as `{{ $labels.<alias> }}`.** The stats alias does
not become a label of that name. vmalert renames it to `stats_result`, so a rule whose expression
says `count() as errorCount` produces the labels
`{alertname, alertgroup, app, stats_result="errorCount"}` and carries the count in the value. An
annotation written as `{{ $labels.errorCount }}` renders empty, and nothing reports the mistake.
Use `{{ $labels.stats_result }}` if you want the alias name itself.

Write the expression without a `_time:` filter. vmalert supplies the time range from the group
interval when the query carries no time filter of its own. Add `_time:5m` and it stops, and a
range query then fails with `range query is not supported for LogsQL expression ... because it
contains time filter`.

When checking the expression yourself, send `start` and `end` as well as `time`, scoped to the
group interval, exactly as vmalert does. Without them the query counts every row in storage
rather than the interval, so a quiet window and a busy one return the same number:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  --data-urlencode 'query=level:error | stats count() as errors' \
  --data-urlencode 'time=2026-09-22T01:30:00Z' \
  --data-urlencode 'start=2026-09-22T01:25:00Z' \
  --data-urlencode 'end=2026-09-22T01:30:00Z' \
  "$VM_LOGS_URL/select/logsql/stats_query" | jq '.data.result[].value[1]'
```

Workflow B in SKILL.md applies to a `vlogs` rule unchanged. `/api/v1/rules` reports
`datasourceType: vlogs` for it, and `would_fire.py` does not: it queries the Prometheus
range API, so use the `stats_query` call above for a logs rule instead.
