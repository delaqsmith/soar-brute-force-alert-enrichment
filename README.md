# 🛡️ Brute-Force Attack Enrichment & Automated Incident Response Using n8n

## 📌 Project Overview

This project demonstrates how Security Orchestration, Automation, and Response (SOAR) principles can be used to streamline the investigation and prioritization of SSH brute-force attacks.

I developed an automated security workflow using **n8n, Wazuh, AbuseIPDB, Trello, and ntfy** to enrich security alerts with threat intelligence, calculate risk scores, classify incident severity, and automatically generate notifications and investigation tickets.

In a traditional SOC environment, an analyst investigating repeated SSH authentication failures may need to manually review the alert, research the source IP address, evaluate threat intelligence, determine the appropriate priority, and document the incident.

This workflow automates those repetitive enrichment and administrative tasks, allowing analysts to begin their investigations with a structured, risk-labeled incident summary.

The automation supports both simulated Wazuh alerts and webhook-based alert intake, making it suitable for testing different security scenarios without requiring a continuously active Wazuh environment.

### 🎯 Project Objectives

- Automate the enrichment of SSH brute-force alerts.
- Integrate Wazuh alert data with AbuseIPDB threat intelligence.
- Normalize security event data from multiple input sources.
- Develop a weighted risk-scoring model using multiple security indicators.
- Classify incidents into CRITICAL, HIGH, MEDIUM, and LOW severity levels.
- Route incidents automatically based on their calculated severity.
- Generate Trello investigation tickets for high-priority incidents.
- Send real-time notifications using ntfy.
- Present enriched incident information through an HTML dashboard.
- Demonstrate how SOAR workflows improve the consistency of SOC alert triage.

---

## 🛠️ Technologies & Tools

| Technology | Purpose |
|------------|---------|
| n8n | SOAR workflow orchestration and automation |
| Wazuh | Security monitoring and alert generation |
| AbuseIPDB | IP reputation and threat intelligence enrichment |
| REST APIs | Communication with external services |
| JavaScript | Alert normalization, enrichment, scoring, and formatting |
| JSON | Structured security alert and API response data |
| Trello | Automated incident tracking and ticket management |
| ntfy | Real-time incident notifications |
| HTML | Incident dashboard and report visualization |
| MITRE ATT&CK | Classification of brute-force attack behavior |

### MITRE ATT&CK Mapping

**Technique:** [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/)

**Tactic:** Credential Access

The simulated Wazuh alerts represent repeated SSH authentication attempts consistent with brute-force activity.

MITRE ATT&CK T1110 provides a standardized classification for this behavior.

---

## ✨ Key Features

### 1. Dual Alert Intake Methods

The workflow supports two entry points:

**Wazuh Webhook Intake**

Receives structured security alerts through an HTTP POST request.

**Brute-Force Alert Simulator**

Accepts manually submitted Wazuh-format JSON alerts through an n8n form.

The simulator allows testing of different source IP addresses, usernames, authentication failure counts, and target systems.

### 2. Automated Threat Intelligence Enrichment

The workflow queries AbuseIPDB to retrieve:

- Abuse confidence score
- ISP and domain information
- Country code
- Total abuse reports
- Most recent abuse report
- Tor exit node status
- Associated hostnames

These indicators provide additional context for determining whether an alert warrants further investigation.

### 3. Weighted Risk Scoring

A custom scoring model evaluates five factors:

- AbuseIPDB confidence score
- Tor exit node status
- Authentication failure count
- Targeted username
- Total abuse reports

Each factor contributes a defined number of points to the final risk score.

### 4. Automated Severity Classification

The calculated risk score is translated into four severity levels:

| Risk Score | Severity |
|------------|----------|
| 90–100 | CRITICAL |
| 70–89 | HIGH |
| 40–69 | MEDIUM |
| 0–39 | LOW |

These thresholds are custom to this project and are not official Wazuh or AbuseIPDB severity classifications.

### 5. Severity-Based Incident Routing

The workflow automatically determines the next action:

| Severity | ntfy Notification | Trello Ticket |
|----------|-------------------|---------------|
| CRITICAL | Yes | Yes |
| HIGH | Yes | Yes |
| MEDIUM | Yes | No |
| LOW | Yes | No |

This routing strategy keeps the Trello investigation board focused on higher-priority alerts while preserving visibility into lower-severity events.

