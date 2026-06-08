# Rollback Runbook — Vision Moderation Service

**Last updated:** see git blame | **Owner:** ml-platform on-call

---

## When to roll back

Trigger rollback immediately if **any** of these are true for > 2 min:

- [ ] Error rate > 1% (AvailabilityBurnFast firing)
- [ ] p99 latency > 1 s (LatencyP99High firing)
- [ ] ModelVersionMismatch alert firing
- [ ] Precision or recall proxy (human-review overturn rate) drops > 5 pp vs baseline dashboard

Do **not** wait for all windows to confirm — one clear signal is enough.

---

## How to roll back

**Option A — workflow_dispatch (preferred)**

1. Go to **Actions → Deploy Vision Moderation Model → Run workflow**
2. Set `IMAGE_TAG` to the last known-good SHA (find it in the production deployment history)
3. Click **Run workflow** — it will skip canary verification and go straight to full rollout

**Option B — CLI**

```bash
# Get the previous good tag from the registry
GOOD_TAG=$(./scripts/deploy.sh production --last-stable-tag)

# Roll back
./scripts/rollback.sh production "$GOOD_TAG"
```

Both paths call the same `rollback.sh` black-box; Option A adds an audit trail in Actions.

---

## What to verify

Wait 5 minutes after rollback completes, then confirm all green:

- [ ] **Grafana → Vision Moderation SLO** dashboard: error rate < 0.5%, p99 < 1 s
- [ ] **Alertmanager**: AvailabilityBurnFast, LatencyP99High, ModelVersionMismatch all resolved
- [ ] `kubectl get pods -n vision-moderation` — all pods Running, `model_version` label matches target
- [ ] One manual test request returns `200` with expected `X-Model-Version` header

---

## Who to notify

| When | Channel | Who |
|------|---------|-----|
| Rollback initiated | `#incidents` Slack | Post with `@ml-platform` |
| Rollback complete | `#incidents` Slack | Update thread, confirm status |
| Error budget > 5% consumed | PagerDuty escalation | Eng Manager + SRE lead |

---

## What NOT to do

- **Do not roll forward** before identifying root cause — you may re-introduce the same failure
- **Do not skip the smoke tests** on the stable version to "save time"
- **Do not silence alerts** during rollback — you need them to confirm recovery
- **Do not roll back the database** — model inference is stateless; DB schema changes need a separate incident process

---

## When to roll forward

Only re-deploy the new version when **all** of the following are true:

1. Root cause is identified and documented in the incident ticket
2. Fix is merged and the build has passed lint, test, scan
3. Staging smoke tests pass with the fix applied
4. At least one approver from ml-platform signs off in the PR

---

*If in doubt: keep the stable version running and escalate. Downtime is cheaper than a bad model in production.*
