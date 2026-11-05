# TP 4.13 : Dépannage Kubernetes – 6 pannes à résoudre

**Objectif :** acquérir une **méthode** de diagnostic. En production, on passe plus de temps à comprendre pourquoi « ça ne marche pas » qu'à écrire du YAML.

**Durée :** 1h30 – **Prérequis :** TP 4.1 à 4.9

## La méthode

```
kubectl get pods            →  STATUS ? READY ? RESTARTS ?
kubectl describe pod <pod>  →  section "Events" en bas : la cause est presque toujours là
kubectl logs <pod> [--previous]  →  erreur de l'application (--previous : conteneur précédent, avant crash)
kubectl describe svc <svc>  →  ligne "Endpoints" : le Service trouve-t-il des pods ?
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it <pod> -- sh / kubectl run debug --image=busybox:1.36 -it --rm -- sh
```

| Symptôme | Causes fréquentes |
|---|---|
| `ImagePullBackOff` / `ErrImagePull` | tag inexistant, faute de frappe, registre privé sans `imagePullSecrets` |
| `CrashLoopBackOff` | l'application plante au démarrage : lire `logs --previous` |
| `Pending` | ressources insuffisantes, PVC non lié, quota |
| `Running` mais `0/1 READY` | readinessProbe en échec |
| Service sans réponse | `selector` ≠ labels des pods, mauvais `targetPort` |
| `CreateContainerConfigError` | ConfigMap ou Secret référencé inexistant |

---

Créez le namespace de travail :
```bash
kubectl create namespace depannage
kubectl config set-context --current --namespace=depannage
```

Pour chaque panne : appliquez le fichier, **diagnostiquez** (notez les commandes utilisées), **corrigez le YAML**, réappliquez, et remplissez le tableau *symptôme → commande → cause → correction*.

### Panne 1 – `panne1.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: front}
spec:
  replicas: 2
  selector: {matchLabels: {app: front}}
  template:
    metadata: {labels: {app: front}}
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpnie
```

### Panne 2 – `panne2.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: api}
spec:
  replicas: 1
  selector: {matchLabels: {app: api}}
  template:
    metadata: {labels: {app: api}}
    spec:
      containers:
        - name: api
          image: python:3.12-slim
          command: ["python", "-c", "import flask; print('demarrage')"]
```

### Panne 3 – `panne3.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web, tier: front}}
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
---
apiVersion: v1
kind: Service
metadata: {name: web}
spec:
  selector: {app: website}
  ports: [{port: 80, targetPort: 80}]
```
Test : `kubectl run t --image=busybox:1.36 -it --rm --restart=Never -- wget -qO- -T3 http://web`

### Panne 4 – `panne4.yaml`
```yaml
apiVersion: v1
kind: Service
metadata: {name: app}
spec:
  selector: {app: app}
  ports: [{port: 80, targetPort: 8080}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: app}
spec:
  replicas: 1
  selector: {matchLabels: {app: app}}
  template:
    metadata: {labels: {app: app}}
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
```
Même test que la panne 3 avec `http://app`.

### Panne 5 – `panne5.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: conf}
spec:
  replicas: 1
  selector: {matchLabels: {app: conf}}
  template:
    metadata: {labels: {app: conf}}
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          envFrom:
            - configMapRef: {name: config-app}
```

### Panne 6 – `panne6.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data}
spec:
  storageClassName: rapide
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
---
apiVersion: v1
kind: Pod
metadata: {name: db}
spec:
  containers:
    - name: redis
      image: redis:7.4-alpine
      volumeMounts: [{name: data, mountPath: /data}]
  volumes:
    - name: data
      persistentVolumeClaim: {claimName: data}
```

<details>
<summary>Corrections (à ne consulter qu'après avoir cherché)</summary>

| # | Symptôme | Commande clé | Cause | Correction |
|---|---|---|---|---|
| 1 | `ErrImagePull` / `ImagePullBackOff` | `describe pod` → *manifest unknown* | faute de frappe dans le tag | `nginx:1.27-alpine` |
| 2 | `CrashLoopBackOff` | `logs --previous` → `ModuleNotFoundError: flask` | dépendance absente de l'image (et le processus se termine) | utiliser une image contenant l'appli (ex. `legeni-webapp`) |
| 3 | Service sans réponse | `describe svc web` → `Endpoints:` vide | `selector: app: website` ≠ label `app: web` | `selector: {app: web}` |
| 4 | `connection refused` | `describe svc app` → `Endpoints: <IP>:8080` alors que nginx écoute sur 80 | nginx écoute sur 80 | `targetPort: 80` |
| 5 | `CreateContainerConfigError` | `describe pod` → *configmap "config-app" not found* | ConfigMap inexistante | créer la ConfigMap (ou `optional: true`) |
| 6 | Pod `Pending`, PVC `Pending` | `describe pvc` → *storageclass "rapide" not found* | StorageClass inexistante | `storageClassName: standard` (ou supprimer la ligne). Un PVC est immuable : le supprimer puis le recréer |
</details>

```bash
kubectl delete namespace depannage
kubectl config set-context --current --namespace=default
```

---

## ✅ Vérifications

- [ ] Les 6 pannes sont résolues : `kubectl get pods` tout `Running` et `READY`
- [ ] Le tableau de diagnostic est complet dans le compte-rendu

## 🎯 Défi en binôme

Chaque étudiant écrit **une panne originale** dans un manifest, l'échange avec son binôme, et chronomètre le temps de résolution.