### 6. Automated Incident Documentation

High-priority alerts generate Trello cards containing a structured incident summary.

Each ticket includes the incident identifier, severity, source IP, targeted host, authentication details, threat intelligence, and risk assessment.

### 7. Real-Time Incident Notifications

The workflow sends incident notifications through ntfy, with notification priority determined by the calculated severity.

### 8. HTML Incident Dashboard

For HIGH and CRITICAL alerts, the workflow generates an HTML dashboard containing the enriched incident information and a direct link to the Trello investigation ticket.

---

## 🧩 Workflow Architecture

The workflow contains 11 nodes organized into five stages:

1. Alert intake
2. Data normalization
3. Threat intelligence enrichment
4. Risk scoring and severity classification
5. Incident notification and ticketing

### Workflow Diagram

```text
 Wazuh Alert Intake Brute-Force Alert Simulator
    (Webhook) (Form)
        │ │
        └──────────────┬───────────────┘
                       │
                       ▼
              Extract Alert Fields
                       │
                       ▼
                AbuseIPDB Lookup
                       │
                       ▼
              Merge AbuseIPDB Data
                       │
                       ▼
              Calculate Risk Score
                       │
                       ▼
              Build Incident Summary
                       │
                       ▼
                Route by Severity
                       │
             ┌─────────┴─────────┐
             │ │
             ▼ ▼
        HIGH/CRITICAL MEDIUM/LOW
             │ │
       ┌─────┴─────┐ │
       ▼ ▼ ▼
    Trello ntfy ntfy
    Ticket Alert Alert
       │
       ▼
  HTML Incident
    Dashboard
```

<img width="1142" height="641" alt="image" src="https://github.com/user-attachments/assets/e2dda37c-dc37-4ec7-b833-6f286de2f85b" />


---

## ⚙️ Workflow Implementation

### Step 1: Configure Wazuh Alert Intake

**Node:** Wazuh Alert Intake

**Type:** Webhook Trigger

The first entry point receives Wazuh security alerts through an HTTP POST request.

**Webhook Path:**

```text
/wazuh-alerts
```

The sample alert represents an SSH brute-force event involving repeated authentication failures.

#### Example Alert Details

| Field | Value |
|-------|-------|
| Source IP | `185.220.101.47` |
| Target Host | `prod-webserver-01` |
| Target Username | `root` |
| Failed Attempts | 87 |
| Service | SSH |
| Decoder | sshd |
| MITRE Technique | T1110 — Brute Force |

The webhook passes the JSON payload downstream for processing.

<img width="1741" height="320" alt="image" src="https://github.com/user-attachments/assets/5fd9c1ce-6ac4-492b-ad63-edaafb17655d" />


### Step 2: Configure the Brute-Force Alert Simulator

**Node:** Brute-Force Alert Simulator

**Type:** Form Trigger

The second entry point provides an alternative way to execute the workflow.

Rather than waiting for a live security event, I can submit Wazuh-format JSON directly through an n8n form.

This enables controlled testing of different attack scenarios.

The form contains a textarea labeled:

```text
Alert JSON
```

The submitted payload is then passed to the same normalization and enrichment pipeline used by the webhook.


### Step 3: Extract and Normalize Alert Fields

**Node:** Extract Alert Fields

**Type:** Code

This node processes alerts received through either entry point.

It determines whether the input originated from the form or webhook and parses the corresponding JSON payload.

The node extracts and normalizes the following fields:

| Normalized Field | Description |
|------------------|-------------|
| `src_ip` | Source IP address |
| `username` | Targeted account |
| `failure_count` | Number of failed authentication attempts |
| `target_host` | Targeted system |
| `service` | Affected service |
| `timestamp` | Event timestamp |
| `alert_id` | Unique incident identifier |

#### Incident ID Generation

Each alert receives a unique identifier beginning with:

```text
INC-HX-
```

The identifier is generated using the current epoch timestamp in milliseconds.

This allows the workflow to associate enrichment results, notifications, and Trello tickets with a specific incident.

Once normalization is complete, downstream nodes can process alerts without needing to know which intake method was used.


### Step 4: Query AbuseIPDB Threat Intelligence

**Node:** AbuseIPDB Lookup

**Type:** HTTP Request

The workflow enriches the source IP address by querying AbuseIPDB.

**API Endpoint:**

```http
GET https://api.abuseipdb.com/api/v2/check
```

