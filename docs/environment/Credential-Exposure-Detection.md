
# HTTP Credential Exposure Detection — SOC Investigation

## Executive Summary

This investigation evaluates whether the Enterprise SOC Lab can identify potential credential submissions over an unencrypted HTTP connection.

A controlled phishing simulation was conducted using a Windows 11 endpoint as the victim and Kali Linux as the attacker-controlled web server.

Initial monitoring with Sysmon and Wazuh successfully captured network connections to the simulated phishing server but did not generate a security alert identifying credential submission.

To address this visibility gap, Suricata was deployed on the Windows endpoint and integrated with Wazuh.

A custom Suricata rule was developed to identify HTTP POST requests containing the parameter pattern `password=`.

During subsequent controlled testing, the rule triggered successfully, and the detection appeared in Wazuh Threat Hunting.

**Final result: Successful detection of a potential credential submission over unencrypted HTTP.**

This result demonstrates detection of a risky HTTP behavior, not confirmation that valid credentials were stolen or an actual account was compromised.

## Investigation Information

| Attribute | Details |
|---|---|
| Investigation type | Controlled phishing / credential exposure simulation |
| Environment | Enterprise SOC Lab |
| Endpoint | WIN11-Client |
| Endpoint IP | 192.168.23.129 |
| Simulated attacker | Kali Linux |
| Attacker web server | 192.168.23.130 |
| Protocol | HTTP |
| Destination port | TCP/80 |
| Endpoint monitoring | Sysmon |
| Network inspection | Suricata 8.0.7 |
| SIEM | Wazuh |
| Suricata signature ID | 1000001 |
| Wazuh rule ID | 86601 |
| Wazuh alert level | 3 |
| Detection outcome | Successful |

## 1. Objectives

The investigation sought to answer the following questions:

1. Could the Windows endpoint access the simulated phishing website?
2. Would Sysmon record the browser's connection to the attacker-controlled server?
3. Would Wazuh collect the network telemetry?
4. Would existing Wazuh detection rules identify the credential-submission activity?
5. Could application-layer inspection detect the insecure login submission?
6. Could the resulting detection be forwarded to and investigated in Wazuh?

The investigation was designed to distinguish:

- **Telemetry collection:** An event was recorded.
- **Threat detection:** A rule identified potentially suspicious behavior.
- **Alert generation:** The SIEM surfaced a security-relevant event.
- **Incident confirmation:** The available evidence supported an actual compromise.

These are related but separate stages of SOC analysis.

## 2. Attack Scenario

The attacker-controlled Kali Linux VM hosted a simulated Microsoft 365 login page.

The page was accessible from the Windows 11 VM at:

```text
http://192.168.23.130/phishing.html
```

The form submitted data to:

```text
http://192.168.23.130/login.php
```

The simulated server-side handler processed the submitted test data and redirected the browser to a legitimate Microsoft website.

### Attack Sequence

```text
Kali Linux
192.168.23.130
     |
     | Hosts simulated login page
     v
Windows 11 / Edge
192.168.23.129
     |
     | Visits phishing.html
     v
Simulated Login Page
     |
     | Submits dummy credentials
     | via unencrypted HTTP POST
     v
Kali Apache / login.php
     |
     | Processes test submission
     v
Browser Redirect
     |
     v
Microsoft Website
```

The simulation used dummy credentials only.

All activity took place within the authorized virtual lab.

## 3. Initial Monitoring — Sysmon and Wazuh

Sysmon had already been installed on the Windows 11 endpoint and integrated with Wazuh.

Event ID 3 network-connection monitoring was enabled through a custom Sysmon configuration.

### Relevant Sysmon Event

Wazuh Discover showed the Windows browser communicating with the simulated phishing server.

Observed fields included:

```text
agent.name: WIN11-Client
data.win.eventdata.destinationIp: 192.168.23.130
data.win.eventdata.destinationPort: 80
data.win.eventdata.image: C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe
```

The matching event was found in:

