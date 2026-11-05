# TP 4.12 : Gérer plusieurs environnements avec Kustomize

**Objectif :** déployer **les mêmes manifests** en `staging` et en `prod` avec des différences maîtrisées (namespace, nombre de réplicas, couleur, version d'image), sans copier-coller de YAML. Kustomize est intégré à `kubectl` (`kubectl apply -k`).

**Durée :** 1h30 – **Prérequis :** TP 4.7 à 4.9

> Ce découpage `base/` + `overlays/` est celui attendu dans le **projet final** (dossier `k8s/`).

---

## Étape 1 : Arborescence

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── redis.yaml
│   └── webapp.yaml
└── overlays/
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```
```bash
mkdir -p ~/k8s/base ~/k8s/overlays/{staging,prod} && cd ~/k8s
```

## Étape 2 : La base (commune à tous les environnements)

Copiez dans `base/webapp.yaml` le Deployment + Service `webapp` des TPs précédents (avec probes et resources) et dans `base/redis.yaml` le StatefulSet + Service Redis, **en supprimant toutes les lignes `namespace:`** (c'est l'overlay qui le fixera). Le Service webapp est en `ClusterIP`.

Dans le Deployment, l'image est écrite **sans tag précis** : `image: legeni-webapp`, et `replicas: 2` (valeur par défaut, que la prod surchargera).

`base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - redis.yaml
  - webapp.yaml
labels:
  - pairs:
      app.kubernetes.io/part-of: legeni
configMapGenerator:
  - name: webapp-config
    literals:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
```

> 💡 `configMapGenerator` ajoute un suffixe de hash au nom de la ConfigMap (`webapp-config-7h2k9…`) et met à jour la référence dans le Deployment : **changer la config déclenche automatiquement un rolling update**. Plus besoin du `rollout restart` du TP 4.7.

## Étape 3 : L'overlay staging

`overlays/staging/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: legeni-staging
resources:
  - ../../base
labels:
  - pairs:
      env: staging
images:
  - name: legeni-webapp
    newName: <votre_id_dockerhub>/legeni-webapp
    newTag: "2.0.0"
configMapGenerator:
  - name: webapp-config
    behavior: merge
    literals:
      - APP_ENV=staging
      - APP_COLOR=#bf8700
```

## Étape 4 : L'overlay prod

`overlays/prod/replicas-patch.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 4
```

`overlays/prod/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: legeni-prod
resources:
  - ../../base
labels:
  - pairs:
      env: prod
patches:
  - path: replicas-patch.yaml
images:
  - name: legeni-webapp
    newName: <votre_id_dockerhub>/legeni-webapp
    newTag: "1.0.0"
configMapGenerator:
  - name: webapp-config
    behavior: merge
    literals:
      - APP_ENV=prod
      - APP_COLOR=#2da44e
```

## Étape 5 : Prévisualiser, comparer, appliquer

```bash
kubectl kustomize overlays/staging | less              # rendu final, rien n'est appliqué
diff <(kubectl kustomize overlays/staging) <(kubectl kustomize overlays/prod)

for env in staging prod; do
  kubectl create namespace legeni-$env --dry-run=client -o yaml | kubectl apply -f -
  kubectl create secret generic redis-secret -n legeni-$env \
    --from-literal=REDIS_PASSWORD="$(openssl rand -hex 12)" --dry-run=client -o yaml | kubectl apply -f -
  kubectl apply -k overlays/$env
done
kubectl get deploy -A -l app.kubernetes.io/part-of=legeni
```
✅ `legeni-staging` : 2 réplicas, version 2.0.0, bannière orange. `legeni-prod` : 4 réplicas, version 1.0.0, bannière verte.

## Étape 6 : Promouvoir une version (ce que fera Jenkins)

```bash
sed -i 's/newTag: .*/newTag: "2.0.0"/' overlays/prod/kustomization.yaml
git diff overlays/prod                                  # la promotion = un commit d'une ligne
kubectl diff -k overlays/prod                           # ce qui va changer
kubectl apply -k overlays/prod
kubectl rollout status deploy/webapp -n legeni-prod
```
> 💡 Si le binaire `kustomize` est installé, `kustomize edit set image …` modifie proprement le fichier. Dans tous les cas, la modification est **commitée dans Git** : l'historique Git devient l'historique des déploiements (principe *GitOps*).

## Étape 7 : Supprimer un environnement

```bash
kubectl delete -k overlays/staging
```

---

## ✅ Vérifications

- [ ] `kubectl kustomize overlays/prod` produit un YAML valide avec `namespace: legeni-prod` partout
- [ ] Les deux environnements tournent en parallèle avec des versions et couleurs différentes
- [ ] La promotion 1.0.0 → 2.0.0 en prod s'est faite par modification d'un seul fichier

## 🎯 Défi

1. Ajoutez un overlay `dev` qui utilise l'image locale (`imagePullPolicy: IfNotPresent` via un patch) et 1 seul réplica.
2. Bonus **Helm** : installez Helm et déployez Redis avec le chart Bitnami (`helm install redis oci://registry-1.docker.io/bitnamicharts/redis -n legeni-dev`). Comparez Kustomize (patchs sur du YAML) et Helm (templates + valeurs) : avantages, inconvénients.
