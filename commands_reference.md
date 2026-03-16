# 🛠️ DevOps Kommando-Referenz — Roche Interview Training

> Alle Kommandos die wir praktisch durchgeführt haben, mit Kontext und Erklärung.

---

## 1. Terraform (IPAPA Workflow)

| Kommando | Kontext | Warum | Wie anwenden |
|----------|---------|-------|-------------|
| `terraform init` | Projekt starten | Lädt Provider-Plugins herunter (z.B. `hashicorp/kubernetes`). Wie "Motor starten". | Im Ordner mit `main.tf` ausführen. Muss nach jeder Provider-Änderung wiederholt werden. |
| `terraform plan` | Vorschau anzeigen | Zeigt was passieren wird: `+` = erstellen, `~` = ändern, `-` = löschen. Nichts wird verändert! | Vor jedem `apply` ausführen. In CI/CD: Team reviewed den Plan vor dem Merge. |
| `terraform apply` | Infrastruktur erstellen | Führt den Plan in der echten Welt aus. Fragt `yes` zur Bestätigung. | Nach dem Review des Plans. Erstellt/ändert echte Ressourcen im Cluster. |
| `terraform destroy` | Alles aufräumen | Löscht alle Ressourcen die Terraform verwaltet. | Nur in Dev/Test! Niemals in Prod ohne Absprache. |

### Terraform Dateien

| Datei | Zweck | Beispiel |
|-------|-------|---------|
| `main.tf` | Provider + Resources definieren | `provider "kubernetes" { config_path = "~/.kube/config" }` |
| `terraform.tfstate` | Terraforms Gedächtnis (State) | Speichert welche Ressourcen existieren. Wenn gelöscht → Terraform vergisst alles! |
| `.terraform/` | Heruntergeladene Plugins | Wird von `terraform init` erstellt. |
| `.terraform.lock.hcl` | Versionssperre für Plugins | Sichert, dass das Team dieselbe Plugin-Version nutzt. |

---

## 2. Kubernetes — kubectl (DELD Workflow)

### Pod Lifecycle & Debugging

| Kommando | Kontext | Warum | Wie anwenden |
|----------|---------|-------|-------------|
| `kubectl apply -f <datei.yaml>` | YAML in den Cluster laden | Erstellt oder aktualisiert Ressourcen deklarativ. K8s vergleicht mit dem aktuellen Zustand. | `kubectl apply -f broken-pod.yaml` |
| `kubectl get pods -n <namespace>` | Status aller Pods anzeigen | Schneller Überblick: Läuft der Pod? Wie viele Restarts? | `kubectl get pods -n roche-dev` |
| `kubectl get all -n <namespace>` | Alle Ressourcen anzeigen | Zeigt Pods, Services, Deployments, ReplicaSets auf einen Blick. | `kubectl get all -n roche-dev` |
| `kubectl get namespaces` | Alle Namespaces auflisten | Überprüfen ob ein Namespace existiert (z.B. nach Terraform Apply). | `kubectl get namespaces` |
| `kubectl describe pod <name> -n <ns>` | **D**escribe (DELD Schritt 1) | Zeigt detaillierte Infos + Events. Immer der ERSTE Schritt beim Debugging! | `kubectl describe pod roche-dev-broken-pod -n roche-dev` |
| `kubectl logs <pod> -n <ns>` | **L**ogs (DELD Schritt 3) | Liest die App-Logs des Containers. Mit `--previous` die Logs des letzten Crashs sehen. | `kubectl logs <pod-name> -n roche-dev --previous` |
| `kubectl delete pod <name> -n <ns>` | **D**eploy Fix (DELD Schritt 4) | Pods sind immutable → oft muss man löschen und neu erstellen. | `kubectl delete pod roche-dev-broken-pod -n roche-dev` |

### Häufige Pod-Status und ihre Bedeutung

| Status | Bedeutung | Lösung |
|--------|-----------|--------|
| `Running` ✅ | Pod läuft normal | Alles gut! |
| `ImagePullBackOff` 🔴 | Image nicht gefunden | Image-Name/Tag in der YAML prüfen (Schreibfehler?) |
| `CrashLoopBackOff` 🔴 | Container startet und crasht sofort | `kubectl logs --previous` lesen. Oft: Port-Problem oder fehlende Config. |
| `Pending` 🟡 | Pod wartet auf Scheduling | Nicht genug CPU/Memory oder Node nicht bereit. |
| `CreateContainerConfigError` 🔴 | Konfigurationsfehler | SecurityContext oder ConfigMap/Secret fehlt. |

---

## 3. Kubernetes RBAC (Role-Based Access Control)

### Der Dreiklang (Wer - Was - Wo)

