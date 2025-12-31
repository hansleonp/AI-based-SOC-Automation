# KI-gestützte Splunk Alert Triage mit n8n + OpenAI + Slack (Brute-Force Beispiel)

Dieses Tutorial zeigt, wie du eine **SOC-ähnliche Alert-Triage-Pipeline** aufbaust:

✅ **Ubuntu Test-Host** erzeugt Security Logs (z.B. Failed Logins)  
✅ **Splunk Enterprise** indexiert diese Logs und triggert einen Alert  
✅ Splunk sendet das Alert-Payload per **Webhook** an **n8n**  
✅ n8n ruft **OpenAI** auf und erzeugt eine strukturierte Bewertung (MITRE/Severity/Actions)  
✅ n8n postet das Ergebnis automatisch in **Slack** (z.B. Channel `#alerts`)  

---

## Inhalt

1. [Architektur](#1-architektur)  
2. [Voraussetzungen](#2-voraussetzungen)  
3. [VM Setup](#3-vm-setup)  
4. [Splunk Enterprise Setup](#4-splunk-enterprise-setup)  
5. [Splunk Universal Forwarder Setup (Ubuntu Test Host)](#5-splunk-universal-forwarder-setup-ubuntu-test-host)  
6. [Splunk Alert erstellen (Test-Brute-Force)](#6-splunk-alert-erstellen-test-brute-force)  
7. [n8n Installation (Docker Compose)](#7-n8n-installation-docker-compose)  
8. [n8n Workflow bauen (Webhook → OpenAI → Slack)](#8-n8n-workflow-bauen-webhook--openai--slack)  
9. [OpenAI Prompting (SOC Analyst Template)](#9-openai-prompting-soc-analyst-template)  
10. [Slack Integration](#10-slack-integration)  
11. [End-to-End Test](#11-end-to-end-test)  
12. [Troubleshooting](#12-troubleshooting)  
13. [FAQ](#13-faq)  
14. [Pro-Tipp](#14-pro-tipp)  

---

## 1) Architektur

**Komponenten (3 VMs empfohlen):**
- **VM 1:** Splunk Enterprise (Indexer + Web UI)
- **VM 2:** n8n (Docker)
- **VM 3:** Ubuntu Test Host (Splunk Forwarder, Logquelle)

**Datenfluss:**
Ubuntu Test Host → Splunk Forwarder → Splunk Index → Splunk Alert → Webhook → n8n → OpenAI → Slack

---

## 2) Voraussetzungen

### Infrastruktur
- 3 VMs oder getrennte Hosts
- Netzwerkzugriff zwischen:
  - Test Host → Splunk (Port `9997`)
  - Splunk → n8n (Port `5678`)
  - n8n → OpenAI API (Internet)
  - n8n → Slack API (Internet)

### Accounts
- **OpenAI API Account** (API Key + Credits erforderlich)
- **Slack Workspace** (für Bot Integration)

---

## 3) VM Setup

Empfohlen:
- Splunk VM: `xxx.xxx.xxx.xxx`
- n8n VM: `xxx.xxx.xxx.xxx`
- Ubuntu Test Host: wird überwacht

✅ **Tipp:** Nach jedem Meilenstein einen Snapshot erstellen.

---

## 4) Splunk Enterprise Setup

### 4.1 Splunk installieren
Splunk Enterprise von der offiziellen Website herunterladen.

Beispiel Linux (Debian/Ubuntu, `.deb`):
```bash
wget <SPLUNK_DOWNLOAD_URL>
sudo dpkg -i splunk-<VERSION>.deb
sudo apt-get -f install
````

Splunk starten:

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

Splunk Web öffnen:

```
http://<SPLUNK_SERVER>:8000
```

---

### 4.2 Receiving Port aktivieren (9997)

Splunk Web:

* **Settings → Forwarding and receiving → Configure receiving**
* **New Receiving Port → 9997**

Optional Firewall:

```bash
sudo ufw allow 9997/tcp
```

---

### 4.3 Index erstellen

Splunk Web:

* **Settings → Indexes → New Index**
* Name z.B.: `xyz-projekt`

---

## 5) Splunk Universal Forwarder Setup (Ubuntu Test Host)

### 5.1 Forwarder installieren

```bash
sudo dpkg -i splunkforwarder-<VERSION>-linux-<ARCH>.deb
sudo apt-get -f install
```

Start + Lizenz akzeptieren:

```bash
sudo /opt/splunkforwarder/bin/splunk start --accept-license
```

Status prüfen:

```bash
sudo /opt/splunkforwarder/bin/splunk status
```

---

### 5.2 Forwarder mit Splunk verbinden

```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server <SPLUNK_SERVER>:9997
sudo /opt/splunkforwarder/bin/splunk list forward-server
```

Erwartet:

* `Active forwards: <SPLUNK_SERVER>:9997`

---

### 5.3 Logs monitoren und Index setzen

```bash
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index xyz-projekt -sourcetype syslog
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index xyz-projekt -sourcetype linux_secure
sudo /opt/splunkforwarder/bin/splunk restart
```

Inputs prüfen:

```bash
sudo /opt/splunkforwarder/bin/splunk list monitor
```

---

### 5.4 Testevent erzeugen

```bash
logger "XYZ_TEST Forwarder funktioniert $(date)"
```

Splunk Suche:

```spl
index="xyz-projekt" XYZ_TEST
```

---

## 6) Splunk Alert erstellen (Test-Brute-Force)

Beispiel Suche (failed logins):

```spl
index="xyz-projekt" sourcetype=linux_secure action=failure
| stats count by _time, user, action, host
```

Jetzt:

* Save As → Alert
* Name: `Test-Brute-Force`
* Trigger: z.B. bei `count > 0` oder `count > 5`

### Test-Trigger erzeugen

Proviziere failed logins, z.B. über falsche SSH Logins auf der Test-VM.

---

## 7) n8n Installation (Docker Compose)

### 7.1 Docker installieren

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

### 7.2 Docker Compose installieren

```bash
sudo apt install -y docker-compose
```

### 7.3 n8n Compose Setup

```bash
mkdir n8n-compose
cd n8n-compose
mkdir n8n_data
nano docker-compose.yaml
```

Inhalt:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=xxx.xxx.xxx.xxx
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_SECURE_COOKIE=false
      - GENERIC_TIMEZONE=America/Toronto
    volumes:
      - ./n8n_data:/home/node/.n8n
```

Start:

```bash
sudo docker-compose up -d
```

n8n öffnen:

```
http://xxx.xxx.xxx.xxx:5678
```

---

## 8) n8n Workflow bauen (Webhook → OpenAI → Slack)

### 8.1 Webhook Node

* Node: **Webhook**
* HTTP Method: **POST**
* URL kopieren

➡️ Diese URL kommt gleich in Splunk als Alert Action.

---

### 8.2 OpenAI Node hinzufügen

* Node: **OpenAI → Message a model**
* Model z.B. `gpt-4.1-mini`
* Credentials: OpenAI API Key (Credits erforderlich)

---

### 8.3 Slack Node hinzufügen

* Node: **Slack → Send a message**
* Channel: `#alerts`
* Message: Output vom OpenAI Node (**output_text**)

---

## 9) OpenAI Prompting (SOC Analyst Template)

Im OpenAI Node **2 System Messages + 1 User Message**:

### System Prompt 1

```text
perform the following steps:

Summarize the alert - Provide a clear summary of what triggered the alert, aff systems/users are affected, and the nature of the activity (e.g., suspicious login, malware detection, lateral movement).

Enrich with threat intelligence - Correlate any IOCs (IP addresses, domains, hashes) with known threat intel sources. Highlight if the indicators are associated with known malware or threat actors.

Assess severity - Based on MITRE ATT&CK mapping, identify tactics/techniques, and provide an initial severity rating (Low, Medium, High, Critical).

Recommend next actions - Suggest investigation steps and potential containment actions.
```

### System Prompt 2

```text
Format output clearly - Return findings in a structured format (Summary, IOC Enrichment, Severity Assessment, Recommended Actions).
```

### User Prompt (dynamisch)

```text
Alert: {{ $json.body.search_name }}

Alert Details:
{{ JSON.stringify($json.body.result, null, 2) }}
```

✅ Wichtig: Für Weiterverarbeitung nutze `output_text`, **nicht** `input_text`.

---

## 10) Slack Integration

Du brauchst einen Slack Bot Token.

Offizielle n8n Anleitung:
[https://docs.n8n.io/integrations/builtin/credentials/slack/](https://docs.n8n.io/integrations/builtin/credentials/slack/)

Kurzfassung:

1. Slack API Apps Page öffnen
2. Create New App → From scratch
3. App Name vergeben
4. OAuth & Permissions → Scopes setzen (z.B. `chat:write`)
5. Install to Workspace
6. Bot User OAuth Token kopieren
7. In n8n Slack Credentials eintragen

---

## 11) End-to-End Test

1. Test Host erzeugt failed logins
2. Splunk indexiert sie
3. Splunk Alert `Test-Brute-Force` triggert
4. Splunk sendet Payload an n8n Webhook
5. n8n ruft OpenAI auf
6. Slack Channel erhält strukturierte Analyse

---

## 12) Troubleshooting

### Splunk empfängt keine Logs vom Forwarder

✅ Inputs prüfen:

```bash
sudo /opt/splunkforwarder/bin/splunk list monitor
```

✅ Port prüfen:

```bash
nc -vz <SPLUNK_SERVER> 9997
```

✅ Forwarder Logs:

```bash
sudo tail -n 80 /opt/splunkforwarder/var/log/splunk/splunkd.log
```

---

### n8n Webhook triggert nicht

✅ Webhook URL in Splunk prüfen
✅ Test mit curl:

```bash
curl -X POST http://<N8N_HOST>:5678/webhook/<PATH> \
  -H "Content-Type: application/json" \
  -d '{"test":"ok"}'
```

---

### OpenAI Node Fehler: `Invalid value: input_text`

✅ Ursache: falsches Feld verwendet
✅ Lösung: nutze **`output_text`** statt `input_text`

---

## 13) FAQ


**Benötige ich OpenAI Credits?**
Ja, API Calls benötigen Budget.

---

## 14) Pro-Tipp

Erstelle Snapshots nach:

* Splunk Installation
* Forwarder Setup
* n8n Setup
* Workflow läuft
* Slack Integration erfolgreich

```
