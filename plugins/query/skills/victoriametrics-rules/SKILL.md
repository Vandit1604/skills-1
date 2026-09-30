---
name: victoriametrics-rules
description: >
  Inspect, debug and write vmalert alerting and recording rules against a running vmalert, for
  VictoriaMetrics and VictoriaLogs. Use this whenever the user mentions an alert that did not
  fire, fired late or fired too often, asks what a rule is doing, wants a new alerting or
  recording rule written, or asks when a rule would have fired over a past window, even if they
  do not say "vmalert" or "rule". Also use it before hand-writing any rule YAML or any curl
  against vmalert, because the live instance already holds the rule state, the last evaluations
  and the exact query it sent. Triggers on: alerting rule, recording rule, vmalert, alert did
  not fire, alert fired late, flapping alert, rule health, rule state, for duration,
  keep_firing_for, notifier, Alertmanager, LogsQL alert, vlogs rule, threshold, paging.
allowed-tools: Bash(curl:*), Bash(jq:*), Bash(python3:*), Read
---

# vmalert rules

Everything here is an HTTP call against a running vmalert and its datasource. Nothing needs the
`vmalert` or `vmalert-tool` binaries, and nothing writes to the datasource.

Pick the workflow by the question:

- An alert fired, did not fire, fired late or flapped in the past: Workflow A. vmalert's record
  of what it decided is the `ALERTS` and `ALERTS_FOR_STATE` series it writes to
  `-remoteWrite.url`.
- A rule is failing, quiet or firing now: Workflow B, then the checklist in "When a Rule Should
  Be Firing Now and Is Not".
- A new rule to write: Workflow C.
- A rule with `type: vlogs`, or any LogsQL rule: read `references/logs-rules.md` before writing or
  checking it.
- A rule with `record:` instead of `alert:`: read `references/recording-rules.md`.
- An endpoint field or parameter not shown here: read `references/api-reference.md`.

## Environment

```bash
# $VMALERT_URL   - the running vmalert, e.g. export VMALERT_URL="http://vmalert.example.com:8880"
# $VM_METRICS_URL - the datasource vmalert queries, for checking an expression yourself
#   single: export VM_METRICS_URL="http://localhost:8428"
#   cluster: export VM_METRICS_URL="https://vmselect.example.com/select/0/prometheus"
# $VM_LOGS_URL    - VictoriaLogs base URL, for a vlogs rule
# $VM_CURL_CONFIG - curl config file with an auth header. Leave unset for a local instance.
```

## Auth Pattern

Every command loads auth from a curl config file, which works for both remote and local:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/api/v1/rules" | jq .
```

When `VM_CURL_CONFIG` is unset, curl reads `/dev/null` and sends no auth header. When set, it must
point to a mode-0600 curl config file containing `header = "Authorization: Bearer <token>"`. Never
print its contents.

`scripts/would_fire.py` calls curl the same way, so it reads the same file and needs no token on
its command line.

## Critical Rules

- **Do not run `vmalert -replay` to answer "when would this have fired".** Replay writes its
  results to `-remoteWrite.url`, so against a real deployment it injects synthetic `ALERTS` and
  `ALERTS_FOR_STATE` into production storage. Workflow B step 4 answers the question and writes
  nothing.
- For a question about the past, read `ALERTS` and `ALERTS_FOR_STATE` first. vmalert keeps only
  the last 20 evaluations of a rule in memory (`-rule.updateEntriesLimit`), which at a 1m
  interval is 20 minutes, and debug mode is off by default.
- Read the rule's own evaluation history before theorising. `samples: 0` means the expression
  returned nothing, which is a different bug from a threshold never crossed.
- Check `health` and `lastError`, not only `state`. A rule failing at evaluation reports
  `state: inactive`, which looks like a quiet rule.
- Never print the contents of `$VM_CURL_CONFIG`.

## Workflow A: Why Did an Alert Fire, Not Fire, or Fire Late

Use this for any window older than the rule's in-memory history. It reads what vmalert decided
and puts it next to what the data says. Copy this checklist and tick it off:

```
- [ ] 1. Rule settings, and -remoteWrite.url in /flags
- [ ] 2. ALERTS and ALERTS_FOR_STATE over the window
- [ ] 3. would_fire.py and the raw values over the same window
- [ ] 4. For each series: the `for` verdict, then what vmalert saw (table in step 3)
```

### 1. Read the rule and check that vmalert records its decisions

Take the rule's settings from vmalert, not from memory or the rule file:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/api/v1/rules" \
  | jq -r '.data.groups[] | .interval as $i | .eval_delay as $d | .rules[] | select(.name=="QueueBacklog")
           | "health=\(.health)\terr=\(.lastError)\texpr=\(.query)\tfor=\(.duration)s\tinterval=\($i)s\tkeep_firing_for=\(.keep_firing_for // 0)s\teval_delay=\(if $d then "\($d)s" else "flag" end)"'
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/flags" \
  | jq -Rr 'select(startswith("-remoteWrite.url") or startswith("-rule.evalDelay"))'
```

