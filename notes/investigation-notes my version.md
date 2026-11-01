# Investigation Notes 

## 1. Environment Baseline

| Attribute | Observed Value |
|---|---|
| Operating System | Rocky Linux 9.8 |
| Virtualization | VMware |
| Wazuh Manager | 4.13.1 |
| Wazuh Indexer | 4.13.1 |
| Wazuh Dashboard | 4.13.1 |
| Mock API | Python HTTP server |
| API Address | 127.0.0.1 |
| API Port | 5000 |
| Integration Name | custom-ticket |
| Integration Script | /var/ossec/integrations/custom-ticket |

The Wazuh Manager baseline was verified:

```text
active
WAZUH_VERSION="v4.13.1"
WAZUH_REVISION="rc1"
WAZUH_TYPE="server"
```

The integration directory was present:

`/var/ossec/integrations`

## 2. Alert Identification

The alert source was:

`/var/ossec/logs/alerts/alerts.json`

Recent alerts were inspected using Python JSON parsing.

The output contained several Windows endpoint events:

| Rule ID | Level | Agent | Description |
|---|---:|---|---|
| 597 | 5 | DESKTOP-9MMM37V | Registry Key Entry Deleted |

Multiple alerts with Rule ID 597 were visible.

The output also contained alerts originating from the Manager host:

| Rule ID | Level | Agent | Description |
|---|---:|---|---|
| 533 | 7 | localhost.localdomain | Listened ports status (netstat) changed (new port opened or closed). |
| 5501 | 3 | localhost.localdomain | PAM: Login session opened. |
| 5502 | 3 | localhost.localdomain | PAM: Login session closed. |
| 5402 | 3 | localhost.localdomain | Successful sudo to ROOT executed. |

Rule ID 533 was selected for the integration test.

The selected alert was relevant to the lab because it provided a concrete detection that could be converted into a structured ticket.

## 3. Local Ticket API Implementation

A local Python ticket receiver was created:

`/opt/wazuh-ticket-lab/mock_ticket_api.py`

The receiver used Python's HTTP server functionality.

The API exposed:

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Verify API availability |
| `/tickets` | POST | Receive ticket payloads |

The server was configured to listen on:

`127.0.0.1:5000`

Ticket records were stored in:

`/opt/wazuh-ticket-lab/tickets/`

### API Health Verification

The health endpoint was tested before integration execution.

The receiver returned:

```json
{
  "status": "ok"
}
```

The server log recorded:

```text
127.0.0.1 - - [01/Nov/2026 07:06:22] "GET /health HTTP/1.1" 200 -
```

The listener inspection showed:

```text
127.0.0.1:5000
```

This confirmed that the local ticket API was listening on the expected loopback address and port.

## 4. Custom Integration Script

The integration script was created at:

`/var/ossec/integrations/custom-ticket`

The script accepted the alert file, API key argument, and webhook URL supplied by Wazuh.

It performed the following operations:

1. Loaded the alert JSON.
2. Extracted the alert ID and timestamp.
3. Read rule ID, level, description, and groups.
4. Extracted agent ID, name, and IP address.
5. Mapped the Wazuh rule level to a ticket priority.
6. Constructed a JSON payload.
7. Submitted the payload to the mock ticket API.
8. Reported HTTP and connection failures.

### Priority Mapping

| Rule Level | Ticket Priority |
|---|---|
| 14 and above | Critical |
| 12–13 | High |
| 7–11 | Medium |
| Below 7 | Low |

The selected Rule ID 533 had a level of 7. The integration therefore assigned it the priority `Medium`.

### Script Permissions

The script was assigned the following ownership and permissions:

```bash
sudo chown root:wazuh /var/ossec/integrations/custom-ticket
sudo chmod 750 /var/ossec/integrations/custom-ticket
```

This allowed the integration script to be executed with the required access.

## 5. Wazuh Configuration

The original Manager configuration was backed up:

`/var/ossec/etc/ossec.conf.pre-lab124`

The backup listing showed a file size of 9,275 bytes.

The custom integration configuration included:

