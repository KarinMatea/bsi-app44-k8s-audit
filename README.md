# BSI APP.4.4 (Kubernetes) Compliance-Check für k3s

Ansible-Rolle, die einen laufenden k3s-Cluster read-only gegen die 21
Anforderungen des BSI IT-Grundschutz-Bausteins **APP.4.4 Kubernetes** (Edition
2023) prüft und einen Markdown-Report ausgibt.

**Scope: nur Abfrage.** Es wird nichts am Cluster verändert. Umsetzung/Remediation
ist bewusst nicht Teil dieses Projekts (kommt später).

## Automatisiert geprüft werden aktuell:

| ID | Anforderung | Was geprüft wird |
|---|---|---|
| A1 | Separierung der Anwendungen | Workloads im `default`-Namespace |
| A3 | Identitäts-/Berechtigungsmanagement | `cluster-admin` an anonyme/breite Gruppen gebunden |
| A4 | Separierung von Pods | `hostPID`/`hostNetwork`/`hostIPC`, privilegierte Container |
| A7 | Separierung der Netze | NetworkPolicy-Abdeckung je Namespace (inkl. Flannel-Warnhinweis) |
| A9 | Service-Accounts | Nutzung des `default`-SA, `automountServiceAccountToken` |
| A11 | Überwachung der Container | fehlende `livenessProbe`/`readinessProbe` |
| A13 | Automatisierte Auditierung | Policy-Engine (Kyverno/Gatekeeper/Kubewarden) vorhanden? |
| A14 | Dedizierte Nodes | Node-Rollen-Labels (bei <3 Nodes als N/A bewertet) |
| A18 | Mikro-Segmentierung | Grobcheck: Namespaces mit mehreren Pods ohne jede NetworkPolicy |
| A21 | Regelmäßiger Restart | Pod-Alter gegen konfigurierbaren Schwellwert (Default 24h) |

Alle anderen (A2, A5, A6, A8, A10, A12, A15, A16, A17, A19, A20) sind als
`automatable: false` markiert und erscheinen als MANUAL.

Was die Role nicht kann:

Von den 21 Anforderungen sind viele organisatorisch/prozessual (Planung,
Backup-Konzept, Betriebsdokumentation, Hochverfügbarkeit über
Brandabschnitte, Node-Attestierung per TPM, ...). Die lassen sich nicht per
API-Query beantworten. Diese Anforderungen tauchen im Report als **🟡 MANUAL**
auf – bewusst, damit der Report nicht etwas als "erfüllt" ausweist, das nur
technisch nicht widerlegt werden konnte.

**Wichtiger Hinweis zu k3s:** k3s nutzt standardmäßig **Flannel** als CNI.
Flannel setzt NetworkPolicies **nicht durch** – selbst wenn A7/A18 technisch
"OK" melden, greifen die Regeln nur, wenn du zusätzlich ein policy-fähiges CNI
(z. B. Calico, Cilium) im Einsatz hast. Das steht auch als Hinweis im Report.

## Neu hier? Erst hier lesen

Für eine ausführliche Einführung (was ist k3s, warum k3s statt "großem"
Kubernetes, was macht jeder einzelne Schritt) siehe
**[GETTING-STARTED.md](./GETTING-STARTED.md)**. Der Rest dieser README ist
die Kurzfassung für alle, die die Konzepte schon kennen.

## Voraussetzungen

Nur eine SSH-/lokale Shell-Verbindung zur VM und Ansible auf dem Steuerrechner
(das kann dieselbe VM sein, siehe `inventory/hosts.ini`). Alles andere -
Basis-Pakete, k3s, Python-/Ansible-Abhängigkeiten - installiert `bootstrap.yml`.

## Schritt 1: VM bootstrappen (einmalig, bei leerer VM)

```bash
# Ansible selbst muss vorhanden sein, bevor Ansible etwas installieren kann:
sudo apt update && sudo apt install -y ansible

cd bsi-app44-k8s-audit
ansible-playbook bootstrap.yml --ask-become-pass
```

Das installiert:
- Basis-Pakete (curl, git, vim, htop, Python3 + pip, ...)
- k3s (Single-Node-Kubernetes) über den offiziellen Installer
- kubeconfig unter `~/.kube/config` für deinen Benutzer
- `kubernetes`-Python-Paket + die `kubernetes.core`-Ansible-Collection

Am Ende zeigt es `kubectl get nodes` zur Kontrolle. Wenn dein Node dort als
`Ready` auftaucht, war's erfolgreich.

**Idempotent:** Ist bereits k3s installiert (`/usr/local/bin/k3s` existiert),
überspringt die Rolle die Installation und führt nur die restlichen Schritte
(kubeconfig, Python-Pakete, Collection) erneut aus - du kannst `bootstrap.yml`
also gefahrlos mehrfach laufen lassen.

## Schritt 2: Compliance-Check laufen lassen

```bash
ansible-playbook playbook.yml
```

Braucht kein `sudo` mehr - der Check liest nur über die Kubernetes-API, die
kubeconfig aus Schritt 1 wird automatisch gefunden (`~/.kube/config`).

Der Report landet in `reports/bsi-app44-report_<timestamp>.md`.

Eigene Werte anpassen (z. B. andere ausgeschlossene Namespaces, anderer
Cluster-Name) über `-e`:

```bash
ansible-playbook playbook.yml -e bsi_cluster_label=homelab-k3s -e bsi_excluded_namespaces='["kube-system","kube-public","kube-node-lease","kyverno"]'
```

## Projektstruktur

```
.
├── ansible.cfg
├── inventory/hosts.ini
├── bootstrap.yml            # Schritt 1: leere VM -> k3s + Tools
├── playbook.yml             # Schritt 2: Compliance-Check (read-only)
├── requirements.yml         # Ansible Collections
├── requirements.txt         # Python-Abhängigkeiten
├── reports/                 # generierte Reports (gitignored)
├── roles/bsi_bootstrap/
│   ├── defaults/main.yml
│   └── tasks/main.yml       # Pakete, k3s-Install, kubeconfig, pip, Collection
└── roles/bsi_app44_check/
    ├── defaults/main.yml
    ├── vars/requirements_catalog.yml   # die 21 Anforderungen als Daten
    ├── tasks/
    │   ├── main.yml
    │   ├── 01_gather_cluster_state.yml
    │   ├── 02_check_*.yml              # ein Check pro automatisierbarer Anforderung
    │   └── 03_generate_report.yml
    └── templates/report.md.j2
```

## Nächste Schritte (nicht Teil dieses Projekts)

- Remediation/Umsetzung der Findings (eigene Rolle/Playbook)
- Tiefere RBAC-Analyse (z. B. `rbac-tool`, `kubectl-who-can`)
- Etcd/Datastore-Verschlüsselung prüfen (A20) – erfordert Node-Zugriff, nicht
  nur API-Zugriff