```
health=ok	err=	expr=queue_depth > 100	for=120s	interval=30s	keep_firing_for=0s	eval_delay=flag
-remoteWrite.url="secret"
-rule.evalDelay="5s"
```

`eval_delay=flag` means the group sets none, so `-rule.evalDelay` applies: the value in `/flags`,
or 30s when `/flags` does not list it.

`/flags` lists only the flags set on the command line and hides URLs as `"secret"`. No
`-remoteWrite.url` line means vmalert kept no record, so skip step 2. Tell the user that
`-remoteWrite.url` stores the alert state and `-remoteRead.url` restores it after a restart.

The line does not say where the series go. Query `$VM_METRICS_URL` first. If step 2 returns
nothing for a rule the user saw fire, ask where `-remoteWrite.url` points.

### 2. Read what vmalert decided

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  --data-urlencode 'query=ALERTS{alertname="QueueBacklog"}' \
  --data-urlencode 'start=2026-09-30T15:12:00Z' \
  --data-urlencode 'end=2026-09-30T15:24:00Z' \
  --data-urlencode 'step=30s' \
  "$VM_METRICS_URL/api/v1/query_range" \
  | jq -r '.data.result[] | (.metric | del(.__name__, .alertname, .alertgroup, .alertstate) | tostring) as $l
           | "\($l)\t\(.metric.alertstate): \([.values[][0] | todate[5:19]] | join(" "))"'
```

```
{"job":"worker","severity":"warning"}	firing: 09-30T15:17:30 09-30T15:18:00 09-30T15:18:30 09-30T15:19:00
{"job":"worker","severity":"warning"}	pending: 09-30T15:15:30 09-30T15:16:00 09-30T15:16:30 09-30T15:17:00 09-30T15:20:30 09-30T15:21:00
```

While an alert is active, every evaluation writes one `ALERTS` sample with value `1`, the alert's
labels, and `alertstate` set to `pending` or `firing`. Set `step` to the group `interval`, so each
point is one evaluation at the timestamp it evaluated for.

One series can hold several episodes, as the `pending` line does here. `ALERTS_FOR_STATE`
separates them: its value is the episode's `activeAt`, so each distinct value is one episode. An
episode fired only if `ALERTS` has `firing` points inside it.

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  --data-urlencode 'query=ALERTS_FOR_STATE{alertname="QueueBacklog"}' \
  --data-urlencode 'start=2026-09-30T15:12:00Z' \
  --data-urlencode 'end=2026-09-30T15:24:00Z' \
  --data-urlencode 'step=30s' \
  "$VM_METRICS_URL/api/v1/query_range" \
  | jq -r '.data.result[] | (.metric | del(.__name__, .alertname, .alertgroup) | tostring) as $l
           | .values | map(.[1] | tonumber) | unique[] | "\($l)\tactiveAt=\(todate)"'
```

```
{"job":"worker","severity":"warning"}	activeAt=2026-09-30T15:15:30Z
{"job":"worker","severity":"warning"}	activeAt=2026-09-30T15:20:30Z
```

The rule has `for: 2m`. The first episode went pending at 15:15:30 and fired at 15:17:30. The
second went pending at 15:20:30 and cleared before `for` elapsed, so it never fired. An episode
resolves at the first evaluation after its last point, 15:19:30 for the first one here.

No line for a series means vmalert never had an active alert for it in the window. If step 3
shows the condition true for that series, vmalert never saw the data: read the third row of the
table.

### 3. Compare with what the data says

