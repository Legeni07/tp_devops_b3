# TP 4.9 : Sondes de santé (probes) et gestion des ressources

**Objectif :** permettre à Kubernetes de savoir **quand un pod est vivant** et **quand il peut recevoir du trafic**, et maîtriser la consommation CPU/RAM de chaque conteneur.

**Durée :** 1h – **Prérequis :** TP 4.7, TP 4.8

| Sonde | Question posée | Si elle échoue |
|---|---|---|
| `startupProbe` | L'application a-t-elle fini de démarrer ? | Les autres sondes attendent ; après N échecs → redémarrage |
| `livenessProbe` | Le processus est-il bloqué ? | Le conteneur est **redémarré** |
| `readinessProbe` | Peut-il servir des requêtes maintenant ? | Le pod est **retiré du Service** (pas redémarré) |

L'application fil rouge expose `/health` (liveness) et `/ready` (readiness : vérifie Redis).

---

## Étape 1 : Ajouter les sondes

Dans `30-webapp.yaml`, ajoutez au conteneur `webapp` :
```yaml
          readinessProbe:
            httpGet: {path: /ready, port: 5000}
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet: {path: /health, port: 5000}
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests: {cpu: 50m, memory: 64Mi}
            limits:   {cpu: 200m, memory: 128Mi}
```
```bash
kubectl apply -f 30-webapp.yaml
kubectl describe pod -l app=webapp | grep -E 'Liveness|Readiness'
```

## Étape 2 : Readiness – panne d'une dépendance

```bash
kubectl scale deployment redis --replicas=0
kubectl get pods -l app=webapp -w            # READY passe à 0/1, mais RESTARTS reste à 0
kubectl describe svc webapp | grep Endpoints   # vide : le Service ne route plus vers ces pods
curl -s -o /dev/null -w '%{http_code}\n' $(minikube service webapp -n legeni-dev --url)
kubectl scale deployment redis --replicas=1  # tout revient automatiquement
```
Question : pourquoi est-il préférable de **ne pas** vérifier Redis dans la `livenessProbe` ?

## Étape 3 : Liveness – processus bloqué

On simule un blocage en pointant la sonde vers une URL qui n'existe pas :
```bash
kubectl patch deployment webapp --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/path","value":"/inexistant"}]'
kubectl get pods -l app=webapp -w            # RESTARTS augmente toutes les ~30 s
kubectl describe pod -l app=webapp | grep -A3 "Liveness probe failed" | head
kubectl rollout undo deployment webapp
```

## Étape 4 : Requests et limits

- **requests** : ce que le scheduler **réserve** sur le nœud pour placer le pod.
- **limits** : le **plafond** ; dépassement CPU → ralenti (throttling), dépassement mémoire → tué (`OOMKilled`).

```bash
kubectl top pods
kubectl describe node | grep -A8 "Allocated resources"
```

Faites l'expérience d'un pod trop gourmand, `oom.yaml` :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom
  namespace: legeni-dev
spec:
  restartPolicy: Never
  containers:
    - name: oom
      image: python:3.12-slim
      command: ["python", "-c", "x = bytearray(300*1024*1024)"]
      resources:
        limits: {memory: 64Mi}
```
```bash
kubectl apply -f oom.yaml && sleep 15
kubectl get pod oom                          # STATUS : OOMKilled
kubectl delete pod oom
```

Et un pod impossible à placer :
```bash
kubectl run gros --image=nginx:1.27-alpine --overrides='{"spec":{"containers":[{"name":"gros","image":"nginx:1.27-alpine","resources":{"requests":{"cpu":"50"}}}]}}'
kubectl get pod gros                         # Pending
kubectl describe pod gros | tail -3          # Insufficient cpu
kubectl delete pod gros
```

## Étape 5 : Quotas de namespace

`50-quota.yaml`
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-dev
  namespace: legeni-dev
spec:
  hard:
    pods: "10"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.memory: 2Gi
```
```bash
kubectl apply -f 50-quota.yaml
kubectl describe quota quota-dev
kubectl scale deployment webapp --replicas=15
kubectl get deploy webapp; kubectl get events --sort-by=.lastTimestamp | tail -3   # "exceeded quota"
kubectl scale deployment webapp --replicas=2
```

> ⚠️ Dès qu'un quota porte sur `requests.cpu`/`limits.memory`, **tout nouveau pod sans `resources` est refusé** dans le namespace (c'est le cas de Redis tant que le défi n'est pas fait). Pour continuer sans quota : `kubectl delete -f 50-quota.yaml`.

---

## ✅ Vérifications

- [ ] Redis arrêté → pods webapp `0/1 READY` sans redémarrage ; Redis relancé → retour à `1/1`
- [ ] Sonde liveness cassée → `RESTARTS` augmente
- [ ] Pod `OOMKilled` et pod `Pending` (Insufficient cpu) observés et expliqués
- [ ] Le quota bloque la création au-delà de la limite

## 🎯 Défi

Redis n'a ni sonde ni ressources. Ajoutez-lui une `readinessProbe` et une `livenessProbe` de type `exec` (`redis-cli -a $REDIS_PASSWORD ping`) et des requests/limits adaptées. Vérifiez avec `kubectl describe`.

<details>
<summary>Solution</summary>

```yaml
          readinessProbe:
            exec:
              command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" ping | grep PONG"]
            periodSeconds: 5
          livenessProbe:
            tcpSocket: {port: 6379}
            periodSeconds: 10
          resources:
            requests: {cpu: 50m, memory: 32Mi}
            limits:   {cpu: 200m, memory: 128Mi}
```
</details>
