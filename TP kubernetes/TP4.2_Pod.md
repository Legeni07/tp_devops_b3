# TP 4.2 : Le Pod – la plus petite unité de Kubernetes

**Objectif :** créer un Pod à partir d'un fichier YAML, savoir l'interroger, y entrer, lire ses événements, et constater qu'un Pod seul **n'est pas résilient** (ce qui justifie le Deployment du TP 4.3).

**Durée :** 45 min – **Prérequis :** TP 4.1 (cluster Minikube démarré)

Travaillez dans le dossier `~/k8s-legeni/tp42/`.

---

## Étape 1 : Écrire le manifeste

`pod.yaml`
```yaml
apiVersion: v1            # groupe/version de l'API (kubectl api-resources | grep pods)
kind: Pod                 # type d'objet
metadata:
  name: my-pod            # nom unique dans le namespace
  labels:                 # étiquettes : c'est par elles que les autres objets trouveront ce Pod
    app: myapp
    tier: front
spec:
  containers:
    - name: nginx-container
      image: nginx:1.27-alpine      # jamais "latest" : on veut savoir ce qui tourne
      ports:
        - containerPort: 80         # documentaire, comme EXPOSE
```

Avant d'appliquer, validez sans rien créer :
```bash
kubectl apply -f pod.yaml --dry-run=server     # le serveur API valide le manifeste
kubectl explain pod.spec.containers.ports      # documentation du champ
```

## Étape 2 : Créer et observer

```bash
kubectl apply -f pod.yaml
kubectl get pods -w                 # Pending → ContainerCreating → Running (Ctrl+C)
kubectl get pod my-pod -o wide      # IP du Pod, nœud
kubectl describe pod my-pod         # lisez la section Events en bas : scheduling, pull, start
kubectl get pod my-pod -o yaml | less   # l'objet complet, avec le "status" ajouté par Kubernetes
```

Questions (compte-rendu) :
1. Dans `describe`, quels composants du cluster apparaissent dans les événements (`From`) ? Reliez-les à l'architecture vue en cours.
2. Dans `-o yaml`, quels champs n'étaient pas dans votre fichier ? Qui les a ajoutés ?

## Étape 3 : Interagir avec le Pod

```bash
kubectl logs my-pod                          # sortie standard de nginx
kubectl exec -it my-pod -- sh                # un shell dans le conteneur
	wget -qO- localhost | head -4
	exit
kubectl port-forward pod/my-pod 8080:80      # tunnel depuis votre poste (laisser tourner)
```
Dans un second terminal : `curl -s localhost:8080 | grep title`, puis relancez `kubectl logs my-pod` : la requête apparaît.

## Étape 4 : Labels et sélecteurs

```bash
kubectl get pods --show-labels
kubectl get pods -l app=myapp                # sélection par label
kubectl label pod my-pod env=dev             # ajouter un label à chaud
kubectl get pods -l 'env in (dev,staging)'
kubectl get pods -L app,env                  # labels en colonnes
```
> 💡 Les labels sont le **ciment** de Kubernetes : un Service ou un Deployment ne connaît pas les noms des Pods, il les sélectionne par labels.

## Étape 5 : Un Pod seul ne se répare pas

```bash
kubectl delete pod my-pod
kubectl get pods                             # plus rien : personne ne le recrée
```
Recréez-le, puis provoquez un **crash du processus** (et non une suppression) :
```bash
kubectl apply -f pod.yaml
kubectl exec my-pod -- sh -c 'kill 1'
kubectl get pod my-pod                       # RESTARTS = 1 : le kubelet a redémarré le CONTENEUR
```
Conclusion à rédiger : le **kubelet** redémarre un conteneur qui plante dans un Pod existant, mais **rien** ne recrée un Pod supprimé ou perdu avec son nœud. C'est le rôle du Deployment (TP 4.3).

## Étape 6 : Un Pod à deux conteneurs (sidecar)

`pod-sidecar.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-sidecar
spec:
  volumes:
    - name: html
      emptyDir: {}                           # volume partagé, vit aussi longtemps que le Pod
  containers:
    - name: web
      image: nginx:1.27-alpine
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
    - name: generateur
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date > /out/index.html; sleep 5; done"]
      volumeMounts:
        - name: html
          mountPath: /out
```
```bash
kubectl apply -f pod-sidecar.yaml
kubectl get pod web-sidecar                  # READY 2/2
kubectl exec web-sidecar -c web -- wget -qO- localhost      # la date change toutes les 5 s
kubectl logs web-sidecar -c generateur       # -c : choisir le conteneur
```
Question : les deux conteneurs partagent le volume, mais aussi l'**adresse IP** du Pod. Comment le vérifier ? *(indice : `kubectl exec web-sidecar -c generateur -- wget -qO- localhost`)*

## Étape 7 : Nettoyage

```bash
kubectl delete -f pod.yaml -f pod-sidecar.yaml
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Pod en marche | `kubectl get pod my-pod` | `1/1 Running` |
| Accès local | `curl localhost:8080` (port-forward actif) | page nginx |
| Sélection par label | `kubectl get pods -l app=myapp` | `my-pod` |
| Redémarrage du conteneur | `kubectl get pod my-pod` après `kill 1` | `RESTARTS 1` |
| Sidecar | `kubectl get pod web-sidecar` | `2/2 Running` |

## Erreurs fréquentes

| Statut | Cause probable | Diagnostic |
|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` | nom ou tag d'image inexistant | `kubectl describe pod` → Events |
| `Pending` longtemps | ressources insuffisantes, aucun nœud disponible | `kubectl describe pod` → `FailedScheduling` |
| `CrashLoopBackOff` | le processus principal se termine | `kubectl logs <pod> --previous` |
| `error validating data` | indentation YAML, champ mal orthographié | `kubectl apply --dry-run=server -f` |

## 🎯 Défi (10 min)

Sans écrire le YAML à la main, générez le manifeste d'un Pod `redis` (image `redis:7.4-alpine`, label `app=cache`), appliquez-le, puis exécutez `redis-cli PING` dans le Pod.

<details>
<summary>Solution</summary>

```bash
kubectl run redis --image=redis:7.4-alpine --labels=app=cache --dry-run=client -o yaml > redis-pod.yaml
kubectl apply -f redis-pod.yaml
kubectl exec redis -- redis-cli PING        # PONG
kubectl delete -f redis-pod.yaml
```
</details>