Run Workflow B step 4 over the same window, with `expr`, `for` and `interval` from
step 1 as `--query`, `--for` and `--step`. It accepts them as printed. The exit code covers all
series, so add the user's labels to the selector, as in `queue_depth{job="batch"} > 100`, when
they ask about one:

```
__name__=queue_depth,job=batch
  condition true at 4 of 4 steps, first 2026-09-30T15:17:30+00:00, last 2026-09-30T15:19:00+00:00
  never fires: no unbroken run reaches for=120s
__name__=queue_depth,job=worker
  condition true at 10 of 12 steps, first 2026-09-30T15:15:30+00:00, last 2026-09-30T15:21:00+00:00
  fires 2026-09-30T15:17:30+00:00 (pending from 2026-09-30T15:15:30+00:00, for=120s)
```

The script prints only when the condition held. To see how high the values went, run the
expression without its comparison:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  --data-urlencode 'query=queue_depth{job="batch"}' \
  --data-urlencode 'start=2026-09-30T15:16:00Z' \
  --data-urlencode 'end=2026-09-30T15:20:00Z' \
  --data-urlencode 'step=30s' \
  "$VM_METRICS_URL/api/v1/query_range" \
  | jq -r '.data.result[] | "\(.metric | del(.__name__) | tostring)\t\([.values[] | "\(.[0] | todate[11:19])=\(.[1])"] | join(" "))"'
```

```
{"job":"batch"}	15:16:00=20 15:16:30=20 15:17:00=20 15:17:30=300 15:18:00=300 15:18:30=300 15:19:00=300 15:19:30=20 15:20:00=20
```

vmalert sees one value per evaluation, while a dashboard draws every sample. A spike which looks
2 minutes long on the dashboard can hold for only 90 seconds of evaluations. To list the raw
samples, query a range selector at the end of the window:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  --data-urlencode 'query=queue_depth{job="batch"}[2m30s]' \
  --data-urlencode 'time=2026-09-30T15:19:20Z' \
  "$VM_METRICS_URL/api/v1/query" \
  | jq -r '.data.result[] | "\(.metric | del(.__name__) | tostring)\t\([.values[] | "\(.[0] | todate[11:19])=\(.[1])"] | join(" "))"'
```

```
{"job":"batch"}	15:16:58=20 15:17:08=20 15:17:18=20 15:17:28=300 15:17:38=300 15:17:48=300 15:17:58=300 15:18:08=300 15:18:18=300 15:18:28=300 15:18:38=300 15:18:48=300 15:18:58=300 15:19:08=300 15:19:18=300
```

Read the two side by side, one series at a time. Check the `for` verdict first: a run of N true
evaluations lasts N-1 intervals, so firing needs `for` / `interval` + 1 of them. A shorter run never
fires, whatever vmalert saw. `job=batch` above holds for 4 evaluations, 90 seconds, and needs 5,
so it never fires for that reason alone. A short run still goes pending, so it still writes
`ALERTS` points. `job=batch` has none, which is a second, separate finding.

| Data | `ALERTS` | Cause |
|---|---|---|
| condition true | pending, then firing at `activeAt` plus `for` | The rule worked. If nobody was paged, check delivery in Workflow B step 5 |
| condition true | pending only | The condition cleared before `for` elapsed, as in the 15:20:30 episode |
| condition true | no point at those times | vmalert never saw it: every evaluation where the condition held writes a pending or firing point. `job=batch` above has none. If step 2 shows points for other series of the same rule at those times, vmalert was running and the rule worked, so the data arrived after the evaluation; raise `eval_delay` on the group or `-rule.evalDelay` (30s unless `/flags` lists it). No API shows when a sample arrived, so this comparison is the evidence. With no points for any series, vmalert was down or the rule was failing |
| condition false | no point | The threshold was never crossed |
| condition false | pending or firing | `keep_firing_for` held it, or the data changed after vmalert read it |

## Workflow B: What Is This Rule Doing Now

### 1. Find the rule and read its health

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/api/v1/rules" \
  | jq -r '.data.groups[].rules[]
           | "\(.name)\tstate=\(.state)\thealth=\(.health)\tsamples=\(.lastSamples)\tfetched=\(.lastSeriesFetched)\terr=\(.lastError)"'
