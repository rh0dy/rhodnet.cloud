+++
date = '2025-03-20T19:03:06Z'
draft = false
title = 'Google Workspace Alerts Wodle'
tags = ['open-source', 'security-engineering', 'security-monitoring', 'siem', 'wazuh', 'google-workspace']
+++

## Real-World Wodle Implementation

In my [previous post](/blog/creating-a-custom-wazuh-wodle), I covered the theory behind creating custom wodles for Wazuh. Now, I want to share a practical implementation I've been using in a production environment: a wodle for monitoring [Google Workspace Alert Center](https://workspace.google.com/intl/en_uk/products/admin/alert-center/) alerts and notifications.

This is an example of extending Wazuh's monitoring capabilities to cover cloud services. Without this custom integration, our security team would need to manually log into Google Workspace to check the Alert Center. By bringing these alerts into our central Wazuh instance, we've streamlined our security operations and gained cross-platform visibility and correlation.

![alt text](google-workspace-alerts-wodle-1.png)

![alt text](google-workspace-alerts-wodle-2.png)

## Capabilities

The wodle connects to the Google Workspace Alert Center API and forwards alerts to Wazuh, allowing you to monitor Google security events alongside your other security telemetry.

The wodle processes over 40 different [alert types](https://developers.google.com/admin-sdk/alertcenter/reference/alert-types) from Google Workspace, including: phishing reports, leaked password notifications, suspicious login attempts, government-backed attack warnings, device compromise alerts, data loss prevention triggers, admin account changes, SSO profile modifications, drive settings changes, new user additions, and account suspension warnings.

## How It Works

- Authenticates with Google Workspace using a service account
- Fetches alerts from the Alert Center API
- Reformats the data for Wazuh compatibility
- Maintains state to avoid duplicate alerts
- Forwards processed alerts to your Wazuh server


```mermaid
flowchart LR
    subgraph Google["Google Workspace"]
        API[Google Alert Center API]
    end

    subgraph Wodle["Wodle"]
        Auth[Google API Service Account Credentials] --> Fetch[Alert Fetcher]
        Fetch -->|"Fetch alerts (every 10min)"| API
        API -->|"Return alerts"| Fetch
        Fetch --> Process[Alert Processor]
        Process -->|"Check against processed IDs"| State[(State File)]
        State -->|"Last timestamp & processed IDs"| Process
        Process -->|"Update state"| State
        Process --> Format[Alert Formatter]
    end

    subgraph Wazuh["Wazuh Server"]
        Format -->|"Flattened alerts"| WazuhD[Wazuh Daemon]
        WazuhD --> Rules[Custom Rules]
        Rules --> Index[Wazuh Indexer]
        Index --> Dashboard[Wazuh Dashboard]
    end

    style Google fill:#000,stroke:#fff
    style Wodle fill:#000,stroke:#fff
    style Wazuh fill:#000,stroke:#fff
```

## Implementation Steps

### Google API Setup

This is probably the most complicated part of the setup. There are detailed step in the [Google dev docs](https://developers.google.com/workspace/guides/get-started), but essentially:

1. Create a Google Cloud project
2. Enable the Google Workspace Alert Center API
3. Create a service account with domain-wide delegation
4. Add the scope https://www.googleapis.com/auth/apps.alerts to the domain-wide delegation settings
5. Download the service account's credentials JSON file

### Wodle Installation

On your Wazuh server:

```bash
# Create directory
sudo mkdir -p /var/ossec/wodles/google_workspace_alerts

# Copy files
sudo cp -r * /var/ossec/wodles/google_workspace_alerts/

# Install dependencies
sudo /var/ossec/framework/python/bin/pip3 install google-api-python-client google-auth google-auth-httplib2

# Make executable
sudo chmod +x /var/ossec/wodles/google_workspace_alerts/google-workspace-alerts
```

### Configuration

Create a config file at `/var/ossec/wodles/google_workspace_alerts/config.json` with your Google API credentials:

```json{filename="config.json"}
{
  "project_id": "my-project-123456",
  "private_key_id": "a1b2c3d4e5f6g7h8i9j0",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgk...\n-----END PRIVATE KEY-----\n",
  "client_email": "my-service-account@my-project-123456.iam.gserviceaccount.com",
  "client_id": "123456789012345678901",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/my-service-account%40my-project-123456.iam.gserviceaccount.com",
  "delegated_account": "admin@yourdomain.com",
  "log_level": "info"
}
```

The `delegated_account` needs to be a Google Workspace admin with appropriate permissions.

### Wazuh Configuration

Update `/var/ossec/etc/ossec.conf` to include the wodle:

```xml{filename="ossec.conf"}
<!-- Add this in the global section -->
<wodle name="command">
  <disabled>no</disabled>
  <tag>google-workspace-alerts</tag>
  <command>/var/ossec/wodles/google_workspace_alerts/google_workspace_alerts</command>
  <interval>10m</interval>
  <ignore_output>no</ignore_output>
  <run_on_start>yes</run_on_start>
  <timeout>0</timeout>
</wodle>
```

### Custom Rules

Add a rule in `/var/ossec/etc/rules/local_rules.xml` (may get wiped after Wazuh upgrades) or a new file in `/var/ossec/ruleset/rules` to handle the alerts:

```xml{filename="local_rules.xml"}
<group name="google_workspace,">
  <rule id="100000" level="5">
    <decoded_as>json</decoded_as>
    <field name="wodle">google-workspace-alerts</field>
    <description>Google Workspace Alert: $(source) - $(type)</description>
    <group>google_workspace_alert,</group>
  </rule>
</group>
```

Restart Wazuh:

```bash
/var/ossec/bin/wazuh-control restart
```

### Testing

You can run the wodle manually for testing:

```bash
/var/ossec/wodles/google_workspace_alerts/google-workspace-alerts
```

Check the logs at `/var/ossec/logs/google_workspace_alerts_wodle.log` for any issues.

### State Management

The wodle maintains state in a JSON file here `/var/ossec/var/google_workspace_alerts_wodle_state.json` to avoid processing duplicate alerts. It stores:

- The timestamp of the last processed alert
- A list of processed alert IDs (up to 10,000)

```json{filename="example_google_workspace_alerts_wodle_state.json"}
{
   "last_processed_time":"2023-10-02T08:55:31.827900+00:00",
   "processed_alert_ids":[
      "fe65ceb5-85ec-407e-aa43-05ed62612345",
      "ecf27e60-255b-460e-863b-1234507bdfbe",
      "bb86292f-f448-4e9e-8d31-014f0b112345",
      "600d7457-546d-4122-aeab-123456c1f667",
      "e7ffcddf-9105-4cd4-9bc2-398dfd123452",
      "a5ebc14f-fc7f-4c2d-be6c-123456955522",
      "700ef36a-7d38-4d8e-81da-6a81c412345",
      "deed3c70-b8ed-4cf7-82a4-1234541a6cc6",
      "f2f9fba5-7bdb-4e7e-a8c5-863d15712345",
      "47783ea1-bb57-4bc1-9541-123458869213",
      "44b971f3-1ea0-4d00-853c-391198612345"
   ]
}
```

If no state file exists on first run, it fetches alerts from the last 24 hours.