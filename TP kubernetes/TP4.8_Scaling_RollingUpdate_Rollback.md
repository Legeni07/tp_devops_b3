# TP 4.8 : Mise à l'échelle, mises à jour sans interruption et retour arrière

**Objectif :** faire évoluer une application **en production sans coupure** : scaling manuel et automatique (HPA), rolling update contrôlé, rollback en une commande.

**Durée :** 1h30 – **Prérequis :** TP 4.7 (application déployée dans `legeni-dev`), addon `metrics-server`

Préparez **deux versions** de l'image (couleur et version différentes pour bien les distinguer) :
```bash
cd app-fil-rouge
docker build --build-arg APP_VERSION=1.0.0 -t $DOCKERHUB_ID/legeni-webapp:1.0.0 .
docker build --build-arg APP_VERSION=2.0.0 -t $DOCKERHUB_ID/legeni-webapp:2.0.0 .
docker push $DOCKERHUB_ID/legeni-webapp:1.0.0 && docker push $DOCKERHUB_ID/legeni-webapp:2.0.0
```

Dans un **second terminal**, laissez tourner une boucle qui interroge l'application en continu. Elle permet de *voir* chaque opération :
```bash
URL=$(minikube service webapp -n legeni-dev --url)
while true; do curl -s -m1 $URL/api/info || echo "❌ ERREUR"; echo; sleep 0.5; done
```

---

## Étape 1 : Scaling manuel

```bash
kubectl scale deployment webapp --replicas=5
kubectl get pods -l app=webapp -o wide -w     # Ctrl+C quand tout est Running
kubectl scale deployment webapp --replicas=2
```
Dans la boucle, observez le nombre de `hostname` différents.

## Étape 2 : Auto-réparation

```bash
kubectl delete pod -l app=webapp --wait=false
kubectl get pods -l app=webapp -w
```
Question : quel objet recrée les pods ? (indice : `kubectl get rs`)

## Étape 3 : Contrôler la stratégie de mise à jour

Ajoutez dans `30-webapp.yaml`, sous `spec:` du Deployment :
```yaml
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # au plus 1 pod en plus pendant la mise à jour
      maxUnavailable: 0    # jamais moins de pods disponibles que "replicas"
```
Passez `replicas: 4` puis `kubectl apply -f 30-webapp.yaml`.

## Étape 4 : Rolling update vers la 2.0.0

```bash
kubectl set image deployment/webapp webapp=$DOCKERHUB_ID/legeni-webapp:2.0.0
kubectl annotate deployment/webapp kubernetes.io/change-cause="passage en 2.0.0" --overwrite
kubectl rollout status deployment/webapp
kubectl get rs -l app=webapp                  # l'ancien ReplicaSet est conservé à 0 réplica
```
✅ Dans la boucle : les versions 1.0.0 et 2.0.0 cohabitent quelques secondes, puis seule la 2.0.0 répond. **Aucune ligne ❌ ERREUR.**

> 💡 En pratique, on modifie le tag dans le YAML puis `kubectl apply` (le YAML dans Git reste la source de vérité). `set image` est utilisé ici pour la démonstration – et par certains pipelines CI.

## Étape 5 : Un déploiement raté… et le rollback

```bash
kubectl set image deployment/webapp webapp=$DOCKERHUB_ID/legeni-webapp:9.9.9   # tag inexistant
kubectl annotate deployment/webapp kubernetes.io/change-cause="version 9.9.9 (erreur)" --overwrite
kubectl rollout status deployment/webapp --timeout=60s     # échoue
kubectl get pods -l app=webapp                             # ErrImagePull / ImagePullBackOff
```
Grâce à `maxUnavailable: 0`, **les anciens pods continuent de servir** : la boucle ne montre pas d'erreur.

```bash
kubectl rollout history deployment/webapp
kubectl rollout undo deployment/webapp                     # retour à la révision précédente
kubectl rollout undo deployment/webapp --to-revision=1     # ou à une révision précise
kubectl rollout status deployment/webapp
```

## Étape 6 : Autoscaling horizontal (HPA)

1. L'HPA a besoin de **requests** CPU pour calculer un pourcentage. Ajoutez au conteneur `webapp` :
	```yaml
	          resources:
	            requests: {cpu: 50m, memory: 64Mi}
	            limits:   {cpu: 200m, memory: 128Mi}
	```
	puis `kubectl apply -f 30-webapp.yaml`.
2. Créez `40-hpa.yaml` :
	```yaml
	apiVersion: autoscaling/v2
	kind: HorizontalPodAutoscaler
	metadata:
	  name: webapp
	  namespace: legeni-dev
	spec:
	  scaleTargetRef:
	    apiVersion: apps/v1
	    kind: Deployment
	    name: webapp
	  minReplicas: 2
	  maxReplicas: 8
	  metrics:
	    - type: Resource
	      resource:
	        name: cpu
	        target:
	          type: Utilization
	          averageUtilization: 50
	```
3. Générez de la charge depuis le cluster :
	```bash
	kubectl apply -f 40-hpa.yaml
	kubectl run charge --image=busybox:1.36 --restart=Never -- \
	  sh -c 'while true; do wget -q -O- http://webapp/ >/dev/null; done'
	kubectl get hpa webapp -w          # TARGETS monte, REPLICAS augmente
	```
4. Arrêtez la charge et observez la décroissance (≈ 5 minutes de stabilisation) :
	```bash
	kubectl delete pod charge
	```

---

## ✅ Vérifications

- [ ] Rolling update 1.0.0 → 2.0.0 sans aucune erreur dans la boucle (capture d'écran)
- [ ] `kubectl rollout history` montre les `change-cause`
- [ ] Le rollback a restauré la 2.0.0 après l'échec de la 9.9.9
- [ ] L'HPA a fait monter le nombre de réplicas au-dessus de 2 (capture de `kubectl get hpa`)

## 🎯 Défi

Recommencez l'étape 4 avec `maxSurge: 0` et `maxUnavailable: 1`, puis avec `strategy: {type: Recreate}`. Pour chaque stratégie, notez le comportement observé dans la boucle et proposez un cas d'usage.

<details>
<summary>Solution</summary>

- `maxSurge: 0 / maxUnavailable: 1` : un pod est supprimé avant que le nouveau ne démarre → capacité réduite mais pas de ressources supplémentaires (cluster très contraint).
- `Recreate` : tous les anciens pods sont supprimés puis les nouveaux créés → **coupure de service** (lignes ❌). Utile quand deux versions ne peuvent pas cohabiter (migration de schéma incompatible, volume `ReadWriteOnce` partagé).
</details>