```

```
VMUptimeHigh	state=firing	health=ok	samples=1	fetched=1	err=
```

`lastSamples` is how many samples the last evaluation returned, and `lastSeriesFetched` is how
many series the datasource read to produce them. `fetched=0` means the selector matches nothing,
so the rule can never fire whatever the threshold is.

Keep the `id` and `group_id` from this response for step 2.

### 2. Read the per-evaluation history

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  "$VMALERT_URL/api/v1/rule?group_id=$GROUP_ID&rule_id=$RULE_ID" \
  | jq -r '.updates[] | "\(.at)\tsamples=\(.samples)\tfetched=\(.series_fetched)\terr=\(.error)"'
```

```
2026-09-25T02:49:50+05:30	samples=1	fetched=1	err=null
2026-09-25T02:49:45+05:30	samples=1	fetched=1	err=null
```

vmalert keeps the last `max_updates_entries` evaluations per rule, 20 by default, set by
`-rule.updateEntriesLimit` or `update_entries_limit` on the rule. The history is in memory, so a
restart clears it. Each entry is one evaluation: `at` is the timestamp it evaluated for, and
`samples` is how many series the expression returned. A column of `samples=0` is the expression
returning nothing.

Each entry also carries a `curl` field holding the exact request vmalert sent to the datasource,
query encoding, `time` and `step` included:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  "$VMALERT_URL/api/v1/rule?group_id=$GROUP_ID&rule_id=$RULE_ID" | jq -r '.updates[0].curl'
```

Run that command to see what vmalert saw. It settles most disagreements about a rule, because it
removes the guesswork about which timestamp and step were used.

### 3. Read the firing alert and its rendered annotations

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/api/v1/alerts" \
  | jq -r '.data.alerts[] | "\(.name)\t\(.state)\tvalue=\(.value)\tactiveAt=\(.activeAt)"'
```

```
VMUptimeHigh	firing	value=45	activeAt=2026-09-25T02:49:15+05:30
```

`activeAt` is when the condition first became true, not when the alert fired. With `for: 5m` the
alert fires `for` after `activeAt`. The annotations in this response are rendered, so this is
where a broken `dashboard` link or an empty `summary` shows up.

### 4. Work out when a rule fires over a past window, read-only

Run the whole alerting expression, comparison included, as a range query. A filtering comparison
drops the samples where it is false, so every returned point is a moment the condition held:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s \
  --data-urlencode 'query=sum(rate(http_requests_total{code=~"5.."}[5m])) by (job) / sum(rate(http_requests_total[5m])) by (job) > 0.05' \
  --data-urlencode 'start=2026-09-22T00:00:00Z' \
  --data-urlencode 'end=2026-09-22T02:00:00Z' \
  --data-urlencode 'step=1m' \
  "$VM_METRICS_URL/api/v1/query_range" | jq '.data.result[].values | length'
```

Run `scripts/would_fire.py` to do that query and apply `for`. `<skill_base_dir>` is the directory
holding this SKILL.md. Run the script; there is no need to read it:

```bash
python3 <skill_base_dir>/scripts/would_fire.py \
  --url "$VM_METRICS_URL" \
  --query 'sum(rate(http_requests_total{code=~"5.."}[5m])) by (job) / sum(rate(http_requests_total[5m])) by (job) > 0.05' \
  --start 2026-09-22T00:00:00Z --end 2026-09-22T02:00:00Z \
  --step 1m --for 5m
```

```
job=api
  condition true at 58 of 58 steps, first 2026-09-22T01:03:00+00:00, last 2026-09-22T02:00:00+00:00
  fires 2026-09-22T01:08:00+00:00 (pending from 2026-09-22T01:03:00+00:00, for=300s)
```

It exits 1 when no series has an unbroken run that reaches the `for` duration, and 0 when at least
one series fires, so it works in a check. Exit 2 means the check could not run, for example the
datasource was unreachable or rejected the query: read the message, and never report exit 2 as
"the rule would not fire". `for` and `--step` accept vmalert durations such as
`1h30m` or `500ms`. A `WARNING from the datasource` line means a query limit cut the result
short: narrow the window or raise the limit before you trust the verdict. The request gives up
after 90 seconds with an error.

### 5. Confirm the alert has somewhere to go

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/api/v1/notifiers" \
  | jq -r '.data.notifiers[] | .kind as $k | .targets[] | "\($k)\t\(.address)\terr=\(.lastError)"'
```