| Setting | Value |
|---|---|
| Name | custom-ticket |
| Hook URL | http://127.0.0.1:5000/tickets |
| Rule ID filter | 533 |
| Alert format | json |
| API key | Local lab placeholder |

The configuration was validated:

```bash
sudo xmllint --noout /var/ossec/etc/ossec.conf
```

No validation error was returned.

The Wazuh Manager was restarted successfully, and the service-state check returned:

```text
active
```

The process check returned:

```text
wazuh-remoted is running...
wazuh-analysisd is running...
wazuh-integratord is running...
```

This confirmed that the Integrator process was running after the integration was configured.

## 6. Integration Execution Evidence

The Manager log contained the following entries:

```text
2026/11/01 07:17:22 wazuh-integratord: INFO: Started (pid: 9126).
2026/11/01 07:17:22 wazuh-integratord: INFO: Enabling integration for: 'custom-ticket'.
```

This confirmed that the Integrator started and enabled the configured custom integration.

The receiver log recorded:

```text
127.0.0.1 - - [01/Nov/2026 07:17:30] "POST /tickets HTTP/1.1" 201 -
```

HTTP 201 indicated that the ticket creation request was accepted and a new ticket record was created.

## 7. Ticket Creation Evidence

The ticket directory contained:

```text
SOC-1931A0B16197.json
```

The directory listing reported:

- File size: 831 bytes.
- Owner: root.
- Group: root.
- Date: November 1, 2026.
- Time: 07:17.

The ticket inspection returned:

| Field | Value |
|---|---|
| Ticket ID | SOC-1931A0B16197 |
| Status | NEW |
| Summary | Wazuh L7 Listened ports status (netstat) changed (new port opened or closed). |
| Alert ID | 1793497650.1763856 |
| Priority | Medium |
| Agent | localhost.localdomain |

The ticket was associated with the following detection:

| Detection Attribute | Value |
|---|---|
| Rule ID | 533 |
| Rule Level | 7 |
| Rule Description | Listened ports status (netstat) changed (new port opened or closed). |
| Rule Group | ossec |
| Rule Fired Times | 1 |
| Alert Timestamp | 2026-11-01 07:17:30.311 |

### Ticket and Alert Correlation

The generated ticket contained the alert ID, rule summary, priority, and agent identity.

The returned ticket ID was:

`SOC-1931A0B16197`

The ticket status was:

`NEW`

The corresponding receiver response was HTTP 201, and the ticket JSON file was present in the ticket directory.

Together, these observations provide evidence of successful alert submission and ticket creation.

## 8. Important Endpoint Attribution Finding

The alert output included multiple events from the Windows endpoint:

`DESKTOP-9MMM37V`

These included Rule ID 597 registry key deletion alerts.

However, the generated ticket showed:

`Agent: localhost.localdomain`

The selected integration filter was Rule ID 533, which corresponded to a Manager-host event describing a listened-port status change.

Therefore, the created ticket was generated from the Wazuh Manager host's alert, not from one of the Windows Rule ID 597 events.

This distinction is important when describing the lab outcome. The workflow successfully processed the selected Rule ID 533 alert, but Windows endpoint Rule ID 597 ticket creation was not demonstrated.

## 9. Findings

| Area | Finding | Status |
|---|---|---|
| Manager service | Active | Verified |
| Manager version | 4.13.1 | Verified |
| Original Integrator state | Not running before integration configuration | Verified |
| Integrator after configuration | Running | Verified |
| Integration registration | custom-ticket enabled | Verified |
| Alert filtering | Rule ID 533 configured | Verified |
| API listener | 127.0.0.1:5000 | Verified |
| API health | HTTP 200 | Verified |
| Integration request | POST /tickets returned HTTP 201 | Verified |
| Ticket creation | One JSON ticket file created | Verified |
| Ticket status | NEW | Verified |
| Ticket priority | Medium | Verified |
| Ticket endpoint attribution | localhost.localdomain | Verified |
| Windows Rule ID 597 ticketing | Not demonstrated | Not tested |
| Duplicate submission handling | Not independently tested | Not tested |
| External ITSM integration | Not configured | Not tested |
| Analyst assignment and ticket lifecycle | Not implemented | Not tested |

