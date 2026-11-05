# TP 4.11 : Exposer ses applications avec un Ingress

**Objectif :** remplacer les ports NodePort (30080, 31xxx…) par de **vrais noms de domaine** routés par un reverse proxy unique : l'Ingress Controller.

**Durée :** 1h – **Prérequis :** TP 4.7, addon `ingress` activé (TP 4.1), et si possible WordPress (TP 4.5) déployé dans le namespace `default`

```
                         ┌──────────── Ingress Controller (nginx) ────────────┐
 legeni.local     ─────▶ │ host: legeni.local          → Service webapp:80    │
 blog.legeni.local ────▶ │ host: blog.legeni.local     → Service wordpress:80 │
                         └────────────────────────────────────────────────────┘
```

---

## Étape 1 : Vérifier l'Ingress Controller

```bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass                   # "nginx"
```

## Étape 2 : Repasser le Service webapp en ClusterIP

Un Ingress cible des Services internes : plus besoin de NodePort. Dans `30-webapp.yaml` remplacez `type: NodePort` par `type: ClusterIP` et supprimez la ligne `nodePort:`.
```bash
kubectl apply -f 30-webapp.yaml
```
> Si `kubectl apply` refuse la modification, supprimez puis recréez le Service : `kubectl delete svc webapp && kubectl apply -f 30-webapp.yaml`.

## Étape 3 : Créer l'Ingress

`60-ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: legeni
  namespace: legeni-dev
spec:
  ingressClassName: nginx
  rules:
    - host: legeni.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp
                port:
                  number: 80
```
```bash
kubectl apply -f 60-ingress.yaml
kubectl get ingress -w                     # attendre qu'une ADDRESS apparaisse
```

## Étape 4 : Résolution de nom

```bash
echo "$(minikube ip) legeni.local blog.legeni.local" | sudo tee -a /etc/hosts
curl -s http://legeni.local/api/info
```
> 🪟 **WSL + navigateur Windows** : lancez `minikube tunnel` dans un terminal séparé, puis ajoutez `127.0.0.1 legeni.local blog.legeni.local` dans `C:\Windows\System32\drivers\etc\hosts` (éditeur lancé en administrateur).

## Étape 5 : Plusieurs applications derrière le même point d'entrée

L'Ingress doit être dans le **même namespace** que le Service ciblé. Créez `blog-ingress.yaml` pour WordPress (namespace `default`, Service `wordpress-service`) :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: blog.legeni.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wordpress-service
                port:
                  number: 80
```
```bash
kubectl apply -f blog-ingress.yaml
curl -sI http://blog.legeni.local | head -1
```

## Étape 6 : Routage par chemin et HTTPS

1. Ajoutez au même hôte une règle `path: /api` → Service `webapp`. Testez `curl http://legeni.local/api/info`.
2. HTTPS avec un certificat auto-signé :
	```bash
	openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
	  -keyout tls.key -out tls.crt -subj "/CN=legeni.local"
	kubectl create secret tls legeni-tls -n legeni-dev --cert=tls.crt --key=tls.key
	```
	Ajoutez dans `spec` de l'Ingress :
	```yaml
	  tls:
	    - hosts: [legeni.local]
	      secretName: legeni-tls
	```
	```bash
	kubectl apply -f 60-ingress.yaml
	curl -k https://legeni.local/api/info
	```

---

## ✅ Vérifications

- [ ] `curl http://legeni.local/api/info` répond sans numéro de port
- [ ] Deux applications servies par deux noms d'hôte différents
- [ ] `curl -k https://legeni.local` fonctionne

## 🎯 Défi

En production, on utilise **cert-manager** pour obtenir automatiquement des certificats Let's Encrypt. Recherchez dans la documentation quelle annotation ajouter à l'Ingress et quel objet (`ClusterIssuer`) créer. Rédigez le YAML (sans l'appliquer).
