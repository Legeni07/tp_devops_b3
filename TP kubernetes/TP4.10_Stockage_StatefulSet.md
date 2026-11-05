# TP 4.10 : Stockage persistant – PV, PVC et StatefulSet

**Objectif :** comprendre comment un pod (éphémère par nature) conserve ses données, et pourquoi les bases de données se déploient en **StatefulSet**.

**Durée :** 1h – **Prérequis :** TP 4.7 (si le quota du TP 4.9 est encore actif, supprimez-le : `kubectl delete resourcequota quota-dev -n legeni-dev`)

---

## Étape 1 : Constater le problème

```bash
kubectl exec deploy/redis -- sh -c 'redis-cli -a "$REDIS_PASSWORD" get visits' 2>/dev/null
kubectl delete pod -l app=redis
kubectl exec deploy/redis -- sh -c 'redis-cli -a "$REDIS_PASSWORD" get visits' 2>/dev/null   # (nil) : données perdues
```

## Étape 2 : Les StorageClasses et le provisionnement dynamique

```bash
kubectl get storageclass          # "standard (default)" sous Minikube
```
Un **PVC** (PersistentVolumeClaim) est une *demande* de stockage ; la StorageClass crée automatiquement le **PV** (PersistentVolume) correspondant.

## Étape 3 : Provisionnement statique (pour comprendre)

`pv-manuel.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manuel
spec:
  capacity: {storage: 1Gi}
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manuel
  hostPath:
    path: /data/pv-manuel
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manuel
  namespace: legeni-dev
spec:
  storageClassName: manuel
  accessModes: [ReadWriteOnce]
  resources:
    requests: {storage: 500Mi}
```
```bash
kubectl apply -f pv-manuel.yaml
kubectl get pv,pvc                # STATUS : Bound
```
Questions : que signifient `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany` ? Que devient le PV si on supprime le PVC avec `Retain` ? avec `Delete` ?
```bash
kubectl delete -f pv-manuel.yaml
```

## Étape 4 : Redis en StatefulSet

Supprimez le Deployment Redis (le Service est conservé) et créez `20-redis-statefulset.yaml` :
```bash
kubectl delete deployment redis
```
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: legeni-dev
spec:
  serviceName: redis
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
          args: ["--requirepass", "$(REDIS_PASSWORD)", "--appendonly", "yes"]
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef: {name: redis-secret, key: REDIS_PASSWORD}
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:          # un PVC est créé automatiquement PAR pod
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests: {storage: 1Gi}
```
```bash
kubectl apply -f 20-redis-statefulset.yaml
kubectl get sts,pods,pvc -l app=redis     # pod "redis-0", PVC "data-redis-0"
```

## Étape 5 : Prouver la persistance

```bash
URL=$(minikube service webapp -n legeni-dev --url)
for i in $(seq 5); do curl -s $URL >/dev/null; done
kubectl exec redis-0 -- sh -c 'redis-cli -a "$REDIS_PASSWORD" get visits' 2>/dev/null
kubectl delete pod redis-0
kubectl wait --for=condition=Ready pod/redis-0 --timeout=60s
kubectl exec redis-0 -- sh -c 'redis-cli -a "$REDIS_PASSWORD" get visits' 2>/dev/null   # ✅ valeur conservée
```

## Étape 6 : Identité stable

```bash
kubectl scale sts redis --replicas=3
kubectl get pods -l app=redis -w          # redis-0, redis-1, redis-2 créés DANS L'ORDRE
kubectl get pvc                           # un PVC par pod
kubectl scale sts redis --replicas=1      # suppression dans l'ordre inverse ; les PVC restent !
```

| | Deployment | StatefulSet |
|---|---|---|
| Nom des pods | aléatoire (`webapp-7c9f…`) | stable (`redis-0`) |
| Stockage | partagé ou aucun | un PVC dédié par pod |
| Ordre de démarrage | parallèle | séquentiel |
| Usage | applications sans état | bases de données, files de messages |

---

## ✅ Vérifications

- [ ] Le compteur survit à la suppression du pod `redis-0`
- [ ] `kubectl get pvc` montre `data-redis-0` en `Bound`
- [ ] Tableau Deployment / StatefulSet complété dans le compte-rendu

## 🎯 Défi

Reprenez le TP 4.5 (WordPress) : transformez MySQL en StatefulSet avec `volumeClaimTemplates` et ajoutez un PVC `ReadWriteOnce` pour `/var/www/html` de WordPress. Prouvez qu'un thème installé survit à la suppression du pod WordPress.
