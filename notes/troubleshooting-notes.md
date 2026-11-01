
# Troubleshooting Notes — Wazuh Alert-to-Ticket Workflow Integration

## 1. Wazuh Integrator Initially Not Running

### Observation

The initial process check returned:

```text
wazuh-integratord not running...
```

Existing Manager logs contained messages such as:

```text
wazuh-integratord: INFO: Remote integrations not configured. Clean exit.
```

These messages appeared in the historical logs from October 29 through October 31.

### Explanation

At that stage, no remote or custom integration was configured.

The Integrator process did not have an active integration to execute and exited cleanly.

This was the original state before configuring `custom-ticket`.

### Resolution

A custom integration was created and registered in `/var/ossec/etc/ossec.conf`.

After the Manager restart, the process check returned:

```text
wazuh-integratord is running...
```

The Manager log also recorded:

```text
2026/11/01 07:17:22 wazuh-integratord: INFO: Started (pid: 9126).
2026/11/01 07:17:22 wazuh-integratord: INFO: Enabling integration for: 'custom-ticket'.
```

**Final result:** Integrator startup and custom integration registration were verified.

---

## 2. Local Ticket API Availability

### Objective

The custom integration needed an available HTTP endpoint to receive ticket payloads.

### API Listener

The listener inspection showed:

```text
127.0.0.1:5000
```

The service was bound to the loopback interface.

### Health Verification

The health endpoint returned:

```json
{
  "status": "ok"
}
```

The receiver log recorded:

```text
127.0.0.1 - - [01/Nov/2026 07:06:22] "GET /health HTTP/1.1" 200 -
```

### Interpretation

The local API was available before the ticket submission was attempted.

Binding to `127.0.0.1` restricted the mock API to local connections rather than exposing it to the wider network.

**Final result:** Local API health was verified.

---

## 3. Integration Script Ownership and Permissions

### Configuration

The custom script was stored at:

`/var/ossec/integrations/custom-ticket`

The following commands were used:

```bash
sudo chown root:wazuh /var/ossec/integrations/custom-ticket
sudo chmod 750 /var/ossec/integrations/custom-ticket
```

### Explanation

The script required suitable ownership and execution permissions for Wazuh integration processing.

If permissions are incorrect, the integration may fail to execute even when the Manager configuration is valid.

**Final result:** The configured ownership and permissions were applied, and the integration subsequently executed successfully.

---

## 4. Wazuh Configuration Validation

### Validation Command

```bash
sudo xmllint --noout /var/ossec/etc/ossec.conf
```

The command returned no error output.

The Manager restart completed successfully, and:

```bash
sudo systemctl is-active wazuh-manager
```

returned:

```text
active
```

### Explanation

XML validation checks the syntax of the configuration file. Successful startup and successful integration execution provide additional operational evidence beyond XML validation alone.

**Final result:** Configuration syntax and Manager startup were validated.

---

## 5. Alert Filtering and Endpoint Attribution

### Observation

The integration was configured to filter for:

```text
Rule ID: 533
```

The recent alert output also contained Windows endpoint events with:

```text
Rule ID: 597
Agent: DESKTOP-9MMM37V
Description: Registry Key Entry Deleted.
```

However, the generated ticket showed:

```text
Rule ID: 533
Agent: localhost.localdomain
```

### Explanation

The integration's rule filter selected Rule ID 533, which corresponded to a Manager-host listened-port change alert.

The Windows Rule ID 597 alerts were not the selected integration target.

### Lesson

Always compare the ticket's agent identity and rule ID with the original alert.

A successfully created ticket does not prove that the intended endpoint or detection type was processed.

**Final result:** Rule ID 533 ticket creation was verified. Windows Rule ID 597 ticket creation was not demonstrated.

---

## 6. Ticket Creation and HTTP Response

### Receiver Log

The receiver recorded:

```text
127.0.0.1 - - [01/Nov/2026 07:17:30] "POST /tickets HTTP/1.1" 201 -
```

### Ticket File

The ticket directory contained:

```text
SOC-1931A0B16197.json
```

The ticket inspection returned:

```text
Ticket ID: SOC-1931A0B16197
Status: NEW
Priority: Medium
Agent: localhost.localdomain
```

### Explanation

HTTP 201 indicated that the API successfully processed the ticket-creation request and created a new resource.

The ticket JSON file provided additional evidence that the request was persisted by the receiver.

**Final result:** Ticket creation was verified through both the HTTP response and the stored ticket record.

---

## 7. Priority Mapping

### Observation

The generated alert had:

```text
Rule ID: 533
Rule Level: 7
```

The ticket priority was:

```text
Medium
```

### Explanation

The integration script used this lab-specific mapping:

| Rule Level | Ticket Priority |
|---|---|
| 14 and above | Critical |
| 12–13 | High |
| 7–11 | Medium |
| Below 7 | Low |

Rule level 7 therefore resulted in a Medium-priority ticket.

### Lesson

Ticket priority mappings should be explicitly defined and documented. Wazuh rule levels should not automatically be treated as equivalent to an organization's incident severity classification.

**Final result:** The configured priority mapping behaved as expected for the selected alert.

---

## 8. Duplicate Handling

### Implementation Consideration

The local receiver was designed to use the alert ID when identifying an existing ticket.

This can help avoid creating multiple ticket records when the same alert is submitted repeatedly.

### Validation Status

The supplied evidence shows one successful ticket creation.

No repeated submission with the same alert ID was demonstrated.

**Final result:** Duplicate-handling behavior remains unvalidated.

---

## 9. Production Integration Limitations

The mock API was designed for a local lab environment.

It did not demonstrate:

- API authentication against an external ticketing platform.
- HTTPS certificate validation.
- Analyst assignment.
- Ticket escalation or reassignment.
- Bidirectional status synchronization.
- Ticket closure and resolution.
- Production retry queues or delivery guarantees.
- External ITSM availability monitoring.

A production integration should use authenticated HTTPS, protected credentials, appropriate access controls, structured error handling, and auditable delivery records.

---

## 10. Lessons Learned

- The Wazuh Integrator must have a configured integration to perform forwarding.
- A successful Manager restart does not by itself prove successful integration execution.
- The custom script must be executable and accessible to the Wazuh integration process.
- A local API health check should be completed before submitting alerts.
- Alert filters determine which detections are forwarded.
- Ticket endpoint attribution must be compared with the original alert.
- HTTP 201 and a stored ticket provide strong evidence of successful ticket creation.
- Ticket creation is an incident-tracking action, not confirmation of malicious activity.
- Rule-level-to-priority mapping should be documented as an explicit policy.
- Duplicate handling must be tested separately rather than assumed.
- A local mock receiver demonstrates the integration mechanism but does not replace production ITSM validation.
