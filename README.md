# 🔐 Enterprise SIEM Lab – ELK Stack

Ein selbst aufgebautes Security Information and Event Management (SIEM) System
basierend auf dem ELK Stack, deployed auf einem privaten Homelab Server.

Dieses Projekt wurde als praktisches Portfolio-Projekt für eine Karriere als
SOC Analyst aufgebaut.

---

## 🎯 Projektziel

Zentralisierte Log-Sammlung, Verarbeitung und Visualisierung von:
- **Windows Event Logs** (Security, System, Application)
- **Linux System Logs** (syslog, auth.log)
- **Docker Container Logs** (alle laufenden Container)

---

## 🏗️ Architektur

```
Windows VM                    Linux Server (freeza)
    │                              │
Winlogbeat                    Filebeat
    │                              │
    └──────────┬───────────────────┘
               ▼
           Logstash:5044
         (ETL Pipeline)
               │
               ▼
        Elasticsearch
         (Datenspeicher)
               │
               ▼
            Kibana
    https://kibana.kamicorp.ch
```

### Warum diese Architektur?

- **Filebeat/Winlogbeat** als leichtgewichtige Agents – minimaler Ressourcenverbrauch
  auf den überwachten Systemen
- **Logstash als zentrale Pipeline** – alle Logs laufen durch einen einzigen
  Verarbeitungspunkt (Single Source of Truth)
- **Separate Indizes pro OS** (`siem-windows-*`, `siem-linux-*`) – ermöglicht
  RBAC nach dem Need-to-Know Prinzip

---

## 🛠️ Tech Stack

| Komponente | Version | Zweck |
|------------|---------|-------|
| Elasticsearch | 9.4.2 | Datenspeicher & Suchengine |
| Kibana | 9.4.2 | Visualisierung & Dashboard |
| Logstash | 9.4.2 | Log-Verarbeitung (ETL) |
| Filebeat | 9.4.2 | Log-Agent für Linux/Docker |
| Winlogbeat | 9.4.2 | Log-Agent für Windows |
| Docker/Compose | latest | Container-Orchestrierung |
| Nginx Proxy Manager | latest | Reverse Proxy & SSL |

---

## 🔒 Security Entscheidungen

### 1. Netzwerk-Isolation
Elasticsearch ist **nicht** von aussen erreichbar – nur intern im Docker Netzwerk.
Einziger Einstiegspunkt ist Kibana via Nginx Proxy Manager.

```
Internet → NPM (HTTPS) → Kibana → Elasticsearch (intern)
```

### 2. Secrets Management
Alle Passwörter sind in einer `.env` Datei gespeichert die **niemals** in Git
eingecheckt wird. Die `.env.example` zeigt welche Variablen benötigt werden.

### 3. Read-Only Volumes
Filebeat mountet Host-Verzeichnisse mit `:ro` (read-only) – der Container
kann Logs lesen aber **nichts schreiben**.

### 4. RBAC-fähige Index-Struktur
Separate Indizes pro Betriebssystem ermöglichen granulare Zugriffsrechte:
- SOC Analyst Linux → Zugriff nur auf `siem-linux-*`
- SOC Analyst Windows → Zugriff nur auf `siem-windows-*`

---

## 🚀 Setup Anleitung

### Voraussetzungen
- Docker & Docker Compose installiert
- Nginx Proxy Manager läuft
- Domain mit DNS-Einträgen konfiguriert

### 1. Repository klonen
```bash
git clone https://github.com/deinusername/elk-siem-lab.git
cd elk-siem-lab
```

### 2. Umgebungsvariablen konfigurieren
```bash
cp .env.example .env
nano .env  # Passwörter anpassen!
```

### 3. Docker Netzwerk erstellen
```bash
docker network create elk-network
```

### 4. Stack starten
```bash
docker compose up -d
```

### 5. kibana_system Passwort setzen
```bash
docker exec -it elasticsearch elasticsearch-reset-password -u kibana_system -i
```

### 6. Nginx Proxy Manager konfigurieren
- Domain: `kibana.deinedomain.com`
- Forward: `http://kibana:5601`
- SSL: Let's Encrypt aktivieren
- Websockets: aktivieren

### 7. Winlogbeat auf Windows installieren
```powershell
# Als Administrator ausführen
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-9.4.2-windows-x86_64.zip" -OutFile "C:\winlogbeat.zip"
Expand-Archive -Path "C:\winlogbeat.zip" -DestinationPath "C:\Program Files\Winlogbeat"
cd "C:\Program Files\Winlogbeat\winlogbeat-9.4.2-windows-x86_64"
.\install-service-winlogbeat.ps1
Start-Service winlogbeat
```

---

## 📚 Learnings

### Technisch
- **Docker Networking** – Container kommunizieren über dedizierte Netzwerke,
  nicht über Host-IPs
- **Index Lifecycle Management** – Tägliche Index-Rotation (`siem-windows-2026.06.16`)
  ermöglicht einfaches Daten-Management und Retention Policies
- **ETL Pipeline** – Logstash transformiert rohe Logs in strukturierte,
  durchsuchbare Felder
- **Beats Familie** – Filebeat, Winlogbeat und weitere Agents für verschiedene
  Log-Quellen

### Security
- **Security by Design** – Ports nur öffnen wenn zwingend nötig
- **Need-to-Know Prinzip** – Separate Indizes ermöglichen granulare Zugriffsrechte
- **Secrets Management** – `.env` Dateien niemals in Git einchecken
- **Read-Only Mounts** – Minimale Berechtigungen für Container

### SOC-relevante Windows Event IDs
| Event ID | Beschreibung | SOC Relevanz |
|----------|-------------|--------------|
| 4624 | Successful Logon | Baseline, Anomalien erkennen |
| 4625 | Failed Logon | Brute-Force Erkennung |
| 4672 | Special Privileges | Privilege Escalation |
| 7045 | New Service installed | Persistence Detection |

---

## 📋 Roadmap / TODOs

- [ ] Passwörter via Docker Secrets statt `.env` (Production-ready)
- [ ] Alert-Regeln für kritische Event IDs (4625 Brute-Force)
- [ ] GeoIP Enrichment in Logstash (IP → Land/Stadt)
- [ ] Retention Policy via ILM (Logs älter als 90 Tage löschen)
- [ ] RBAC Konfiguration in Kibana (separate Rollen pro Analyst)
- [ ] Metricbeat für System-Metriken (CPU, RAM, Netzwerk)

---

## 📖 Referenzen

- [Elastic Documentation](https://www.elastic.co/docs)
- [ICT Minimalstandard Schweiz](https://www.bwl.admin.ch/bwl/de/home/themen/ikt/ikt_minimalstandard.html)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