#### Query Parameters

| Parameter | Value |
|-----------|-------|
| `ipAddress` | Dynamic `src_ip` value |
| `maxAgeInDays` | `90` |

The `ipAddress` parameter is populated using the normalized alert data.

The 90-day reporting window provides a configurable period for reviewing reported abuse activity.

This setting balances the need for recent threat intelligence with a broader historical perspective.

#### Authentication

Authentication is handled using a configured n8n credential.

The AbuseIPDB API key is not embedded directly into the workflow logic.

#### Extracted Intelligence

The API response provides the following enrichment fields:

```text
abuse_score
isp
domain
total_reports
last_reported
is_tor
country_code
hostnames
```

### Step 5: Merge Alert and Threat Intelligence Data

**Node:** Merge AbuseIPDB Data

**Type:** Code

The original Wazuh alert fields and AbuseIPDB reputation indicators are combined into one unified object.

This allows the scoring and reporting nodes to reference both the original security event and its associated threat intelligence.

#### Data Categories

**Original Alert Data**

- Source IP
- Username
- Failed login attempts
- Target host
- Service
- Timestamp
- Incident ID

**Enrichment Data**

- Abuse confidence score
- ISP
- Domain
- Total reports
- Tor status
- Country code
- Hostnames
- Most recent abuse report

Centralizing these fields simplifies the remaining workflow logic.


### Step 6: Calculate the Weighted Risk Score

**Node:** Calculate Risk Score

**Type:** Code

The workflow uses a custom additive scoring model to assess the priority of each alert.

Five factors contribute to the total score.

#### Weighted Risk Scoring Model

| Factor | Condition | Points |
|--------|-----------|--------|
| AbuseIPDB Reputation | `abuse_score >= 80` | +40 |
| Tor Exit Node | `is_tor === true` | +25 |
| Authentication Failures | `failure_count >= 50` | +20 |
| Targeted Username | `username === "root"` | +10 |
| Abuse Report Volume | `total_reports >= 100` | +5 |
| **Maximum Possible Score** | | **100** |

#### Risk Scoring Logic

The following JavaScript illustrates the scoring model:

```javascript
let risk_score = 0;
const reasons = [];

if (abuse_score >= 80) {
  risk_score += 40;
  reasons.push("High AbuseIPDB confidence score");
}

if (is_tor === true) {
  risk_score += 25;
  reasons.push("Source identified as Tor exit node");
}

if (failure_count >= 50) {
  risk_score += 20;
  reasons.push("High authentication failure count");
}

if (username === "root") {
  risk_score += 10;
  reasons.push("Privileged account targeted");
}

if (total_reports >= 100) {
  risk_score += 5;
  reasons.push("High community abuse report volume");
}

let severity;

if (risk_score >= 90) {
  severity = "CRITICAL";
} else if (risk_score >= 70) {
  severity = "HIGH";
} else if (risk_score >= 40) {
  severity = "MEDIUM";
} else {
  severity = "LOW";
}
```

The model returns three important values:

- `risk_score`: Total weighted score
- `severity`: Calculated severity classification
- `reasons`: Array identifying the contributing risk factors

The `reasons` array provides transparency into how the final score was calculated.

#### Example Risk Calculation

Consider an alert with the following indicators:

| Indicator | Value | Points |
|-----------|-------|--------|
| AbuseIPDB Score | 90 | +40 |
| Tor Exit Node | True | +25 |
| Failed Attempts | 87 | +20 |
| Targeted Username | root | +10 |
| Total Reports | 150 | +5 |
| **Total** | | **100** |

**Final Severity: CRITICAL**

*Note: This example uses hypothetical enrichment values to demonstrate the scoring calculation. It is not a recorded AbuseIPDB response.*

The scoring model is intended to support initial triage rather than replace analyst judgment.

### Step 7: Build the Incident Summary

**Node:** Build Incident Summary

**Type:** Code

This node formats the enriched alert data into a structured plain-text incident report.

The summary includes:

- Incident ID
- Severity classification
- Source IP address
- Targeted host
- Targeted username
- Failed authentication count
- Risk score
- AbuseIPDB confidence score
- Tor exit node status
- ISP and country information
- Contributing risk factors

The formatted summary is reused by the notification and reporting components.

This centralized approach ensures that multiple outputs present consistent incident information.

#### Example Incident Summary

