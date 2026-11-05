# TP 4.1 : Installation d'un cluster Kubernetes (Minikube) et premiers pas

**Objectif :** disposer d'un cluster Kubernetes mono-nœud fonctionnel, savoir l'interroger et l'explorer avant d'y déployer quoi que ce soit.

**Durée :** 1h – **Prérequis :** Docker installé et utilisable sans `sudo` (TP 2.2)

---

## Étape 1 : Installer kubectl et Minikube

```bash
# kubectl (client Kubernetes)
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
kubectl version --client

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
minikube version
```

> ℹ️ Le script `../minikube/install_minikube.sh` installe aussi KVM/libvirt : inutile si on utilise le **driver docker**, recommandé sous WSL et en VM.

## Étape 2 : Démarrer le cluster

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```
⚠️ Ne **jamais** lancer `minikube` avec `sudo`. En cas d'erreur `permission denied ... docker.sock` ou `DRV_NOT_HEALTHY`, voir [`../minikube/kube_error.md`](../minikube/kube_error.md).

## Étape 3 : Explorer le cluster

```bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A                      # les composants du plan de contrôle tournent... dans des pods !
kubectl api-resources | head -30         # tous les types d'objets disponibles
kubectl explain deployment.spec.replicas # documentation intégrée
```

Questions (compte-rendu) :
1. Identifiez dans `kubectl get pods -n kube-system` : l'API server, etcd, le scheduler, le controller-manager, kube-proxy, CoreDNS. Associez chacun à son rôle vu en cours.
2. Quelle est l'adresse IP du nœud ? Sur quel runtime de conteneurs tourne-t-il (`kubectl get node -o wide`) ?

## Étape 4 : Activer les modules complémentaires

```bash
minikube addons list
minikube addons enable metrics-server    # nécessaire pour kubectl top et l'autoscaling (TP 4.8)
minikube addons enable ingress           # nécessaire pour le TP 4.11
minikube addons enable dashboard
```
Après 1 à 2 minutes :
```bash
kubectl top nodes
minikube dashboard --url                 # ouvrir l'URL dans le navigateur
```

## Étape 5 : Le contexte kubectl

`kubectl` peut piloter plusieurs clusters : la configuration est dans `~/.kube/config`.
```bash
kubectl config get-contexts
kubectl config current-context
```
> 💡 C'est ce fichier (aplati avec `kubectl config view --flatten --minify`) qui sera donné à Jenkins dans le projet final pour qu'il puisse déployer.

## Étape 6 : Premier déploiement impératif (et pourquoi on ne le fera plus)

```bash
kubectl create deployment hello --image=nginx:1.27-alpine
kubectl expose deployment hello --port=80 --type=NodePort
kubectl get deploy,rs,pods,svc -l app=hello
minikube service hello --url
kubectl delete deployment,service hello
```
Les commandes impératives ne laissent **aucune trace versionnée**. À partir du TP 4.2, tout se fait en **YAML déclaratif** (`kubectl apply -f`), stocké dans Git.

Astuce pour générer un squelette YAML sans rien créer :
```bash
kubectl create deployment hello --image=nginx:1.27-alpine --dry-run=client -o yaml > hello.yaml
```

---

## ✅ Vérifications

- [ ] `kubectl get nodes` → `Ready`
- [ ] `kubectl top nodes` affiche CPU/RAM
- [ ] Le pod `ingress-nginx-controller` est `Running` dans le namespace `ingress-nginx`
- [ ] Tableau composant → rôle complété

## Commandes utiles de gestion de Minikube

| Action | Commande |
|---|---|
| Arrêter / redémarrer | `minikube stop` / `minikube start` |
| Repartir de zéro | `minikube delete && minikube start --driver=docker` |
| Logs du cluster | `minikube logs` |
| Se connecter au nœud | `minikube ssh` |
| Utiliser une image locale sans la pousser | `minikube image load legeni-webapp:1.0.0` |
