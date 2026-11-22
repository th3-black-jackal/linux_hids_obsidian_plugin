# HIDS SIEM Fetcher

An Obsidian plugin for fetching **HIDS/SIEM alerts and events** directly from a Django-based SIEM backend and converting them into structured Obsidian notes.

The plugin is implemented against the project's `SIEM_API_DOCUMENTATION.md` specification and is designed to support:

* Threat Intelligence workflows
* Threat Hunting
* HIDS event investigation
* Incident analysis
* SIEM-to-Obsidian knowledge management
* Incremental event synchronization

---

## Features

### Fetch SIEM Events

The plugin communicates with the SIEM API using the base API path:

```text
/api/siem/
```

Authentication uses Django Token Authentication:

```http
Authorization: Token <token>
```

The plugin supports the SIEM class mapping:

```text
0–8
```

and implements the filtering options exposed by the SIEM API.

Supported filters include:

```text
since
until
hours
days
date
type
host
min_confidence
ordering
```

Pagination is also supported automatically.

---

## Obsidian Note Generation

Every fetch operation creates a new timestamped Markdown note inside the configured Obsidian folder.

Example:

```text
SIEM/
└── SIEM-Events-2026-08-22-10-30.md
```

Each generated note contains YAML frontmatter such as:

```yaml
---
source: hids-siem
filter: latest-events
event_count: 42
alert_count: 7
tags:
  - siem
  - hids
---
```

This makes the imported data immediately usable with Obsidian features such as:

* Search
* Tags
* Dataview
* Graph View
* Backlinks
* Threat investigation notes
* Case-management workflows

---

## Event Tables

Events are converted into Markdown tables containing important SIEM fields.

Example:

| Time                | Host           | Class             | Confidence | Alert | Topic   | ID   |
| ------------------- | -------------- | ----------------- | ---------: | ----- | ------- | ---- |
| 2026-08-22 09:12:41 | workstation-01 | Process Execution |       0.97 | Yes   | process | 4812 |
| 2026-08-22 09:10:13 | server-02      | Network Activity  |       0.83 | No    | network | 4811 |

The generated records include:

* Event timestamp
* Host
* Detection class
* Confidence
* Alert status
* Event topic
* Event ID

---

## Aggregate Statistics

The plugin can optionally retrieve SIEM statistics from:

```text
/api/siem/events/stats/
```

The statistics are inserted into the generated Obsidian note before the event table.

This can be useful for quickly understanding activity across:

* Hosts
* Detection classes
* Alerts
* Event types
* Confidence levels
* Time ranges

---

## Available Commands

The plugin provides several commands through the Obsidian Command Palette.

### Latest Events

Fetch the most recent SIEM events.

Ordering defaults to:

```text
-event_time
```

which returns the newest events first.

---

### Latest Alerts

Fetch only events marked as alerts.

---

### Alerts — Last 24 Hours

Fetch alerts generated during the previous 24 hours.

---

### Alerts — Last 7 Days

Fetch alerts generated during the previous seven days.

---

### Custom SIEM Filter

Open a filter dialog and construct a custom SIEM query.

Available parameters include:

```text
since
until
hours
days
date
type
host
min_confidence
ordering
```

The dialog can be opened from either:

* The Obsidian Command Palette
* The plugin's shield icon in the ribbon

---

## Incremental Synchronization

The plugin supports incremental synchronization so that Obsidian does not continuously download the same SIEM events.

After a successful synchronization, the plugin stores the newest:

```text
received_at
```

timestamp as a local watermark.

For example:

```text
Last synchronization:
2026-08-22T09:45:31Z
```

The next synchronization automatically sends that value using the API's:

```text
since
```

filter.

Conceptually:

```text
SIEM Database
     │
     │ events since watermark
     ▼
HIDS SIEM Fetcher
     │
     ├── Create Obsidian note
     │
     └── Save newest received_at
                    │
                    ▼
              New Watermark
```

This prevents previously synchronized rows from being fetched repeatedly.

---

## Pagination

The SIEM API's pagination is handled automatically.

If an API response contains a:

```json
{
  "next": "..."
}
```

link, the plugin follows it and continues retrieving data.

A configurable page limit prevents accidentally downloading an excessive number of events in a single synchronization.

---

## CORS

The plugin uses Obsidian's:

```typescript
requestUrl()
```

API instead of the browser's normal `fetch()` implementation.

As a result, requests are made through Obsidian and are not subject to normal browser CORS restrictions.

This is particularly useful when the SIEM backend is hosted on another machine, for example:

```text
Obsidian Desktop
192.168.1.20
       │
       │ HTTP API
       ▼
Django SIEM Backend
192.168.1.50:8000
```

---

## Error Handling

API and network errors are surfaced through Obsidian notices.

### Authentication Failure

For example:

```text
HTTP 401
```

usually indicates an invalid or expired API token.

---

### Invalid Filters

For malformed SIEM queries, the backend may return:

```text
HTTP 400
```

The plugin attempts to display the offending filter field returned by the API.

---

### Network Failures

Connection failures are also surfaced to the user.

