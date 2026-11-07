# 🧪 Testing Results — Brute-Force Attack Enrichment Using n8n

## 📌 Overview

This document records three executions of an n8n security automation workflow that processes simulated Wazuh SSH authentication alerts, enriches source IP addresses through AbuseIPDB, calculates a weighted risk score, and routes incidents to ntfy and, for HIGH/CRITICAL alerts, Trello.

**Scope:** These were lab tests using sample or manually submitted Wazuh-format alerts. The results validate the observed automation behavior; they do not establish that the target systems were compromised or that a live attack occurred.

## 🎯 Validation Objectives

- Confirm that Wazuh-format alert fields are extracted and normalized.
- Confirm AbuseIPDB enrichment is incorporated into the risk score.
- Verify severity classifications against the configured thresholds.
- Verify HIGH/CRITICAL alerts qualify for Trello ticket creation and LOW/MEDIUM alerts do not.
- Verify notification delivery and dashboard rendering where observed.

## 🧮 Risk-Scoring Methodology

The workflow uses a custom additive model with a maximum score of 100.

| Indicator | Trigger condition | Points |
|---|---|---:|
| AbuseIPDB reputation | `abuse_score >= 80` | +40 |
| Tor exit node | `is_tor === true` | +25 |
| Failed SSH authentications | `failure_count >= 50` | +20 |
| Targeted username | `username === "root"` | +10 |
| Abuse report volume | `total_reports >= 100` | +5 |
| **Maximum** | | **100** |

| Risk score | Severity | Expected routing |
|---|---|---|
| 0–39 | LOW | ntfy only |
| 40–69 | MEDIUM | ntfy only |
| 70–89 | HIGH | ntfy and Trello |
| 90–100 | CRITICAL | ntfy and Trello |

**Important:** The minimum score that triggers Trello ticket creation is **70**, not 75. These are lab-defined thresholds, not official Wazuh or AbuseIPDB categories. See [Risk Scoring Methodology](risk-scoring.md) for further explanation.

---

## 🔴 Test 1 — Wazuh SSH Brute-Force Alert

### Objective

Verify enrichment, risk scoring, and high-priority escalation for a sample SSH brute-force alert targeting a privileged account.

### Input Alert

| Field | Value |
|---|---|
| Entry path | Wazuh Alert Intake (sample/webhook path) |
| Source IP | `185.220.101.47` |
| Target host | `prod-webserver-01` |
| Target username | `root` |
| Failed authentication attempts | 87 |
| Service | SSH |
| MITRE ATT&CK | T1110 — Brute Force |

### Actual Execution Results

| Metric | Observed value |
|---|---|
| AbuseIPDB confidence score | **100** |
| Total abuse reports | **298** |
| Tor exit node | **True** |
| Final risk score | **100/100** |
| Severity | **CRITICAL** |
| Trello ticket created | **Yes** |
| ntfy notification sent | **Yes** |

### Scoring Breakdown

| Triggered factor | Evidence | Points |
|---|---|---:|
| High AbuseIPDB confidence | 100 ≥ 80 | +40 |
| Tor exit node | True | +25 |
| High failure count | 87 ≥ 50 | +20 |
| Privileged username | `root` | +10 |
| High report volume | 298 ≥ 100 | +5 |
| **Total** | | **100** |

### Findings

The alert met all five configured scoring conditions. The original SSH event supplied 30 points (failure volume and `root` targeting), while the AbuseIPDB enrichment supplied 70 additional points. The resulting **CRITICAL** classification was consistent with the scoring rules.

The AbuseIPDB response showed a confidence score of 100 and 298 reports. These indicators support further investigation, but neither the reputation score nor Tor exit-node classification independently proves malicious intent. The 87 failed logins are consistent with brute-force behavior; the available evidence does **not** show a successful login or confirmed compromise.

The observed Trello ticket and ntfy notification confirm that the high-priority escalation actions executed in this test.

### Test Outcome

**PASS — CRITICAL scoring, ticket creation, and ntfy notification observed.** Dashboard rendering was not separately reported for this test.

**Evidence to attach:**

- `../screenshots/test1-abuseipdb.png` — AbuseIPDB score, reports, and Tor status
- `../screenshots/test1-risk-score.png` — Risk score, severity, and contributing reasons
- `../screenshots/test1-trello-ticket.png` — Created Trello card
- `../screenshots/test1-ntfy.png` — Delivered notification

*Screenshot paths above are suggested filenames; add the corresponding screenshots before using them as links.*

---

## 🟢 Test 2 — Low-Severity SSH Authentication Alert