```text
SSH BRUTE-FORCE SECURITY ALERT

Incident ID: INC-HX-EXAMPLE
Severity: CRITICAL

Source IP: 185.220.101.47
Target Host: prod-webserver-01
Target Username: root
Service: SSH
Failed Attempts: 87

Risk Score: 100/100
AbuseIPDB Score: 90/100
Tor Exit Node: Yes
Total Abuse Reports: 150

Risk Factors:
- High AbuseIPDB confidence score
- Source identified as Tor exit node
- High authentication failure count
- Privileged account targeted
- High community abuse report volume

Recommended Action:
Prioritize analyst investigation and
correlate the alert with authentication
and network security logs.
```

*Note: The enrichment values and calculated score above are illustrative, not verified lab results.*

### Step 8: Route Alerts by Severity

**Node:** Route by Severity

**Type:** IF

The workflow evaluates the severity assigned by the risk-scoring node.

#### Routing Condition

```javascript
severity === "CRITICAL" ||
severity === "HIGH"
```

#### TRUE Branch

Alerts classified as CRITICAL or HIGH trigger:

1. Trello ticket creation
2. ntfy notification

#### FALSE Branch

Alerts classified as MEDIUM or LOW trigger:

1. ntfy notification only

This routing approach reduces unnecessary ticket creation while maintaining notification coverage.

**Minimum score required for Trello ticket creation: 70**


### Step 9: Create an Investigation Ticket in Trello

**Node:** Create Trello Ticket

**Type:** Trello

For HIGH and CRITICAL alerts, the workflow automatically creates an investigation card on a configured SOC Trello board.

#### Ticket Title Format

```text
{alert_id} — [{severity}] Brute-Force Attack Detection
```

#### Ticket Description

The full incident summary generated earlier in the workflow is used as the Trello card description.

This provides the analyst with a centralized investigation record containing the most relevant alert and enrichment information.

The Trello API returns a URL for the newly created card.

That URL is subsequently used by the HTML incident dashboard.


### Step 10: Send Real-Time Notifications Using ntfy

**Node:** Send ntfy Alert

**Type:** HTTP Request

The workflow sends a POST request to an ntfy topic.

**Lab Notification Endpoint:**

```text
https://ntfy.sh/haxlab-bruteforce
```

#### Dynamic Notification Priority

| Severity | ntfy Priority |
|----------|---------------|
| CRITICAL | urgent |
| HIGH | high |
| MEDIUM | default |
| LOW | low |

The notification includes:

- Source IP address
- Target host
- Targeted username
- Authentication failure count
- Risk score
- AbuseIPDB score
- Tor exit node status

This enables analysts to review key incident details without immediately opening the n8n workflow.

**Security Note:** Public ntfy topics may be accessible to others. Production environments should use authenticated or access-controlled notification channels rather than publicly accessible topics containing sensitive incident data.


### Step 11: Generate the Brute-Force Incident Dashboard

**Node:** Brute-Force Report Portal

**Type:** HTML

The final reporting node renders a structured incident dashboard within n8n.

The dashboard displays:

| Field | Description |
|-------|-------------|
| Source IP | Origin of the suspicious activity |
| Target Host | System receiving authentication attempts |
| Username | Account targeted by the attempts |
| Service | Affected network service |
| Failure Count | Number of failed authentication attempts |
| Risk Score | Calculated weighted score |
| Severity | CRITICAL, HIGH, MEDIUM, or LOW |
| AbuseIPDB Score | Reputation confidence indicator |
| ISP | Source network provider |
| Country | Geographic network information |
| Total Reports | Community abuse reports |
| Tor Status | Whether the source is identified as a Tor exit node |

For HIGH and CRITICAL alerts, the dashboard also includes a direct link to the generated Trello ticket.

**Important:** In the current workflow configuration, the dashboard only executes for HIGH and CRITICAL alerts because it is connected downstream of the Trello ticket node.

---

## 🧪 Testing & Validation

I used three testing scenarios to evaluate the workflow's enrichment, scoring, routing, and notification functionality.

The scenarios demonstrate how different authentication failure counts, usernames, and IP reputation indicators influence the automated response.

### Test 1: Wazuh SSH Brute-Force Alert

#### Objective

Evaluate how the workflow processes an SSH brute-force alert representing repeated authentication failures against a production server.

#### Test Input

