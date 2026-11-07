# 🧮 Risk Scoring Model & Severity Thresholds

## 📌 Overview

This document explains the custom weighted risk-scoring model used in the **Brute-Force Attack Enrichment Using n8n** project. The workflow combines information from a Wazuh SSH authentication alert with AbuseIPDB IP reputation data to produce a transparent, reproducible **0–100 risk score** and a corresponding severity label.

The score is a **triage heuristic**, not a probability of compromise or proof that an IP address is malicious. Analysts should validate the results against authentication logs, asset context, and other telemetry before taking action.

## ⚖️ Scoring Weights

Each condition contributes its assigned points when true. Otherwise, it contributes zero.

| Factor | Condition | Points | Rationale |
|---|---|---:|---|
| AbuseIPDB reputation | `abuse_score >= 80` | **40** | Strong negative community reputation is the largest single signal in this model. |
| Tor exit node | `is_tor === true` | **25** | An anonymizing exit node can complicate source attribution; Tor use alone is not malicious. |
| Authentication failures | `failure_count >= 50` | **20** | A high number of failures may indicate automated guessing, provided the count covers a meaningful time window. |
| Targeted username | `username === "root"` | **10** | Attempts against a privileged account merit additional attention. |
| Abuse report volume | `total_reports >= 100` | **5** | Numerous reports add corroborating context, but may overlap with the reputation score. |
| **Maximum possible score** | All five conditions met | **100** | |

### Calculation

```text
risk_score =
    (abuse_score >= 80       ? 40 : 0)
  + (is_tor === true         ? 25 : 0)
  + (failure_count >= 50     ? 20 : 0)
  + (username === "root"     ? 10 : 0)
  + (total_reports >= 100    ?  5 : 0)
```

All five checks are binary: a value just above a threshold earns the same points as a much larger value. For example, 50 and 500 failed attempts both contribute 20 points. This makes the logic easy to audit but sacrifices some nuance.

## 🚦 Severity Thresholds

| Risk score | Severity | Workflow response |
|---:|---|---|
| **90–100** | **CRITICAL** | ntfy notification + Trello investigation ticket + HTML report |
| **70–89** | **HIGH** | ntfy notification + Trello investigation ticket + HTML report |
| **40–69** | **MEDIUM** | ntfy notification only |
| **0–39** | **LOW** | ntfy notification only |

**Ticket creation starts at 70 points, not 75.** The `Route by Severity` IF node checks whether the severity is `HIGH` or `CRITICAL`. In the current workflow, the HTML report is downstream of Trello ticket creation, so MEDIUM and LOW alerts do not generate that report.

The workflow uses these ntfy priorities: CRITICAL → `urgent`, HIGH → `high`, MEDIUM → `default`, LOW → `low`.

## 🧪 Worked Examples

These are **hypothetical enrichment values**, not recorded AbuseIPDB results. Actual scores depend on the API response at execution time.

| Scenario | Abuse score | Tor | Failures | Username | Reports | Calculation | Score | Severity | Trello? |
|---|---:|---|---:|---|---:|---|---:|---|---|
| Low-volume login failures | 0 | No | 3 | `ubuntu` | 0 | 0 | **0** | LOW | No |
| Repeated root login attempts | 20 | No | 87 | `root` | 10 | 20 + 10 | **30** | LOW | No |
| Poor reputation, repeated root attempts | 90 | No | 60 | `root` | 25 | 40 + 20 + 10 | **70** | HIGH | Yes |
| Tor exit node, repeated root attempts | 20 | Yes | 87 | `root` | 150 | 25 + 20 + 10 + 5 | **60** | MEDIUM | No |
| All five indicators present | 90 | Yes | 87 | `root` | 150 | 40 + 25 + 20 + 10 + 5 | **100** | CRITICAL | Yes |

### How the sample Wazuh alerts affect scoring

- **Production-server scenario:** `185.220.101.47`, 87 failed attempts, `root` → **30 base points** before enrichment.
- **Low-volume form scenario:** `8.8.8.8`, 3 failed attempts, `ubuntu` → **0 base points** before enrichment.
- **Staging-server scenario:** `185.220.101.47`, 60 failed attempts, `root` → **30 base points** before enrichment.

The first and third scenarios have the same base score. Their final severity depends on the AbuseIPDB data returned for the source IP. Neither should be labeled HIGH or CRITICAL solely from the provided Wazuh fields.

