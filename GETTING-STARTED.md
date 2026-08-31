# Erste Schritte: Von der leeren VM zum BSI-Compliance-Report

Diese Anleitung richtet sich an dich, wenn du k3s/Kubernetes noch nicht
im Detail kennst und verstehen willst, was bei jedem Schritt passiert -
nicht nur Befehle copy-pasten willst.

## 1. Einleitung: Was ist k3s und warum nutzen wir es?

**Kubernetes** ist die Software, die dafür sorgt, dass Container (z. B.
Docker-Container) auf mehreren Maschinen automatisch gestartet, überwacht,
neu gestartet und vernetzt werden. Man nennt das "Orchestrierung". Ein
Kubernetes-**Cluster** besteht normalerweise aus mehreren Rollen:

- **Control Plane**: das "Gehirn" - entscheidet, welcher Container wo
  läuft, speichert den gewünschten Zustand, stellt die API bereit, über
  die du mit `kubectl` sprichst.
- **Nodes (Worker)**: die Maschinen, auf denen die eigentlichen Container
  laufen.

Normales Kubernetes ("vanilla k8s", oft über ein Tool namens `kubeadm`
installiert) ist recht schwergewichtig: viele Einzelkomponenten, hoher
Speicherbedarf, aufwendige Installation. Für ein Homelab mit einer VM ist
das Overkill.

**k3s** (von Rancher/SUSE entwickelt) ist eine **abgespeckte, aber voll
kompatible** Kubernetes-Distribution:

- Läuft als **ein einziges Binary** (~70 MB), keine 10 Einzeldienste.
- Nutzt standardmäßig **SQLite** statt des schwergewichtigen `etcd` als
  Datenspeicher (bei einem Single-Node-Setup wie unserem völlig
  ausreichend).
- Ein einzelner Server kann gleichzeitig **Control Plane UND Node** sein -
  genau das, was wir mit einer VM wollen.
- 100% API-kompatibel zu "echtem" Kubernetes: alles, was du hier lernst
  (Namespaces, Pods, NetworkPolicies, RBAC, Kyverno, ...), funktioniert
  1:1 auch später auf einem "großen" Cluster oder in deinem
  OpenShift-Umfeld.
- Bringt **containerd** (die Container-Runtime) und ein einfaches CNI
  (**Flannel**, siehe Hinweis weiter unten) direkt mit - du musst Docker
  nicht separat installieren.

Kurz: k3s ist der Grund, warum du mit einer einzigen, kleinen VM überhaupt
einen "echten" Kubernetes-Cluster zum Testen und Lernen bekommst.

### Was macht `bootstrap.yml` konkret?

Das Playbook läuft **einmalig** und in dieser Reihenfolge:

1. **System-Pakete installieren** (`apt`/`dnf`): curl, git, Python3+pip
   usw. - die Grundausstattung, die auf einer leeren VM fehlt.
2. **k3s installieren**: lädt das offizielle Install-Script von
   `get.k3s.io` herunter und führt es aus. Das Script erkennt deine
   CPU-Architektur, lädt das passende k3s-Binary, legt einen systemd-Dienst
   `k3s.service` an und startet ihn.
3. **Warten, bis der Node bereit ist**: k3s braucht ein paar Sekunden, bis
   die Control Plane hochgefahren ist. Das Playbook fragt in einer
   Schleife `kubectl get nodes` ab, bis der Status `Ready` erscheint.
4. **kubeconfig einrichten**: k3s legt seine Zugangsdaten unter
   `/etc/rancher/k3s/k3s.yaml` ab (nur für root lesbar). Wir kopieren die
   Datei nach `~/.kube/config` für deinen Benutzer, damit du (und Ansible)
   ohne `sudo` mit dem Cluster sprechen können.
5. **Python- und Ansible-Abhängigkeiten**: installiert das
   `kubernetes`-Python-Paket (das braucht die `kubernetes.core`
   Ansible-Collection intern) und die Collection selbst.
6. **Funktionstest**: ruft `kubectl get nodes` auf und zeigt dir das
   Ergebnis - siehst du deinen Node mit Status `Ready`, hat alles
   funktioniert.

**Wichtiger Hinweis zu Flannel:** k3s bringt standardmäßig Flannel als
CNI (Netzwerk-Plugin) mit. Flannel ist einfach und zuverlässig, setzt
aber **keine NetworkPolicies durch**. Das heißt: du kannst NetworkPolicy-
Objekte anlegen (und unser Compliance-Check prüft auch darauf), aber sie
haben ohne ein policy-fähiges CNI (z. B. Calico) technisch **keine
Wirkung**. Für den Einstieg lassen wir das bewusst so - der
Compliance-Report weist an den passenden Stellen darauf hin.