| Field | Value |
|-------|-------|
| Source IP | `185.220.101.47` |
| Target Host | `prod-webserver-01` |
| Username | `root` |
| Failed Attempts | 87 |
| Service | SSH |
| MITRE Technique | T1110 |

#### Testing Procedure

1. Execute the workflow using the Wazuh Alert Intake node.
2. Inspect the normalized alert fields.
3. Review the AbuseIPDB enrichment response.
4. Examine the calculated risk score and contributing factors.
5. Confirm the severity classification.
6. Verify which branch of the routing node executes.
7. Inspect the ntfy notification.
8. If classified as HIGH or CRITICAL, verify Trello ticket creation and dashboard generation.

#### Analysis

This scenario includes two risk factors that can be determined directly from the original Wazuh alert:

| Risk Factor | Points |
|-------------|--------|
| Failure Count ≥ 50 | +20 |
| Username = root | +10 |
| **Base Score** | **30** |

The remaining score depends on the live AbuseIPDB response.

Potential additional contributions include:

- High AbuseIPDB score: +40
- Tor exit node status: +25
- High abuse report volume: +5

Therefore, the final score can range from 30 to 100 under this model.

#### Findings to Document

Actual values returned during execution:

| Metric | Observed Result |
|--------|-----------------|
| AbuseIPDB Score | `100` |
| Total Reports | `298` |
| Tor Exit Node | `True` |
| Final Risk Score | `100` |
| Severity | `CRITICAL` |
| Trello Ticket Created | `Yes` |
| ntfy Notification Sent | `Yes` |

**📸 Screenshot Placeholders**

<img width="1908" height="864" alt="image" src="https://github.com/user-attachments/assets/ddbb4fde-6262-4738-b34c-fe7581b5d944" />

<img width="1898" height="645" alt="image" src="https://github.com/user-attachments/assets/28311c28-6c08-468d-9916-dc988bdc5092" />

<img width="1899" height="641" alt="image" src="https://github.com/user-attachments/assets/57336737-28ef-4b87-af40-bf7cd28a8f17" />


### Test 2: Low-Risk Authentication Alert

#### Objective

Evaluate how the workflow handles a low-volume authentication failure event involving a non-root username and a whitelisted IP address.

#### Test Input

| Field | Value |
|-------|-------|
| Source IP | `8.8.8.8` |
| Target Host | `dev-server-03` |
| Username | `ubuntu` |
| Failed Attempts | 3 |
| Wazuh Rule Level | 5 |
| Service | SSH |

The test uses a simulated Wazuh alert submitted through the Brute-Force Alert Simulator.

#### Expected Scoring Behavior

Based on the original alert:

| Risk Factor | Points |
|-------------|--------|
| Failure Count ≥ 50 | 0 |
| Username = root | 0 |
| **Base Score** | **0** |

The final score depends on the AbuseIPDB enrichment response.

If none of the three reputation-based thresholds are met, the final score will be 0 and the severity will be LOW.

#### Expected Routing

For a LOW classification:

- The ntfy notification should execute.
- The Trello ticket node should not execute.
- The HTML dashboard should not execute in the current configuration.

#### Findings to Document

| Metric | Observed Result |
|--------|-----------------|
| AbuseIPDB Score | `0` |
| Total Reports | `196` |
| Final Risk Score | `5` |
| Severity | `LOW` |
| Routing Branch | `False` |
| Trello Ticket Created | `No` |
| ntfy Notification Sent | `Yes` |

#### Important Observation

The test source IP `8.8.8.8` is commonly associated with Google's public DNS infrastructure.

This makes it useful for testing how the workflow handles a source IP that may have a relatively low reputation risk.

However, the scoring model does not explicitly use AbuseIPDB whitelist status.

As a result, a whitelisted IP is not automatically assigned LOW severity; its classification still depends on the five weighted scoring factors.

**📸 Screenshot Placeholders**

<img width="1885" height="602" alt="image" src="https://github.com/user-attachments/assets/9afe4642-9166-4807-8d42-4a1e59844749" />

<img width="1889" height="909" alt="image" src="https://github.com/user-attachments/assets/f30053d2-4791-4a79-9486-f68b21a66aca" />


### Test 3: High-Severity Brute-Force Alert

#### Objective

Validate that a simulated high-risk SSH brute-force alert triggers the appropriate severity-based automation.

#### Test Input