### Objective

Validate LOW classification and notification-only routing for a low-volume SSH authentication event, even when AbuseIPDB reports a nonzero community report count.

### Input Alert

| Field | Value |
|---|---|
| Entry path | Brute-Force Alert Simulator (form) |
| Source IP | `8.8.8.8` |
| Target host | `dev-server-03` |
| Target username | `ubuntu` |
| Failed authentication attempts | 3 |
| Wazuh rule level | 5 |
| Service | SSH |
| MITRE ATT&CK label in sample | T1110 — Brute Force |

### Actual Execution Results

| Metric | Observed value |
|---|---|
| AbuseIPDB confidence score | **0** |
| Total abuse reports | **196** |
| Final risk score | **5/100** |
| Severity | **LOW** |
| IF routing branch | **False** |
| Trello ticket created | **No** |
| ntfy notification sent | **Yes** |

### Scoring Breakdown

| Factor | Observed condition | Points |
|---|---|---:|
| AbuseIPDB confidence | 0 < 80 | +0 |
| Tor exit node | No points contributed, inferred from total | +0 |
| Failure count | 3 < 50 | +0 |
| Username | `ubuntu` ≠ `root` | +0 |
| Report volume | 196 ≥ 100 | +5 |
| **Total** | | **5** |

**Note:** Tor status was not separately supplied in the Test 2 results. Its zero-point contribution is inferred from the reported final score and the other known factors; it should not be presented as a directly observed `is_tor` value.

### Findings

AbuseIPDB returned **196 reports but a confidence score of 0**. This is a useful example of why report count and reputation score should be evaluated separately. The scoring model adds five points for report volume even when the confidence-score threshold is not met. This is the model's configured behavior, not necessarily a software defect.

The three authentication failures were below the 50-failure threshold, and the username `ubuntu` did not match the model's literal `root` condition. The resulting **LOW (5/100)** classification correctly selected the **False** branch. The notification was sent and no Trello ticket was created.

The low score does not prove the activity was benign. In particular, `ubuntu` may be privileged on some systems, and the model does not evaluate host context or successful authentications.

### Test Outcome

**PASS — LOW scoring, False-branch routing, ntfy notification, and Trello suppression observed.** The dashboard was not expected to run on this branch in the documented workflow design.

**Evidence to attach:**

- `../screenshots/test2-abuseipdb.png`
- `../screenshots/test2-risk-score.png`
- `../screenshots/test2-routing.png`
- `../screenshots/test2-ntfy.png`

---

## 🔴 Test 3 — Simulated High-Risk SSH Brute-Force Alert

### Objective

Test high-priority escalation and dashboard rendering using a manually submitted Wazuh-format alert. The scenario was intended to exercise HIGH routing, but live enrichment produced a CRITICAL result.

### Input Alert

| Field | Value |
|---|---|
| Entry path | Brute-Force Alert Simulator (form) |
| Source IP | `185.220.101.47` |
| Target host | `staging-api-01` |
| Target username | `root` |
| Failed authentication attempts | 60 |
| Wazuh rule level | 10 |
| Service | SSH |
| MITRE ATT&CK | T1110 — Brute Force |

### Actual Execution Results

| Metric | Observed value |
|---|---|
| AbuseIPDB confidence score | **100** |
| Tor exit node | **True** |
| Total abuse reports | **298** |
| Final risk score | **100/100** |
| Severity | **CRITICAL** |
| Trello ticket created | **Yes** |
| HTML dashboard rendered | **Yes** |

### Scoring Breakdown

| Triggered factor | Evidence | Points |
|---|---|---:|
| High AbuseIPDB confidence | 100 ≥ 80 | +40 |
| Tor exit node | True | +25 |
| High failure count | 60 ≥ 50 | +20 |
| Privileged username | `root` | +10 |
| High report volume | 298 ≥ 100 | +5 |
| **Total** | | **100** |

### Findings

The simulated alert contributed a base score of 30 points from 60 failed SSH authentications and targeting `root`. AbuseIPDB enrichment contributed the remaining 70 points. The resulting **CRITICAL (100/100)** classification correctly satisfied the high-priority routing condition.

The Trello ticket was created and the HTML dashboard rendered, validating the ticketing-to-dashboard portion of the pipeline. **ntfy delivery was not explicitly reported in the Test 3 results**, so it is not marked as verified for this test.

Because the outcome was CRITICAL, this test **did not independently validate HIGH (70–89) severity classification**. It did validate that the shared HIGH/CRITICAL routing condition accepted a CRITICAL event.

### Test Outcome