## 2. Voraussetzungen

- Eine VM mit Debian/Ubuntu (oder RHEL/Fedora-Familie) - laut deiner
  Angabe ist sie aktuell komplett leer.
- SSH-Zugang oder direkter Shell-Zugriff mit einem Benutzer, der `sudo`
  nutzen darf.
- Internetzugang von der VM aus (für apt/dnf-Pakete, das k3s-Install-
  Script und die Ansible Galaxy Collection).

## 3. Ansible auf der VM installieren

Falls Ansible selbst noch nicht da ist (wahrscheinlich, VM ist ja leer):

```bash
sudo apt update
sudo apt install -y ansible
```

Prüfen, ob es geklappt hat:

```bash
ansible --version
```

## 4. Bootstrap ausführen (k3s + Tools installieren)

```bash
ansible-playbook bootstrap.yml --ask-become-pass
```

Du wirst nach deinem `sudo`-Passwort gefragt (daher `--ask-become-pass`).
Das dauert je nach Internetverbindung 1-3 Minuten. Am Ende siehst du eine
Ausgabe wie:

```
TASK [bsi_bootstrap : Ergebnis des Bootstraps ausgeben] ***
ok: [localhost] => {
    "msg": [
        "k3s Node-Status:",
        ["deine-vm   Ready    control-plane,master   45s   v1.30.x+k3s1"]
    ]
}
```

Steht dort `Ready`, ist dein Cluster einsatzbereit.

### Manuell nachprüfen (optional, aber empfehlenswert)

```bash
kubectl get nodes
kubectl get pods -A
```

`kubectl get pods -A` sollte dir ein paar Pods im Namespace `kube-system`
zeigen (CoreDNS, Traefik als Ingress-Controller, das lokale
Storage-Provisioning) - das sind die mitgelieferten k3s-Basiskomponenten,
kein Grund zur Sorge.

### Falls etwas schiefgeht

- **"Ready" erscheint nie / Timeout**: `sudo systemctl status k3s` und
  `sudo journalctl -u k3s -f` zeigen dir die Logs des k3s-Dienstes.
- **kubectl: command not found**: das k3s-Installer legt standardmäßig
  auch einen Symlink `/usr/local/bin/kubectl` an. Falls der fehlt, prüfe
  `which k3s` und nutze übergangsweise `k3s kubectl ...` statt `kubectl ...`.
- **Permission denied bei ~/.kube/config**: prüfe mit `ls -la ~/.kube/`,
  ob die Datei dir gehört (`whoami` vs. Owner-Spalte). Notfalls manuell
  fixen: `sudo chown $(whoami):$(whoami) ~/.kube/config`.

## 5. Compliance-Check laufen lassen

Jetzt, wo k3s läuft, kannst du den eigentlichen BSI-APP.4.4-Check
ausführen - **ohne `sudo`**, da er nur lesend über die Kubernetes-API
arbeitet:

```bash
ansible-playbook playbook.yml
```

Das Ergebnis liegt danach unter `reports/bsi-app44-report_<timestamp>.md`.

## 6. Report lesen

Öffne die generierte `.md`-Datei. Sie ist in drei Abschnitte gegliedert
(Basis-/Standard-/Anforderungen bei erhöhtem Schutzbedarf), jede
Anforderung hat einen Status:

- ✅ **OK** - technisch unauffällig (mit Einschränkungen, die im Text
  jeweils genannt werden)
- ❌ **FINDING** - konkreter technischer Verstoß gefunden
- 🟡 **MANUAL** - lässt sich nicht automatisiert bewerten (organisatorisch,
  oder Grobcheck reicht nicht aus)
- ⚪ **NOT_APPLICABLE** - Anforderung passt nicht auf dieses Setup (z. B.
  "dedizierte Nodes" bei einem Single-Node-Cluster)

## 7. Als Git-Projekt sichern

```bash
git init
git add .
git commit -m "Initial commit: k3s bootstrap + BSI APP.4.4 compliance check"
```

Remote auf GitHub anlegen (Web-UI) und dann:

```bash
git remote add origin https://github.com/<dein-user>/<dein-repo>.git
git branch -M main
git push -u origin main
```