```
static	blackhole	err=
```

A firing rule with a notifier `lastError` is a delivery problem, not a rule problem. An address
of `blackhole` means `-notifier.blackhole` is set and nothing is being sent by design.

To confirm notifications are actually leaving vmalert, read its own counters:

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s "$VMALERT_URL/metrics" \
  | jq -Rr 'select(test("^vmalert_alerts_(sent|send_errors)_total[{ ]"))'
```

```
vmalert_alerts_sent_total{addr="blackhole"} 1
vmalert_alerts_send_errors_total{addr="blackhole"} 0
```

One counter per notifier address. The counters count from vmalert's start, so read them twice and
compare. `sent_total` rising while an alert fires is proof the notification path works end to end.
`sent_total` flat with a firing alert means nothing is being delivered, and `send_errors_total`
rising tells you vmalert is trying and failing.

These counters end at Alertmanager. Whether Alertmanager routed the alert to a person is its own
question; use the `alertmanager-query` skill for it.
`vmalert_alerts_send_duration_seconds` is the same set if you need latency.

## Workflow C: Writing a New Rule

### 1. Run the expression before writing the rule around it

Use the `victoriametrics-query` skill for MetricsQL, or `victorialogs-query` for LogsQL. An
expression which returns nothing produces a rule which never fires, and no later check reports
that as an error.

### 2. Write the rule

```yaml
groups:
  - name: api-availability
    interval: 1m
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{code=~"5.."}[5m])) by (job)
            / sum(rate(http_requests_total[5m])) by (job) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} serves {{ $value | humanizePercentage }} 5xx"
          dashboard: "https://grafana.example.com/d/abc123/api?var-job={{ $labels.job }}"
```

Give every alert a `dashboard` annotation which opens the panel showing the same series, with
the alert's own labels templated into the URL. The person woken at 03:00 starts there. Step 3 of
Workflow B shows the rendered link, so check it resolves before trusting it.

`for` holds the alert until the condition has been true that long. `keep_firing_for` holds it
after the condition clears, which stops a flapping alert from resolving and re-firing.

### 3. Check when it would fire

Use Workflow B step 4 against the window of a real past incident. Silence there means the rule would
have missed it.

### 4. Deploy, reload, then verify against the live instance

```bash
curl -q --config "${VM_CURL_CONFIG:-/dev/null}" -s -X POST "$VMALERT_URL/-/reload"
```

`/-/reload` sends vmalert a SIGHUP to re-read its rule files. It may be protected by
`-reloadAuthKey`. Then run Workflow B steps 1 and 2 and confirm `health=ok`,
`lastError` empty, and `fetched` greater than zero.

## When a Rule Should Be Firing Now and Is Not

For a past window, use Workflow A. For a rule that is quiet now, work down this list.
Each step distinguishes two causes the previous one cannot.

1. `lastError` non-empty, or `health` not `ok`. The rule is failing, not quiet.
2. `fetched=0` in Workflow B step 1. The selector matches nothing. Check label names against
   the datasource.
3. `samples=0` across the history in Workflow B step 2, with `fetched` above zero. The series
   exist and the threshold was never crossed. Run the `curl` from the update and look at the value.
4. `activeAt` set but no firing alert. The condition is true but `for` has not elapsed yet.
5. The condition is true in Workflow B step 4 but the rule never went pending. Suspect late
   data, and confirm it with Workflow A. `eval_delay` on the group shifts the evaluation
   back to compensate.
6. Still unexplained. Set `debug: true` on the group or the single rule and read vmalert's log.
   It records the series count per evaluation and every state change. Debug mode is off by
   default and logs only the evaluations after you turn it on, so it helps only when the problem
   happens again.

## Important Notes

- `/api/v1/rule` is the only endpoint here which is **not** wrapped in `{"status":...,"data":...}`; its fields sit at the top level
- `/api/v1/rule` and `/api/v1/alert` both require `group_id` **and** the rule or alert id, taken from `/api/v1/rules`
- `/-/reload` is the only write, and may be protected by `-reloadAuthKey`
- `type: graphite` rules take a Graphite render query as `expr`. Workflow B applies unchanged
