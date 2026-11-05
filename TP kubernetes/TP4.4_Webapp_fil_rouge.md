# TP 4.4 : Déployer l'application fil rouge sur Kubernetes

**Objectif :** déployer **votre** image `legeni-webapp` (construite aux TPs 2.6/2.9) dans le cluster, l'exposer sur le port 8180, lui passer sa configuration par variables d'environnement et vérifier que la charge est répartie entre les réplicas.

**Durée :** 45 min – **Prérequis :** TP 4.3, image `legeni-webapp` poussée sur Docker Hub (TP 2.9) **ou** construite localement

Travaillez dans le dossier `~/k8s-legeni/tp44/`.

---

## Étape 1 : Rendre l'image disponible pour le cluster

Le nœud Minikube a **son propre** démon de conteneurs : il ne voit pas les images de votre machine. Deux possibilités :

| Option | Commandes | `image:` dans le manifeste |
|---|---|---|
| A. Depuis Docker Hub (recommandé, comme en production) | `docker push <id>/legeni-webapp:1.0.0` (TP 2.9) | `<id>/legeni-webapp:1.0.0` |
| B. Charger l'image locale | `minikube image load legeni-webapp:1.0.0` | `legeni-webapp:1.0.0` + `imagePullPolicy: IfNotPresent` |

```bash
minikube image ls | grep legeni      # vérification (option B)
```

## Étape 2 : Le Deployment

`webapp-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp-container
          image: <votre_id_dockerhub>/legeni-webapp:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000            # gunicorn écoute sur 5000 (voir le Dockerfile)
          env:
            - name: APP_ENV
              value: "k8s-dev"
            - name: APP_COLOR
              value: "#2da44e"
```

## Étape 3 : Le Service sur le port 8180

`webapp-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - port: 8180                 # port du Service dans le cluster
      targetPort: 5000           # port du conteneur
      nodePort: 30180            # port sur le nœud
```
```bash
kubectl apply -f webapp-deployment.yaml -f webapp-service.yaml
kubectl rollout status deployment/webapp-deployment
kubectl get pods -l app=webapp -o wide
URL=$(minikube service webapp-service --url) && echo $URL
curl -s $URL/api/info; echo
```
Ouvrez `$URL` dans le navigateur : bannière verte, environnement `k8s-dev`.

## Étape 4 : Vérifier la répartition et la limite du compteur en mémoire

```bash
for i in $(seq 1 10); do curl -s $URL/api/info | python3 -c 'import sys,json; print(json.load(sys.stdin)["hostname"])'; done | sort | uniq -c
```
Rechargez maintenant la page d'accueil plusieurs fois et observez le **nombre de visites** : il « saute » d'une valeur à l'autre.

Question : expliquez pourquoi, à l'aide de la ligne « Stockage du compteur » de la page. Quelle solution est mise en place au TP 4.7 ?

## Étape 5 : Le cas d'un Pod qui ne démarre pas

Changez l'image en `<votre_id_dockerhub>/legeni-webapp:9.9.9` (tag inexistant) et appliquez.
```bash
kubectl apply -f webapp-deployment.yaml
kubectl get pods -l app=webapp          # un Pod en ErrImagePull / ImagePullBackOff...
curl -s $URL/api/info; echo             # ...mais l'application répond toujours !
kubectl describe pod -l app=webapp | grep -A3 "Failed"
```
Question : pourquoi les anciens Pods n'ont-ils pas été supprimés ? *(indice : stratégie `RollingUpdate`, détaillée au TP 4.8)*

Revenez à l'état sain :
```bash
kubectl rollout undo deployment/webapp-deployment
kubectl rollout status deployment/webapp-deployment
```
⚠️ `rollout undo` corrige le **cluster** mais pas votre **fichier** : remettez le tag `1.0.0` dans `webapp-deployment.yaml`, sinon le prochain `kubectl apply` recassera tout. En production, le retour arrière se fait de préférence par un `git revert` suivi d'un nouveau déploiement.

## Étape 6 : Versionner les manifestes

```bash
cd ~/k8s-legeni && git init -b main 2>/dev/null; git add tp44 && git commit -m "TP 4.4 : webapp sur Kubernetes"
```
> 💡 À partir de maintenant, **la vérité est dans Git** : toute modification passe par le fichier puis `kubectl apply`. C'est ce que Jenkins (TP 3.8) et Ansible (TP 5.6) automatiseront.

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Déploiement | `kubectl rollout status deployment/webapp-deployment` | `successfully rolled out` |
| Accès | `curl $URL/api/info` | JSON avec `"env": "k8s-dev"` |
| Répartition | boucle de l'étape 4 | 2 hostnames |
| Résilience à une mauvaise image | `curl` pendant l'étape 5 | l'application répond toujours |
| Retour arrière | `kubectl rollout undo` | 2 Pods `Running` sur la bonne image |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| `ErrImagePull` sur une image locale | image absente du nœud Minikube | `minikube image load` + `imagePullPolicy: IfNotPresent` |
| `pull access denied` | dépôt Docker Hub privé ou ID erroné | rendre le dépôt public ou créer un `imagePullSecret` |
| Page inaccessible, Pods `Running` | `targetPort` ≠ 5000 | `kubectl get endpointslices`, corriger `targetPort` |
| Pods `Running` mais `0/1 READY` | probe mal configurée (si ajoutée) | voir TP 4.9 |

## 🎯 Défi (15 min)

Sans modifier `webapp-deployment.yaml`, déployez une **seconde** instance de l'application nommée `webapp-prod` (3 réplicas, `APP_ENV=prod`, bannière rouge) accessible sur le nodePort `30181`, qui coexiste avec la première. Indice : `sed` ou copie du fichier, et attention aux labels.

<details>
<summary>Solution</summary>

Il faut changer le **nom** du Deployment, son **label/selector** (sinon les deux Services sélectionneraient les Pods des deux Deployments) et le Service :
```bash
sed -e 's/webapp-deployment/webapp-prod/' -e 's/app: webapp$/app: webapp-prod/' \
    -e 's/replicas: 2/replicas: 3/' -e 's/k8s-dev/prod/' -e 's/#2da44e/#d1242f/' \
    webapp-deployment.yaml > webapp-prod.yaml
sed -e 's/webapp-service/webapp-prod/' -e 's/app: webapp$/app: webapp-prod/' -e 's/30180/30181/' \
    webapp-service.yaml > webapp-prod-svc.yaml
kubectl apply -f webapp-prod.yaml -f webapp-prod-svc.yaml
curl -s $(minikube service webapp-prod --url)/api/info
```
La bonne pratique (un namespace par environnement, un seul jeu de manifestes) est vue aux TPs 4.7 et 4.12.
</details>