| Konzept | Bedeutung | Kubernetes Objekt |
|---------|-----------|-------------------|
| **Wer** | Die Identität (User, Pod, CI/CD Pipeline) | `ServiceAccount` |
| **Was** | Die Berechtigungen (z.B. Pods löschen) | `Role` (Namespace) oder `ClusterRole` (Global) |
| **Wo** | Die Zuordnung von "Wer" zu "Was" | `RoleBinding` oder `ClusterRoleBinding` |

### RBAC Kommandos

| Kommando | Kontext | Wie anwenden |
|----------|---------|-------------|
| `kubectl apply -f rbac/` | Alle RBAC YAMLs aus einem Ordner anwenden | Erstellt SA, Role und RoleBinding auf einen Schlag |
| `kubectl get rolebinding <name> -n <ns>` | Überprüfen der Zuordnung | `kubectl get rolebinding roche-app -n roche-dev` |
| `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa_name>` | Testen! Prüft ob ein SA etwas darf | `kubectl auth can-i delete pods --as=system:serviceaccount:roche-dev:roche-app` |

---

## 3. Kubernetes Security (RACE Prinzip)

### Security Context — Zwei Ebenen

| Ebene | Wo in der YAML | Was gehört hierhin |
|-------|----------------|-------------------|
| **Pod-Level** (`spec.securityContext`) | Unter `spec:`, vor `containers:` | `fsGroup`, `runAsUser`, `runAsGroup`, `runAsNonRoot` |
| **Container-Level** (`spec.containers[].securityContext`) | Unter dem jeweiligen Container | `allowPrivilegeEscalation`, `capabilities`, `readOnlyRootFilesystem` |

### RACE Einstellungen

| Buchstabe | Setting | Wert | Warum |
|-----------|---------|------|-------|
| **R** | `runAsNonRoot` | `true` | Container darf nie als Root (UID 0) laufen |
| **A** | `allowPrivilegeEscalation` | `false` | Kein "Sudo" möglich |
| **C** | `capabilities: drop` | `- ALL` | Alle Linux-Kernel-Rechte entzogen |
| **E** | `readOnlyRootFilesystem` | `true` | Angreifer können keine Dateien schreiben |

### User/Group IDs

| Setting | Wert | Bedeutung |
|---------|------|-----------|
| `runAsUser: 1000` | UID 1000 | Normaler User, kein Root. Darf keine Ports < 1024 öffnen. |
| `runAsGroup: 3000` | GID 3000 | Neue Dateien gehören dieser Gruppe. |
| `fsGroup: 2000` | GID 2000 | Externe Volumes werden für diese Gruppe freigeschaltet. Nur auf Pod-Ebene! |

---

## 4. Helm (Paketmanager für K8s)

### Helm Kommandos

| Kommando | Kontext | Warum | Wie anwenden |
|----------|---------|-------|-------------|
| `helm create <name>` | Chart-Gerüst erstellen | Erzeugt die komplette Ordnerstruktur mit Beispiel-Templates. | `helm create roche-app` |
| `helm template <path>` | Vorschau rendern | Zeigt das fertige YAML ohne es zu deployen. Wie `terraform plan`. | `helm template ./roche-app` |
| `helm install <name> <path> -n <ns>` | Erstmalig deployen | Erstellt alle Ressourcen im Cluster. Wie `terraform apply`. | `helm install roche-app ./roche-app -n roche-dev` |
| `helm upgrade <name> <path> -n <ns>` | Update ausrollen | Ändert ein bestehendes Release. Revision wird erhöht. | `helm upgrade roche-app ./roche-app -n roche-dev` |
| `helm uninstall <name> -n <ns>` | Alles aufräumen | Löscht alle Ressourcen des Releases. | `helm uninstall roche-app -n roche-dev` |
| `helm list -n <ns>` | Installierte Releases anzeigen | Überblick über alle Helm-Deployments. | `helm list -n roche-dev` |

### Helm Chart Struktur (Wer-Was-Wie)

| Datei | Frage | Zweck |
|-------|-------|-------|
| `Chart.yaml` | **Wer** bin ich? | Name, Version, App-Version des Charts |
| `values.yaml` | **Was** will ich konfigurieren? | Image, Ports, Replicas, Security — die **Fernbedienung** |
| `templates/` | **Wie** baue ich es zusammen? | YAML-Vorlagen mit `{{ .Values.xxx }}` Variablen |

### 🧠 Interview-Bonus: Helm vs. Terraform Analogie

