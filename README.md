# url-shortener-gitops

GitOps-Repo für die Kubernetes-Manifeste des [url-shortener](https://gitlab.ict-learnfactory.ch/ict-learnfactory/210/durchfuehrungen/210-ae-ims-24/luis/url-shortener).
Hier liegt der gewünschte Cluster-Zustand als YAML — Git ist die einzige Wahrheit, Änderungen laufen über Commits statt manueller `kubectl`-Befehle.

## Inhalt

| Datei | Beschreibung |
|-------|--------------|
| `configmap.yaml` | DB-Init-SQL (Tabelle `urls`) für MariaDB |
| `db.yaml` | MariaDB Deployment + Service |
| `keeper.yaml` | keeper Deployment + Service |
| `shorty.yaml` | shorty Deployment + Service (NodePort) |

Die Secrets (DB-Passwörter, API-Key) liegen **nicht** im Git, sondern werden als
Kubernetes-Secret aus der `.env` des App-Repos erzeugt.

## Deploy in Minikube

```bash
# cluster starten
minikube start --memory=4096 --cpus=2

# images im app-repo bauen (in den docker von minikube)
minikube image build -t keeper:local ../url-shortener/keeper
minikube image build -t shorty:local ../url-shortener/shorty

# secret aus der .env des app-repos (kommt nicht ins git)
kubectl create secret generic url-shortener-secret --from-env-file=../url-shortener/.env

# manifeste anwenden
kubectl apply -f .
kubectl get pods -w

# shorty erreichbar machen (fenster offen lassen)
kubectl port-forward service/shorty 3000:3000
```

Test in einem zweiten Terminal:

```bash
curl.exe -X POST http://127.0.0.1:3000/shorten -H "content-type: application/json" -d '{\"longUrl\":\"https://duckduckgo.com/\"}'
```

## Mögliche nächste Schritte

- Konkrete, versionierte Image-Tags aus der Container-Registry statt `:local`
- Eigener Namespace pro Umgebung (dev/staging/prod)
- Mehr Replicas + Ingress statt NodePort für den produktiven Betrieb
