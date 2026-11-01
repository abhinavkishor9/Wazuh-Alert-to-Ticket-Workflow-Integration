# Lab Timeline 

## 2026-10-29 to 2026-10-31 — Historical Baseline

| Date / Time (IST) | Activity / Observation | Result |
|---|---|---|
| 2026-10-29 07:05:14 | Wazuh logs recorded that remote integrations were not configured. | Integrator exited cleanly |
| 2026-10-30 06:07:37 | Wazuh logs again reported no remote integrations configured. | No integration active |
| 2026-10-30 06:31:08 | Integrator logged that remote integrations were not configured. | No integration active |
| 2026-10-31 07:01:38 | Integrator logged that remote integrations were not configured. | No integration active |
| 2026-10-31 07:19:53 | Integrator logged that remote integrations were not configured. | No integration active |
| 2026-10-31 07:30:23 | Integrator logged that remote integrations were not configured. | No integration active |
| 2026-10-31 07:34:58 | Integrator logged that remote integrations were not configured. | No integration active |

## 2026-11-01 — Integration Implementation and Validation

| Time (IST) | Activity / Observation | Result |
|---|---|---|
| Time not recorded | Verified Wazuh Manager version and service state. | Manager 4.13.1; active |
| Time not recorded | Inspected the Wazuh integration directory. | Integration directory present |
| Time not recorded | Checked the initial Integrator process state. | Not running |
| Time not recorded | Inspected recent JSON alerts. | Identified Windows Rule ID 597 and Manager Rule ID 533 events |
| Time not recorded | Selected Rule ID 533 as the integration filter. | Target rule established |
| Time not recorded | Created the local mock ticket API. | Receiver implementation created |
| 07:06:22 | Mock ticket API health endpoint was requested. | HTTP 200 |
| Time not recorded | Verified the mock API listener. | Listening on 127.0.0.1:5000 |
| Time not recorded | Created the custom integration script. | custom-ticket created |
| Time not recorded | Applied script ownership and permissions. | root:wazuh; mode 750 |
| Time not recorded | Backed up the Wazuh Manager configuration. | ossec.conf.pre-lab124 created |
| Time not recorded | Added the custom-ticket integration configuration with Rule ID 533 filtering. | Integration configured |
| Time not recorded | Validated ossec.conf using xmllint. | No XML syntax error reported |
| Time not recorded | Restarted the Wazuh Manager. | Restart successful |
| Time not recorded | Checked Manager and Integrator process states. | Manager active; Integrator running |
| 07:17:22 | Wazuh Integrator started with PID 9126. | Process started |
| 07:17:22 | Integrator logged that custom-ticket was enabled. | Integration enabled |
| 07:17:30.311 | Selected Rule ID 533 alert timestamp. | Alert generated at recorded time |
| 07:17:30 | Mock receiver recorded POST /tickets. | HTTP 201 |
| 07:17 | Ticket directory listing showed SOC-1931A0B16197.json. | Ticket file created |
| Time not recorded | Inspected the generated ticket record. | Status NEW; priority Medium |