| Field | Value |
|-------|-------|
| Source IP | `185.220.101.47` |
| Target Host | `staging-api-01` |
| Username | `root` |
| Failed Attempts | 60 |
| Wazuh Rule Level | 10 |
| Service | SSH |

#### Initial Risk Factors

The original alert contributes:

| Risk Factor | Points |
|-------------|--------|
| Failure Count ≥ 50 | +20 |
| Username = root | +10 |
| **Base Score** | **30** |

Additional points are determined by the AbuseIPDB response.

#### Expected Behavior

If the combined score reaches 70 or higher:

1. The alert is classified as HIGH or CRITICAL.
2. The True routing branch executes.
3. A Trello investigation ticket is created.
4. An ntfy notification is sent.
5. The HTML incident dashboard renders.
6. The dashboard displays a link to the Trello investigation ticket.

#### Findings to Document

| Metric | Observed Result |
|--------|-----------------|
| AbuseIPDB Score | `100` |
| Tor Exit Node | `True` |
| Total Reports | `298` |
| Final Risk Score | `100` |
| Severity | `CRITICAL` |
| Trello Ticket Created | `Yes` |
| Dashboard Rendered | `Yes` |

**📸 Screenshot Placeholders**

<img width="1896" height="603" alt="image" src="https://github.com/user-attachments/assets/be8c50e5-3152-49c1-b8d5-2e77d9526827" />

<img width="1885" height="910" alt="image" src="https://github.com/user-attachments/assets/3d6f7c34-0060-43ca-a052-712c16d0c5e8" />

---

## 🔍 Findings & Security Analysis

### 1. Automated Enrichment Provides Additional Context

Wazuh identifies suspicious authentication behavior, but the original alert does not necessarily provide enough information to assess the reputation of the source infrastructure.

By integrating AbuseIPDB, the workflow adds external threat intelligence to the event.

This enables analysts to evaluate authentication activity alongside reputation indicators such as abuse confidence, Tor status, and community reports.

### 2. Multiple Indicators Support Risk Prioritization

The scoring model combines information from the security alert and threat intelligence response.

This is more informative than evaluating only the number of failed authentication attempts.

For example, an IP with numerous failed login attempts and a high abuse confidence score may warrant greater attention than an IP associated with a small number of failures and limited negative reputation data.

### 3. Privileged Account Targeting Increases Priority

The model assigns additional points when the targeted username is `root`.

This reflects the importance of identifying authentication attempts against privileged accounts.

However, the scoring model currently recognizes only the literal username `root`.

Other privileged accounts would require additional detection logic.

### 4. Severity-Based Routing Reduces Unnecessary Ticket Creation

The workflow separates notification delivery from incident ticket generation.

HIGH and CRITICAL alerts create Trello tickets, while MEDIUM and LOW alerts generate notifications only.

This helps reduce ticket volume and keeps the investigation board focused on higher-priority events.

### 5. Centralized Incident Summaries Improve Consistency

The Build Incident Summary node generates one structured report that can be reused by multiple workflow components.

This reduces duplicated formatting logic and helps maintain consistent incident details across notifications and reports.

### 6. The Risk Model Has Important Limitations

The weighted model is useful for demonstrating automated triage, but several limitations should be considered:

- Reputation data can change over time.
- Tor usage alone does not establish malicious intent.
- A high abuse report count may not indicate current malicious activity.
- Authentication failure volume should be interpreted within a defined time window.
- The model does not currently account for successful logins following failed attempts.
- The model does not directly incorporate Wazuh rule severity.
- A low score does not guarantee that an event is benign.

These limitations reinforce the need for analyst review and correlation with additional security telemetry.

---

## 🧠 Skills Demonstrated

| Skill | Application |
|-------|-------------|
| SOAR Automation | Developed a multi-stage security enrichment and response workflow |
| Wazuh Alert Processing | Parsed and normalized security event data |
| Threat Intelligence | Integrated AbuseIPDB reputation lookups |
| REST API Integration | Configured authenticated HTTP requests |
| JavaScript | Implemented parsing, scoring, and conditional logic |
| Risk Assessment | Developed a weighted security scoring model |
| Incident Prioritization | Classified alerts by severity |
| Workflow Orchestration | Routed incidents to appropriate response actions |
| Incident Documentation | Automated Trello investigation ticket creation |
| Security Notifications | Configured severity-aware ntfy notifications |
| JSON Processing | Extracted and merged structured security data |
| Dashboard Development | Presented incident details through HTML |