Examples include:

* Backend offline
* Incorrect IP address
* Incorrect port
* Connection timeout
* DNS failure

---

## Installation

Download the plugin archive:

```text
hids-siem-fetcher-plugin.zip
```

Extract it into:

```text
<your-vault>/.obsidian/plugins/
```

For example:

```text
Threat-Intel-Vault/
└── .obsidian/
    └── plugins/
        └── hids-siem-fetcher/
            ├── main.js
            ├── manifest.json
            └── styles.css
```

Then open Obsidian and navigate to:

```text
Settings
  → Community plugins
  → Installed plugins
  → HIDS SIEM Fetcher
  → Enable
```

---

## Configuration

Open the plugin settings page and configure the SIEM backend.

Example backend URL:

```text
http://192.168.1.50:8000
```

The plugin automatically communicates with the SIEM endpoints underneath:

```text
/api/siem/
```

You will also need to configure your API token.

Example:

```text
Backend Base URL:
http://192.168.1.50:8000

API Token:
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

You can additionally configure:

* Output folder
* Maximum pagination depth
* Statistics inclusion
* Synchronization behavior

---

## Building From Source

The full TypeScript source project is available in:

```text
hids-siem-fetcher-plugin-source.zip
```

Install dependencies:

```bash
npm install
```

Build the plugin:

```bash
npm run build
```

The generated JavaScript can then be installed into the Obsidian plugin directory.

---

## API Authentication

The plugin currently assumes the authentication mechanism documented by the SIEM API:

```http
Authorization: Token <token>
```

The exact authentication mechanism is ultimately controlled by the Django project's authentication configuration.

If the backend is changed to another authentication method, such as:

* JWT
* Session authentication
* OAuth
* Bearer tokens

the authorization logic must be updated.

The relevant implementation is located inside:

```typescript
SiemClient.get()
```

For example, JWT authentication may require changing:

```http
Authorization: Token <token>
```

to:

```http
Authorization: Bearer <token>
```

---

## Architecture

```text
┌─────────────────────────────┐
│         Linux HIDS          │
│                             │
│  Processes / Files / Logs   │
│  Network / Security Events  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Django Backend        │
│                             │
│      SIEM API / Database    │
│                             │
│        /api/siem/           │
└──────────────┬──────────────┘
               │
               │ REST API
               ▼
┌─────────────────────────────┐
│     HIDS SIEM Fetcher       │
│      Obsidian Plugin        │
│                             │
│ • API Client                │
│ • Filters                   │
│ • Pagination                │
│ • Incremental Sync          │
│ • Markdown Generator        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Obsidian Vault        │
│                             │
│ • SIEM Events               │
│ • Alerts                    │
│ • Threat Intel              │
│ • Investigation Notes       │
│ • Threat Hunting            │
└─────────────────────────────┘
```

---

## Example Workflow

A typical workflow could look like:

```text
Linux Host
    │
    ▼
HIDS detects suspicious activity
    │
    ▼
Alert stored in SIEM database
    │
    ▼
Django SIEM API
    │
    ▼
HIDS SIEM Fetcher
    │
    ▼
Obsidian Investigation Note
    │
    ├── IOC enrichment
    ├── MITRE ATT&CK mapping
    ├── Threat actor research
    ├── Related incidents
    └── Investigation timeline
```

This allows Obsidian to act as the investigation and knowledge layer on top of the HIDS/SIEM infrastructure.

---

## Project Goals

The long-term goal of the integration is to bridge traditional HIDS/SIEM telemetry with an analyst-oriented knowledge environment.

Potential future capabilities include:

* IOC extraction
* IOC enrichment
* MITRE ATT&CK mapping
* Threat actor profiling
* Threat Intelligence feeds
* Investigation timelines
* Case management
* Malware analysis notes
* Threat hunting queries
* Event correlation
* Graph-based incident analysis
* MISP integration
* OpenCTI integration
* Local LLM-assisted analysis
* Automated investigation summaries
* Detection engineering notes

---

## Security Considerations

The Obsidian vault may contain sensitive security information.

Consider protecting:

* SIEM API tokens
* Internal IP addresses
* Hostnames
* Investigation notes
* IOC data
* Incident information


Production versions of the plugin should preferably use Obsidian's plugin settings storage for sensitive configuration.

---


Backend integration targets:

* Python
* Django
* Django REST APIs
* HIDS/SIEM event databases

---

## Status

**Working prototype / active development**

Current functionality includes:

* [x] SIEM API integration
* [x] Event retrieval
* [x] Alert retrieval
* [x] Time-based filters
* [x] Host filtering
* [x] Event-type filtering
* [x] Confidence filtering
* [x] Custom filter dialog
* [ ] MITRE ATT&CK mapping
* [ ] Threat Intelligence feeds
Future work:

* [ ] LLM-assisted threat analysis
* [ ] Investigation graph generation

---

## Disclaimer
I have used AI to generate the code

This project is intended for defensive cybersecurity, threat intelligence, threat hunting, incident investigation, research, and authorized security monitoring environments.