| Konzept | 🚢 Helm (Applikationen) | 🏗️ Terraform (Infrastruktur) |
|---------|-------------------------|------------------------------|
| **Was es ist** | Paketmanager für Kubernetes | Infrastructure as Code |
| **Die Code-Basis** | `templates/*.yaml` | `*.tf` Dateien |
| **Die Variablen** | `values.yaml` (Fernbedienung) | `variables.tf` / `terraform.tfvars` |
| **Der Speicher (Gedächtnis)** | `Secret` im K8s Cluster (Release) | `terraform.tfstate` |
| **1. Bauplan generieren** | `helm create <name>` | `terraform init` (plus `main.tf` anlegen) |
| **2. Vorschau (Dry-Run)** | `helm template` | `terraform plan` |
| **3. Erstes Ausrollen** | `helm install` | `terraform apply` |
| **4. Updates ausrollen** | `helm upgrade` | `terraform apply` |
| **5. Alles löschen** | `helm uninstall` | `terraform destroy` |

---

## 5. Observability Stack (PLG)

| Tool | Rolle | Was es sammelt |
|------|-------|---------------|
| **P**rometheus | Metriken sammeln + Alerting | CPU, Memory, Request Count, Error Rate |
| **L**oki | Logs aggregieren | Application Logs, Container Logs |
| **G**rafana | Visualisierung | Dashboards für Prometheus-Metriken + Loki-Logs |
| AlertManager | Benachrichtigungen | Feuert Alerts an PagerDuty, Slack, E-Mail |

---

## 6. Alle Mnemonics auf einen Blick

| Mnemonic | Bedeutung | Kontext |
|----------|-----------|---------|
| **IPAPA** | Init, Plan, Apply, Push, Automate | Terraform Workflow |
| **DELD** | Describe, Events, Logs, Deploy fix | K8s Pod Troubleshooting |
| **RACE** | RunAsNonRoot, AllowPrivEsc, Capabilities, readOnlyFS | K8s Container Security |
| **PLG** | Prometheus, Loki, Grafana | Observability Stack |
| **A-K-R** | Automation, Kubernetes, Reliability | Deine 3 Kernbotschaften |

---

## 7. Interview-Bonus: Terraform vs. Ansible

| Feature | 🏗️ Terraform | 🤖 Ansible |
|---------|--------------|------------|
| **Hauptzweck** | Infrastructure Provisioning (Server, K8s, Netzwerke bauen) | Configuration Management (Software auf Servern installieren/konfigurieren) |
| **Paradigma** | **Deklarativ** ("Ich will 3 Server haben") | **Prozedural** ("Installiere Nginx, dann starte den Service") |
| **Gedächtnis** | Ja (Hat ein `terraform.tfstate` File. Weiß exakt was existiert) | Nein (Kein State-File. Führt Playbooks einfach nacheinander aus) |
| **Lifecycle** | Kann Ressourcen erstellen, ändern UND **löschen** (`destroy`) | Gut im Erstellen/Ändern, schlecht im verlässlichen Löschen |
| **Interview-Fazit** | "Damit baue ich den leeren Kubernetes-Cluster in der Cloud." | "Damit richte ich die Linux-VMs ein, falls wir kein K8s hätten." |

---

## 8. Docker & CI/CD (GitHub Actions)

### Docker Best Practices (Interview-Wissen)
Wenn das Image zu groß oder unsicher ist, nennst du diese 3 Punkte:
1. **Multi-Stage Builds**: Im `Dockerfile` nutzt du ein großes Image (z.B. Maven/Node) nur zum Kompilieren (bauen) des Codes. Das fertige Programm (Binary) kopierst du dann in ein zweites, winziges Image. Das spart extrem viel Platz und löscht alle Build-Tools aus dem finalen Image.
2. **Alpine / Distroless Images**: Nutze als Basis extrem kleine Linux-Versionen (wie `alpine`), die auf das absolute Minimum reduziert sind. Weniger Tools = weniger Angriffsfläche für Hacker.
3. **.dockerignore**: Verhindert, dass unnötige Dateien (wie `.git` oder lokale Logs) überhaupt in das Image hochgeladen werden.

### Der CI/CD Workflow (z.B. GitHub Actions)
Eine Pipeline automatisiert den Weg vom Code-Commit bis zum Deployment.

| Phase | Was passiert hier? | Tools |
|-------|--------------------|-------|
| **1. Lint/Test** | Code wird geprüft (Syntax-Fehler, Security-Scans) und Unit-Tests laufen. | SonarQube, Jest, PyTest |
| **2. Build** | `docker build` wird ausgeführt (Multi-Stage). Aus dem Code wird ein Image. | Docker |
| **3. Push** | `docker push` schiebt das Image sicher in eine Registry. | Docker Hub, AWS ECR, Harbor |
| **4. Deploy (CD)** | Hier übernimmt GitOps (ArgoCD zieht das Image) oder die Pipeline führt `helm upgrade` aus. | ArgoCD, Helm |
