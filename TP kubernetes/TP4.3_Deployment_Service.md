# TP 4.3 : Deployment, ReplicaSet et Service – une application qui se répare et se répartit

**Objectif :** déployer une application en plusieurs **réplicas** avec un Deployment, observer l'**auto-réparation** (self-healing), puis l'exposer par un **Service** qui répartit la charge entre les Pods.

**Durée :** 1h – **Prérequis :** TP 4.2

Travaillez dans le dossier `~/k8s-legeni/tp43/`.

---

## Étape 1 : Le Deployment

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3                       # état DÉSIRÉ : 3 Pods en permanence
  selector:
    matchLabels:
      app: myapp                    # ① le Deployment gère les Pods portant ce label...
  template:                         # ...modèle des Pods à créer
    metadata:
      labels:
        app: myapp                  # ② ...qui DOIT être identique à ①
    spec:
      containers:
        - name: nginx-container
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
```
```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods -l app=myapp
```
Question : observez les **noms** des Pods (`my-deployment-<hash>-<suffixe>`). À quoi correspond le hash ? Quel objet intermédiaire a été créé ?

> ⚠️ Erreur n°1 des débutants : un label différent entre `selector.matchLabels` et `template.metadata.labels`. Essayez de mettre `app: autre` en ② : `kubectl apply` refuse avec `selector does not match template labels`.

## Étape 2 : L'auto-réparation

Dans un second terminal, laissez tourner : `kubectl get pods -l app=myapp -w`

```bash
kubectl delete pod -l app=myapp --wait=false      # on supprime les 3 Pods d'un coup
kubectl get pods -l app=myapp                      # 3 nouveaux Pods, déjà en création
kubectl describe rs -l app=myapp | grep -A5 Events
```
Comparez avec l'étape 5 du TP 4.2 : qui a recréé les Pods ? *(réponse attendue : le contrôleur ReplicaSet du `kube-controller-manager`, qui compare en boucle l'état désiré à l'état réel)*

## Étape 3 : Mettre à l'échelle

```bash
kubectl scale deployment my-deployment --replicas=5
kubectl get pods -l app=myapp
kubectl scale deployment my-deployment --replicas=2
kubectl get pods -l app=myapp
```
Puis faites-le **de façon déclarative** : remettez `replicas: 3` dans le fichier et `kubectl apply -f deployment.yaml`.

Question : pourquoi la méthode déclarative est-elle préférable en équipe ? Que se passe-t-il au prochain `kubectl apply` si un collègue a fait un `kubectl scale` à la main ?

## Étape 4 : Le Service

Les Pods ont des IP **éphémères** (elles changent à chaque recréation). Le Service fournit une adresse et un nom **stables**.

`service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort                    # accessible depuis l'extérieur du cluster
  selector:
    app: myapp                      # envoie le trafic aux Pods portant ce label
  ports:
    - port: 80                      # port du Service (dans le cluster)
      targetPort: 80                # port du conteneur
      nodePort: 30080               # port ouvert sur le nœud (30000-32767)
```
```bash
kubectl apply -f service.yaml
kubectl get svc my-service
kubectl get endpointslices -l kubernetes.io/service-name=my-service   # les IP des 3 Pods derrière le Service
minikube service my-service --url
```

## Étape 5 : Vérifier la répartition de charge

On personnalise la page de chaque Pod pour voir qui répond :
```bash
for p in $(kubectl get pods -l app=myapp -o name); do
  kubectl exec $p -- sh -c 'echo "Servi par $(hostname)" > /usr/share/nginx/html/index.html'
done
URL=$(minikube service my-service --url)
for i in $(seq 1 10); do curl -s $URL; done | sort | uniq -c
```
✅ Les réponses viennent des **3 Pods**.

Testez le **DNS interne** depuis un Pod jetable :
```bash
kubectl run client --rm -it --image=busybox:1.36 --restart=Never -- sh
	wget -qO- my-service                                    # nom court (même namespace)
	wget -qO- my-service.default.svc.cluster.local          # nom complet
	nslookup my-service
	exit
```

## Étape 6 : Les types de Service

Modifiez `type:` et observez :
```bash
kubectl patch svc my-service -p '{"spec":{"type":"ClusterIP"}}'
kubectl get svc my-service                       # plus de port 30080 : le nodePort a été libéré
curl -s --max-time 3 $URL || echo "inaccessible depuis l'extérieur"
kubectl apply -f service.yaml                    # retour à l'état décrit dans Git
```
Complétez le tableau (compte-rendu) :

| Type | Accessible depuis | Usage typique |
|---|---|---|
| `ClusterIP` (défaut) | ? | ? |
| `NodePort` | ? | ? |
| `LoadBalancer` | ? (`minikube tunnel` en local) | ? |
| Ingress (TP 4.11, ce n'est pas un type de Service) | ? | ? |

## Étape 7 : Nettoyage

```bash
kubectl delete -f service.yaml -f deployment.yaml
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| 3 réplicas | `kubectl get deploy my-deployment` | `3/3` |
| Auto-réparation | suppression des Pods | 3 nouveaux Pods en moins de 30 s |
| Service | `kubectl get endpointslices -l kubernetes.io/service-name=my-service` | 3 adresses |
| Répartition | boucle `curl` | 3 hostnames différents |
| DNS interne | `wget -qO- my-service` depuis `client` | réponse d'un Pod |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| Le Service ne répond pas, aucun endpoint | `selector` du Service ≠ labels des Pods | `kubectl get pods --show-labels`, corriger le selector |
| `connection refused` via le Service | `targetPort` ≠ port d'écoute du conteneur | aligner `targetPort` sur `containerPort` |
| `provided port is already allocated` | `nodePort` déjà utilisé par un autre Service | changer de `nodePort` ou le laisser attribuer |
| `minikube service` ne rend pas la main (driver docker) | tunnel nécessaire sous WSL/Docker Desktop | laisser tourner, utiliser l'URL affichée |

## 🎯 Défi (15 min)

Déployez `hashicorp/http-echo:1.0` en 2 réplicas, configuré pour répondre `version bleue` (argument `-text=version bleue`, port 5678), derrière un Service `echo` en ClusterIP sur le port 80. Testez-le uniquement depuis un Pod jetable.

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 2
  selector:
    matchLabels: { app: echo }
  template:
    metadata:
      labels: { app: echo }
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo:1.0
          args: ["-text=version bleue"]
          ports: [{ containerPort: 5678 }]
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector: { app: echo }
  ports: [{ port: 80, targetPort: 5678 }]
```
```bash
kubectl apply -f echo.yaml
kubectl run t --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- echo   # version bleue
kubectl delete -f echo.yaml
```
</details>