**PASS — CRITICAL scoring, Trello ticket creation, and HTML dashboard rendering observed.** HIGH-only threshold behavior and Test 3 ntfy delivery remain unverified.

**Evidence to attach:**

- `../screenshots/test3-abuseipdb.png`
- `../screenshots/test3-risk-score.png`
- `../screenshots/test3-routing.png`
- `../screenshots/test3-trello-ticket.png`
- `../screenshots/test3-dashboard.png`

---

## 📊 Cross-Test Comparison

| Metric | Test 1 | Test 2 | Test 3 |
|---|---|---|---|
| Entry path | Wazuh sample | Form simulator | Form simulator |
| Source IP | `185.220.101.47` | `8.8.8.8` | `185.220.101.47` |
| Failed attempts | 87 | 3 | 60 |
| AbuseIPDB score | 100 | 0 | 100 |
| Total reports | 298 | 196 | 298 |
| Tor status | True | Not directly reported | True |
| Risk score | **100** | **5** | **100** |
| Severity | CRITICAL | LOW | CRITICAL |
| Trello ticket | Yes | No | Yes |
| ntfy notification | Yes | Yes | Not reported |
| HTML dashboard | Not reported | Not expected on LOW branch | Yes |
| Outcome | PASS (observed checks) | PASS (observed checks) | PASS (observed checks) |

### What the Tests Demonstrate

1. **Consistent scoring:** Each reported final score matches the configured weights and known alert/enrichment indicators.
2. **Conditional escalation:** CRITICAL events created Trello tickets, while the LOW event did not.
3. **Notification coverage:** ntfy delivery was observed in Tests 1 and 2.
4. **Report generation:** The HTML incident dashboard rendered in Test 3.
5. **Value of enrichment:** Test 3's authentication-based score of 30 rose to 100 when external reputation factors were incorporated.

## ⚠️ Coverage Gaps and Limitations

- **HIGH and MEDIUM thresholds were not directly tested.** Current tests exercise only LOW and CRITICAL classifications.
- **No successful compromise was demonstrated.** The simulated alerts represent failed authentication attempts.
- **AbuseIPDB data is time-dependent.** Re-running the same IP lookups may produce different scores and report counts.
- **The model does not use Wazuh rule level, asset criticality, report recency, successful logins, or most privileged usernames other than `root`.**
- **No measured execution duration was provided.** Avoid claiming a quantified time saving until actual runtimes are recorded.
- **No error-path tests were reported.** API failures, missing data, rate limits, and Trello/ntfy outages remain to be validated.

## 🔬 Recommended Additional Tests

Use controlled mock enrichment responses so each boundary is reproducible and does not depend on changing public reputation data.

| Target score | Expected severity | Expected Trello behavior |
|---:|---|---|
| 39 | LOW | No ticket |
| 40 | MEDIUM | No ticket |
| 69 | MEDIUM | No ticket |
| 70 | HIGH | Create ticket |
| 89 | HIGH | Create ticket |
| 90 | CRITICAL | Create ticket |

**Scoring-model caveat:** Because the current model assigns only five fixed point increments (40, 25, 20, 10, and 5), not every integer score above is attainable. For boundary testing, inject a controlled `risk_score` into the severity-classification/routing stage or temporarily isolate the classification function. Separately test attainable full-model scores such as 40, 65, 70, 85, 90, and 100.

Additional recommended checks include malformed JSON, invalid IP input, unavailable AbuseIPDB service, missing enrichment fields, failed Trello creation, and duplicate alerts.

## 🛡️ Suggested Analyst Follow-Up

For CRITICAL events, review SSH logs for successful logins near the failed attempts, correlate source IP activity across hosts, inspect account activity, verify threat intelligence through independent sources, and record conclusions in the investigation ticket. For LOW events, review relevant context and monitor for repeated activity rather than treating a low score as proof of safety.

## 📸 Evidence Organization

Store redacted execution screenshots in `screenshots/` using the filenames suggested in each test section. Screenshots should show node output, scoring reasons, routing paths, and successful integrations without exposing API credentials, internal secrets, or sensitive incident data.

## 🏁 Conclusion

Across three lab executions, the n8n workflow produced the reported weighted risk scores, correctly routed observed LOW and CRITICAL cases, created Trello tickets for both CRITICAL cases, delivered ntfy notifications in two verified tests, and rendered an incident dashboard in one verified test.

These results support the workflow's effectiveness as a **demonstration of automated SOC alert enrichment and triage**. Further boundary, failure-handling, and live-integration testing would be required before describing it as production-ready.
