# TP 4.7 : Namespaces, ConfigMaps et Secrets – déployer l'application fil rouge

**Objectif :** déployer l'application fil rouge (web + Redis) dans un namespace dédié, en **sortant la configuration du code** (ConfigMap) et en protégeant les données sensibles (Secret).

**Durée :** 1h30 – **Prérequis :** TP 4.1 à 4.3, image `legeni-webapp` poussée sur Docker Hub (TP 2.9)

> Si vous n'avez pas poussé l'image, chargez-la directement dans Minikube : `minikube image load legeni-webapp:1.0.0`
> et utilisez `image: legeni-webapp:1.0.0` avec `imagePullPolicy: IfNotPresent`.

Créez un dossier `~/k8s-legeni/` : un fichier par ressource.

---

## Étape 1 : Le namespace

`00-namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: legeni-dev
  labels:
    env: dev
```
```bash
kubectl apply -f 00-namespace.yaml
kubectl config set-context --current --namespace=legeni-dev   # namespace par défaut de kubectl
```

## Étape 2 : La configuration (ConfigMap)

`10-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: legeni-dev
data:
  APP_ENV: "dev"
  APP_COLOR: "#1f6feb"
  REDIS_HOST: "redis"
  REDIS_PORT: "6379"
```

## Étape 3 : Le secret

On ne stocke **jamais** un secret en clair dans Git. On le crée en ligne de commande (ou avec Ansible Vault dans le projet) :
```bash
kubectl create secret generic redis-secret -n legeni-dev \
  --from-literal=REDIS_PASSWORD='S3cr3t-Redis!'
kubectl get secret redis-secret -o yaml           # la valeur est encodée en base64...
kubectl get secret redis-secret -o jsonpath='{.data.REDIS_PASSWORD}' | base64 -d; echo
```
Question : base64, est-ce du chiffrement ? Qui peut lire ce secret ?

## Étape 4 : Redis (Deployment + Service ClusterIP)

`20-redis.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: legeni-dev
  labels: {app: redis}
spec:
  replicas: 1
  selector:
    matchLabels: {app: redis}
  template:
    metadata:
      labels: {app: redis}
    spec:
      containers:
        - name: redis
          image: redis:7.4-alpine
          args: ["--requirepass", "$(REDIS_PASSWORD)"]
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef: {name: redis-secret, key: REDIS_PASSWORD}
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: legeni-dev
spec:
  type: ClusterIP          # accessible uniquement depuis l'intérieur du cluster
  selector: {app: redis}
  ports:
    - port: 6379
```


## Étape 5 : L'application web

`30-webapp.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: legeni-dev
  labels: {app: webapp}
spec:
  replicas: 2
  selector:
    matchLabels: {app: webapp}
  template:
    metadata:
      labels: {app: webapp}
    spec:
      containers:
        - name: webapp
          image: <votre_id_dockerhub>/legeni-webapp:1.0.0
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: webapp-config        # toutes les clés deviennent des variables d'env
            - secretRef:
                name: redis-secret         # REDIS_PASSWORD, lu par l'application
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: legeni-dev
spec:
  type: NodePort
  selector: {app: webapp}
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30080
```

## Étape 6 : Déployer et tester

```bash
kubectl apply -f .                         # applique tous les fichiers du dossier
kubectl get all
kubectl exec deploy/webapp -- env | grep -E 'APP_|REDIS'
minikube service webapp -n legeni-dev --url
```
Rafraîchissez plusieurs fois : le `hostname` change (2 pods), le compteur est partagé (Redis).

Testez la résolution DNS interne :
```bash
kubectl run -it --rm dnstest --image=busybox:1.36 --restart=Never -- nslookup redis
kubectl run -it --rm dnstest --image=busybox:1.36 --restart=Never -- nslookup redis.legeni-dev.svc.cluster.local
```

## Étape 7 : Modifier la configuration

```bash
kubectl edit configmap webapp-config          # APP_COLOR: "#d1242f"
kubectl exec deploy/webapp -- env | grep APP_COLOR   # inchangé !
kubectl rollout restart deployment webapp     # les pods relisent les variables au démarrage
kubectl rollout status deployment webapp
```
✅ La bannière devient rouge. Pourquoi la modification n'est-elle pas prise en compte sans redémarrage ?

---

## ✅ Vérifications

- [ ] `kubectl get all -n legeni-dev` : 2 pods webapp + 1 pod redis `Running`
- [ ] `nslookup redis` répond depuis un pod
- [ ] La couleur change via la ConfigMap, sans reconstruire l'image
- [ ] Aucun mot de passe en clair dans les fichiers YAML

## 🎯 Défi

1. Supprimez le `secretRef` du Deployment `webapp` et réappliquez : que renvoie `/ready` ? Pourquoi `/health` reste-t-il à 200 ? Remettez le `secretRef`.
2. Ajoutez un label `env: dev` sur toutes les ressources et listez-les avec `kubectl get all -l env=dev`.
3. Écrivez un fichier `redis-secret.yaml` équivalent à la commande de l'étape 3 (champ `stringData`) et expliquez pourquoi il **ne doit pas** être commité. Quelles solutions existent (Ansible Vault, Sealed Secrets, External Secrets) ?

<details>
<summary>Solution</summary>

1. `/ready` → 503 (`NOAUTH Authentication required` : l'application n'a pas le mot de passe), `/health` ne teste que le processus.
2. Ajouter `labels: {env: dev}` dans `metadata` de chaque objet (et dans le template des pods).
3. ```yaml
   apiVersion: v1
   kind: Secret
   metadata: {name: redis-secret, namespace: legeni-dev}
   type: Opaque
   stringData:
     REDIS_PASSWORD: "S3cr3t-Redis!"
   ```
   base64 n'est pas un chiffrement : quiconque lit le dépôt lit le mot de passe.
</details>