## 🧠 Why These Weights?

**AbuseIPDB reputation (40%)** receives the greatest weight because it brings external context to an otherwise local authentication event. However, crowdsourced reports can be incomplete, outdated, or biased toward frequently scanned infrastructure.

**Tor status (25%)** is a contextual signal. It can make attribution harder, but Tor also has legitimate privacy uses; treating it as inherently malicious would generate false positives.

**Failure count (20%)** reflects repeated authentication activity. The count should be associated with a clear observation period: 50 failures in one minute and 50 failures over several months are not equivalent.

**Root username (10%)** adds priority for privileged account targeting. This simple check does not cover other administrative or service accounts.

**Report volume (5%)** adds a small amount of corroboration. It may double-count information already represented in `abuseConfidenceScore`, so its low weight is deliberate.

These weights are **design choices for a lab**, not empirically validated industry standards.

## 🔍 Explainability: The `reasons` Array

The scoring node records each triggered condition in a `reasons` array. This allows an analyst to see **why** a score was assigned rather than relying only on the final number.

Example:

```json
{
  "risk_score": 70,
  "severity": "HIGH",
  "reasons": [
    "High AbuseIPDB confidence score",
    "High authentication failure count",
    "Privileged account targeted"
  ]
}
```

This example corresponds to a source with `abuse_score >= 80`, `failure_count >= 50`, `username = root`, no Tor flag, and fewer than 100 abuse reports.

## ⚠️ Limitations and Edge Cases

1. **Missing enrichment data:** A failed AbuseIPDB request or missing field must not silently be treated as a trustworthy zero-risk result. Flag the lookup as incomplete and route for review.
2. **Changing reputation:** AbuseIPDB values may change between runs. Record the lookup timestamp and reporting window (`maxAgeInDays = 90`) when comparing results.
3. **Whitelisting:** The five-factor model does **not** currently use `isWhitelisted`. A whitelisted IP is not automatically LOW severity.
4. **Shared infrastructure:** VPNs, proxies, NAT gateways, and cloud services may serve many users; IP reputation alone cannot identify an individual actor.
5. **Threshold jumps:** Scores change abruptly at thresholds. For example, 49 failures earn 0 points while 50 earn 20.
6. **Unmodeled context:** The score does not include successful authentication after failures, asset criticality, user history, geolocation anomalies, or Wazuh rule level.
7. **Duplicate incidents:** Multiple related alerts may create multiple tickets unless deduplication is added.
8. **No automatic containment:** HIGH and CRITICAL classifications create notifications and tickets; they do not justify blocking an IP or disabling an account without investigation.

## 🛠️ Suggested Improvements

- Validate all input fields and clearly flag unavailable threat intelligence.
- Define a fixed time window for `failure_count` and correlate repeated attempts across hosts.
- Add account privilege classification beyond a literal `root` check.
- Include Wazuh rule level, asset criticality, and evidence of successful authentication.
- Add alert deduplication, correlation, and investigation status tracking.
- Test thresholds against labeled historical alerts and measure false-positive/false-negative rates before production use.
- Record the score, contributing reasons, enrichment timestamp, and scoring-model version for auditability.
- Generate the HTML dashboard independently of Trello so all severities can be reviewed.

## 📊 Suggested Validation Table

Fill in this table using **actual n8n execution output** rather than assumptions:

| Test | Abuse score | Tor | Reports | Base points | Final score | Severity | Routing result |
|---|---:|---|---:|---:|---:|---:|---|
| Production Wazuh alert | TBD | TBD | TBD | 30 | TBD | TBD | TBD |
| Low-volume form alert | TBD | TBD | TBD | 0 | TBD | TBD | TBD |
| Staging form alert | TBD | TBD | TBD | 30 | TBD | TBD | TBD |

Screenshots worth including: `Calculate Risk Score` output (including `reasons`), `Route by Severity` branch, ntfy notification, and Trello card for a HIGH/CRITICAL test.

## 📚 References

- [AbuseIPDB API documentation](https://docs.abuseipdb.com/)
- [Wazuh documentation](https://documentation.wazuh.com/)
- [n8n documentation](https://docs.n8n.io/)
- [MITRE ATT&CK T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/)

---

**Summary:** This scoring model makes initial SSH brute-force triage consistent and explainable. Its value is in prioritizing investigation and reducing repetitive work—not in replacing evidence-based analyst decisions.
