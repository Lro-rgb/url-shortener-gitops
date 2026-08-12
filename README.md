# url-shortener-gitops

GitOps-Repo für die Kubernetes-Manifeste des [url-shortener](https://github.com/Lro-rgb/url-shortener).
Hier liegt der gewünschte Cluster-Zustand als YAML — Git ist die einzige Wahrheit, Änderungen laufen über Commits statt manueller `kubectl`-Befehle.

## Inhalt

| Datei | Beschreibung |
|-------|--------------|
| `configmap.yaml` | DB-Init-SQL (Tabelle `urls`) für MariaDB |
| `db.yaml` | MariaDB Deployment + Service |
| `keeper.yaml` | keeper Deployment + Service |
| `shorty.yaml` | shorty Deployment + Service (intern, ClusterIP) |
| `ingress.yaml` | Ingress, einziger Eingang von aussen auf shorty |
| `argocd/application.yaml` | ArgoCD Application, die dieses Repo mit dem Cluster verbindet |

Die Secrets (DB-Passwörter, API-Key) liegen **nicht** im Git, sondern werden als
Kubernetes-Secret aus der `.env` des App-Repos erzeugt.

## Deploy in Minikube

```bash
# cluster starten und ingress-addon aktivieren
minikube start --memory=4096 --cpus=2
minikube addons enable ingress

# images im app-repo bauen (in den docker von minikube)
minikube image build -t keeper:local ../url-shortener/keeper
minikube image build -t shorty:local ../url-shortener/shorty

# secret aus der .env des app-repos (kommt nicht ins git)
kubectl create secret generic url-shortener-secret --from-env-file=../url-shortener/.env

# manifeste anwenden
kubectl apply -f .
kubectl get pods -w
```

### Zugriff über den Ingress

Unter Windows/Mac (Docker-Treiber) braucht der Ingress einen Tunnel. Das Fenster
offen lassen:

```bash
minikube tunnel
```

Test in einem zweiten Terminal (Host-Header statt hosts-Datei):

```bash
curl.exe -X POST http://127.0.0.1/shorten -H "Host: shorty.local" -H "content-type: application/json" -d '{\"longUrl\":\"https://duckduckgo.com/\"}'
```

Alternativ ohne Ingress (Fallback): `kubectl port-forward service/shorty 3000:3000`
und gegen `http://127.0.0.1:3000` testen.

## ArgoCD

ArgoCD beobachtet dieses Repo und gleicht den Cluster automatisch mit den Manifesten ab.

```bash
# argocd installieren
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ui erreichbar machen (fenster offen lassen)
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

UI unter https://127.0.0.1:8081, Login `admin`, Passwort:

```powershell
$pw = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pw))
```

Da dieses Repo privat ist, muss es in der UI unter Settings -> Repositories mit
Zugangsdaten (GitLab Access Token, read_repository) verbunden werden. Sonst bleibt
ArgoCD im Status Unknown hängen.

Danach die Application anwenden:

```bash
kubectl apply -f argocd/application.yaml
```

Wichtig: Das Secret `url-shortener-secret` und die `:local`-Images liegen weiter
ausserhalb von Git (siehe oben) und müssen vorhanden sein.

## Mögliche nächste Schritte

- Konkrete, versionierte Image-Tags aus der Container-Registry statt `:local`
- Eigener Namespace pro Umgebung (dev/staging/prod)
- Mehr Replicas und Argo CD für automatischen Sync