```text
wazuh-archives-*
```

Search used:

```text
data.win.eventdata.destinationIp: "192.168.23.130" AND data.win.eventdata.destinationPort: "80"
```

### Evidence — Browser Network Connection

![Edge Connection to Simulated Phishing Server](../screenshots/wazuh-phishing-network-connection.png)

### Analysis

The Sysmon event established that the Windows browser communicated with the attacker-controlled web server over TCP port 80.

However, the event did not reveal the contents of the HTTP request or prove that a login form was submitted.

No corresponding phishing-specific alert was identified during the initial Wazuh investigation.

**Initial finding:** Network visibility existed, but application-layer credential-submission detection was insufficient.

## 4. Detection Gap

The initial investigation demonstrated a limitation of relying solely on endpoint process and network metadata.

Sysmon could identify:

- The executable initiating the connection
- Source and destination network information
- Destination port
- Connection timestamps

However, Sysmon could not determine:

- Which HTTP form fields were submitted
- Whether the user submitted a login form
- Whether a request body contained credential-like parameters
- Whether a remote website was impersonating Microsoft

Additionally, access to a website over HTTP is not automatically proof of malicious intent.

An HTTP connection can be legitimate, although transmitting credentials over that connection creates a meaningful security risk.

The investigation therefore required application-layer visibility.

## 5. Detection Engineering — Suricata

Suricata 8.0.7 was installed on the Windows 11 endpoint, together with Npcap for packet capture.

The Windows endpoint's active network interface was monitored directly.

Suricata produced JSON telemetry in:

```text
C:\Program Files\Suricata\log\eve.json
```

The Wazuh agent collected this file through the following configuration:

```xml
<localfile>
  <location>C:\Program Files\Suricata\log\eve.json</location>
  <log_format>json</log_format>
</localfile>
```

This provided centralized visibility into Suricata events through Wazuh.

## 6. Custom HTTP Credential Detection Rule

A custom rule was created in:

```text
C:\Program Files\Suricata\rules\local.rules
```

### Detection Signature

```suricata
alert http any any -> any any (msg:"SOC LAB - Possible credential submission over unencrypted HTTP"; flow:established,to_server; http.method; content:"POST"; http.request_body; content:"password="; nocase; classtype:policy-violation; sid:1000001; rev:1;)
```

### Detection Logic

The rule identifies HTTP requests meeting these conditions:

1. Traffic is recognized as HTTP.
2. The request is sent from client to server over an established connection.
3. The HTTP method is `POST`.
4. The HTTP request body contains the parameter pattern `password=`.

A match generates a Suricata alert with signature ID `1000001`.

Unlike an IP-address-based indicator rule, this detection does not depend on knowing the simulated attacker's IP in advance.

It is a behavior- and content-pattern-based detection focused on potentially insecure credential transmission.

### Limitations of the Rule

The rule does not detect every possible credential submission.

Potential gaps include:

- Different form parameter names
- JSON or other request-body formats
- URL encoding or alternate encodings
- Requests sent over HTTPS
- Login fields that do not include the literal pattern `password=`

The rule also cannot determine whether the submitted credentials are valid or whether the destination is definitively malicious.

False positives are possible when ordinary applications submit password-like parameters over HTTP.

## 7. Validation and Troubleshooting

The initial Suricata deployment successfully captured network traffic, but the first test did not produce the expected credential-submission alert.

During troubleshooting:

- The Suricata configuration passed validation.
- The custom rule was loaded.
- The packet-capture engine started successfully.
- Network-flow events appeared in the logs.
- Some early flows were observed only partially.
- The original alert search returned no results.

A later controlled browser interaction generated the HTTP POST request under conditions where Suricata successfully inspected it.

The subsequent detection confirmed that application-layer matching was functioning.

This distinction was important: successful packet capture alone did not establish successful HTTP inspection or rule matching.

## 8. Successful Detection

The Windows 11 endpoint opened the simulated phishing page in an Edge InPrivate session and submitted dummy credentials.