---

## 💡 What I Learned

### Building Multi-Stage SOAR Workflows

This project strengthened my understanding of how multiple automation nodes can work together to process a security event from initial detection through enrichment and notification.

### Combining Security Telemetry With Threat Intelligence

I learned how to merge internal security alert data with external reputation indicators to produce a more contextualized incident assessment.

### Developing Transparent Risk Scoring

Implementing a weighted scoring model demonstrated how individual indicators can contribute to an overall risk classification.

The reasons array also reinforced the importance of making automated security decisions understandable to analysts.

### Implementing Conditional Incident Routing

I gained experience configuring logic that determines which response actions execute based on the severity of an alert.

### Automating Incident Documentation

Integrating Trello demonstrated how security findings can be automatically converted into structured investigation records.

### Designing Reusable Workflow Components

Centralizing alert normalization and incident summary formatting made the workflow easier to maintain and extend.

---

## 🚀 Future Improvements

### 1. Integrate With a Live Wazuh Environment

Configure Wazuh to forward qualifying security alerts to the n8n webhook automatically.

This would extend the project from simulated alert processing to continuous security monitoring.

### 2. Improve the Risk-Scoring Model

Expand the model to incorporate:

- Wazuh rule severity
- Authentication attempts within a defined time window
- Repeated activity across multiple hosts
- Successful logins following failed attempts
- Asset criticality
- Historical incident activity

The weights and thresholds could then be evaluated against labeled security events to reduce false positives and improve prioritization.

### 3. Add Automated Enrichment From Multiple Sources

Integrate additional threat intelligence providers to compare reputation indicators and reduce reliance on a single source.

### 4. Add Duplicate Alert Detection

Implement incident correlation and deduplication to prevent repeated alerts from generating unnecessary Trello tickets.

### 5. Implement Error Handling and Retry Logic

Add dedicated handling for:

- API rate limits
- Authentication failures
- Invalid IP addresses
- Missing alert fields
- External service outages
- Failed ticket creation
- Notification delivery errors

### 6. Expand the HTML Dashboard

Move dashboard generation into a shared reporting branch so MEDIUM and LOW alerts can also be visualized.

### 7. Improve Incident Lifecycle Management

Expand the Trello integration to support:

- Investigation ownership
- Incident status tracking
- Escalation history
- Analyst notes
- Resolution documentation

### 8. Strengthen Notification Security

Replace public notification topics with authenticated, access-controlled messaging infrastructure suitable for sensitive security events.

---

## 🔐 Security Considerations

Several security controls should be considered before adapting this workflow to a production SOC environment.

- Authenticate and validate incoming webhook requests.
- Store API credentials securely in n8n.
- Restrict access to workflow execution data.
- Avoid exposing sensitive incident information through public notification topics.
- Validate submitted JSON and IP address fields.
- Implement appropriate rate limiting and retry controls.
- Avoid treating external reputation scores as conclusive evidence of malicious activity.
- Require analyst review before taking disruptive response actions.
- Sanitize untrusted values before rendering them in HTML.
- Protect Trello boards and incident records with appropriate access controls.

---

## 📚 References

- [n8n Documentation](https://docs.n8n.io/)
- [Wazuh Documentation](https://documentation.wazuh.com/)
- [AbuseIPDB API Documentation](https://docs.abuseipdb.com/)
- [Trello Developer Documentation](https://developer.atlassian.com/cloud/trello/)
- [ntfy Documentation](https://docs.ntfy.sh/)
- [MITRE ATT&CK — Brute Force (T1110)](https://attack.mitre.org/techniques/T1110/)

---

## 🏁 Conclusion

This project demonstrates how SOAR automation can streamline the enrichment, prioritization, and documentation of SSH brute-force security alerts.

By combining Wazuh security telemetry with AbuseIPDB threat intelligence, I developed a workflow that normalizes alert data, calculates weighted risk scores, classifies incident severity, and routes alerts to the appropriate notification and ticketing systems.

The project provided hands-on experience with security workflow orchestration, API integration, JavaScript-based risk scoring, incident prioritization, and automated security reporting.

It also established a foundation for more advanced SOC automation involving live SIEM integrations, multi-source threat intelligence, incident correlation, and automated investigation workflows.