Suricata generated a matching security event.

The alert was forwarded through the Wazuh agent and appeared in Wazuh Discover.

### Confirmed Alert Fields

| Field | Observed value |
|---|---|
| `agent.name` | WIN11-Client |
| `data.event_type` | alert |
| `data.alert.signature` | SOC LAB - Possible credential submission over unencrypted HTTP |
| `data.alert.signature_id` | 1000001 |
| `data.alert.severity` | 1 |
| `data.app_proto` | http |
| `data.dest_ip` | 192.168.23.130 |
| `data.dest_port` | 80 |
| `data.alert.action` | allowed |

### Evidence — Suricata Detection in Discover

![Suricata HTTP Credential Detection](../screenshots/suricata-http-credential-alert.png)

### Interpretation

The detection confirmed that:

- The network sensor observed the relevant HTTP request.
- The request matched the custom detection rule.
- Suricata generated a security event.
- The event reached the Wazuh platform.

The alert did not establish that real credentials were compromised. Only dummy values were used in this simulation.

## 9. Wazuh Threat Hunting Verification

The detection was subsequently verified in the Wazuh Threat Hunting interface.

The analyst used the search:

```text
rule.groups: suricata
```

The matching event appeared in the Wazuh alerts index:

```text
wazuh-alerts-4.x-2026.10.12
```

The alert retained Suricata's signature information.

### Wazuh Alert Properties

| Property | Value |
|---|---|
| Wazuh rule ID | 86601 |
| Wazuh rule level | 3 |
| Suricata signature ID | 1000001 |
| Suricata severity | 1 |
| Protocol | HTTP |
| Action | Allowed |

### Evidence — Security Alert in Threat Hunting

![Wazuh Threat Hunting Credential Exposure Alert](../screenshots/wazuh-suricata-threat-alert.png)

This confirmed that the detection was not merely archived as raw telemetry.

It was also processed into a Wazuh security alert.

## 10. Alert Severity Assessment

The investigation identified a difference between Suricata's severity classification and Wazuh's alert level.

| System | Observed severity | Interpretation |
|---|---|---|
| Suricata | 1 | Highest Suricata severity category |
| Wazuh | 3 | Low Wazuh alert level |

Suricata and Wazuh use different severity scales.

The low Wazuh rule level may underrepresent the potential risk of credentials being transmitted over unencrypted HTTP.

### Proposed Improvement

A future custom Wazuh rule should raise the alert priority for this particular Suricata signature.

Proposed configuration:

```text
Suricata signature: 1000001
Existing Wazuh rule: 86601
Proposed Wazuh alert level: 10
```

The proposed change should apply only to the credential-exposure signature, not all Suricata events.

**Status: Planned, not yet implemented.**

## 11. Detection Timeline

The investigation progressed through the following phases:

| Phase | Activity | Outcome |
|---|---|---|
| 1 | Initial phishing simulation | Simulated form submission completed |
| 2 | Sysmon investigation | Browser connection to Kali observed |
| 3 | Wazuh baseline investigation | No relevant phishing-specific alert identified |
| 4 | Archive indexing | Raw network events became searchable |
| 5 | Suricata deployment | Packet capture established |
| 6 | Custom rule development | HTTP credential-pattern rule created |
| 7 | Initial rule testing | No alert generated |
| 8 | Retesting | Custom Suricata rule triggered |
| 9 | Wazuh Discover verification | Detection event found |
| 10 | Wazuh Threat Hunting verification | Security alert confirmed |
| 11 | Severity review | Wazuh level 3 identified for future tuning |

This is a sequence of investigation activities rather than a second-by-second attack timeline.

Exact timestamps can be extracted from the saved event evidence if a more detailed chronology is required.

## 12. MITRE ATT&CK Context

The simulated scenario is relevant to the following MITRE ATT&CK technique:

| Technique | ID | Relevance |
|---|---|---|
| Phishing | T1566 | Simulated phishing activity and credential collection context |

The broader scenario involves a deceptive login page. The exact ATT&CK phishing subtechnique depends on the initial delivery method, which is not established by the Suricata HTTP alert alone.

The HTTP submission alert itself detects a network request pattern, not the entire phishing technique.

No additional post-compromise techniques, such as persistence or command-and-control, were demonstrated in this investigation.

## 13. Findings

### Finding 1 — Endpoint Network Visibility Was Functional

**Status:** Confirmed

Sysmon successfully recorded browser communication with the simulated phishing server.

Wazuh ingested and archived the relevant network telemetry.

### Finding 2 — Default Detection Coverage Was Insufficient

**Status:** Observed in the tested scenario

No relevant phishing-specific alert was identified during initial monitoring.

The existing telemetry established a connection but did not independently identify a credential submission.

### Finding 3 — Application-Layer Detection Improved Visibility

**Status:** Confirmed

Suricata identified the tested HTTP POST request pattern and generated the custom alert.

The detection did not require hardcoding the destination IP address.

### Finding 4 — Wazuh Successfully Processed the Detection

**Status:** Confirmed

The Suricata event appeared in both Discover and Threat Hunting.

The processing pipeline from network packet inspection to SIEM alerting was successfully validated.

### Finding 5 — Alert Prioritization Requires Tuning

**Status:** Improvement identified

The alert was assigned Wazuh rule level 3.

A targeted higher-priority rule is proposed to improve analyst visibility and prioritization.

### Finding 6 — Operational Deployment Is Not Yet Complete

**Status:** Pending

Suricata was run interactively during testing.

Automatic background operation and long-term monitoring configuration remain to be implemented.

## 14. Recommended SOC Response

For a real-world alert indicating possible credential exposure over HTTP, an analyst should:

1. Verify the HTTP request, source endpoint, destination, and detection signature.
2. Determine whether the destination is legitimate, suspicious, or known malicious.
3. Investigate whether sensitive credentials may have been transmitted.
4. Correlate the event with browser activity and endpoint telemetry.
5. Assess potential impact on the associated user or account.
6. Escalate confirmed or strongly suspected exposure according to organizational procedures.
7. Coordinate credential rotation, session revocation, or containment when warranted.
8. Preserve relevant evidence and document the investigation outcome.

In this controlled lab, the event was generated using dummy credentials and did not require real-world account remediation.

## 15. Future Detection Improvements

### Wazuh Alert Tuning

Create and validate a targeted Wazuh rule that elevates signature `1000001` from rule level 3 to level 10.

### Persistent Suricata Operation

Configure Suricata to run automatically in the background so monitoring continues without an open PowerShell session.

### Expanded HTTP Detection

Test additional request-body formats and credential-field patterns.

### Correlation with Sysmon

Correlate HTTP alerts with Sysmon Event ID 1 and Event ID 3 data to build a richer endpoint-to-network investigation timeline.

### Broader Network Detection Coverage

Introduce maintained rulesets, threat intelligence, and additional indicators that support more realistic intrusion-detection scenarios.

### Alert Quality Evaluation

Evaluate detection precision, false positives, and missed cases using multiple benign and simulated malicious HTTP requests.

## 16. Conclusion

This investigation demonstrated the difference between collecting network telemetry and detecting security-relevant behavior.

Sysmon and Wazuh initially provided visibility into the Windows endpoint's connection to the simulated phishing server but did not identify the credential-submission activity.

By introducing Suricata application-layer inspection and developing a custom detection rule, the SOC lab successfully identified a potential credential submission over unencrypted HTTP.

The alert was processed by Wazuh and became visible in Threat Hunting.

The project demonstrates an iterative detection-engineering workflow:

**Baseline monitoring → Detection gap → New telemetry source → Custom rule → Retesting → Confirmed security alert → Severity improvement plan**

Future work will focus on alert prioritization, automatic background monitoring, and broader detection coverage.
