# TP 4.5 : WordPress + MySQL – une application à état sur Kubernetes

**Objectif :** déployer une application en deux tiers (WordPress + base MySQL) en combinant **Secret**, **PersistentVolumeClaim**, **Deployment** et **Services**, et prouver que les données survivent à la perte du Pod de base de données.

**Durée :** 1h – **Prérequis :** TP 4.3 (Deployment, Service)

Travaillez dans le dossier `~/k8s-legeni/tp45/`, un fichier par ressource.

```
Navigateur ──▶ Service wordpress (NodePort) ──▶ Pod WordPress ──▶ Service mysql (headless) ──▶ Pod MySQL ──▶ PVC ──▶ volume
```

---

## Étape 1 : Un namespace dédié

```bash
kubectl create namespace wordpress --dry-run=client -o yaml > 00-namespace.yaml
kubectl apply -f 00-namespace.yaml
kubectl config set-context --current --namespace=wordpress
```

## Étape 2 : Le Secret (mot de passe de la base)

On ne l'écrit **jamais** en clair dans un Deployment ni dans Git :
```bash
kubectl create secret generic mysql-pass --from-literal=password="$(openssl rand -base64 12)"
kubectl get secret mysql-pass -o jsonpath='{.data.password}' | base64 -d; echo
```

## Étape 3 : Le stockage (PVC)

`10-mysql-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce            # monté en lecture-écriture par un seul nœud
  resources:
    requests:
      storage: 2Gi
```
```bash
kubectl apply -f 10-mysql-pvc.yaml
kubectl get pvc,pv             # STATUS Bound : la StorageClass "standard" de Minikube a créé le PV
kubectl get storageclass
```

## Étape 4 : MySQL (Deployment + Service headless)

`20-mysql.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  strategy:
    type: Recreate             # une seule instance doit écrire dans le volume
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
            - name: MYSQL_DATABASE
              value: wordpress
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: mysql                  # ce nom devient le nom DNS utilisé par WORDPRESS_DB_HOST
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
  clusterIP: None              # service "headless" : uniquement de la résolution DNS interne
```
```bash
kubectl apply -f 20-mysql.yaml
kubectl logs -f deploy/mysql   # attendre "ready for connections" (~1 min), Ctrl+C
```

Questions :
1. Pourquoi `strategy: Recreate` et non `RollingUpdate` (par défaut) pour une base de données ?
2. Pourquoi le Service MySQL n'a-t-il pas besoin d'être exposé à l'extérieur du cluster ?

## Étape 5 : WordPress (Deployment + Service NodePort)

`30-wordpress.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:6-apache
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql
            - name: WORDPRESS_DB_USER
              value: root
            - name: WORDPRESS_DB_NAME
              value: wordpress
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
```
```bash
kubectl apply -f 30-wordpress.yaml
kubectl get pods -w            # attendre 2 Pods Running, Ctrl+C
minikube service wordpress-service -n wordpress --url
```
Ouvrez l'URL, terminez l'installation de WordPress et **publiez un article** intitulé « Test de persistance ».

Vérifiez la connexion à la base par le DNS interne :
```bash
kubectl exec deploy/wordpress -- getent hosts mysql
kubectl exec deploy/mysql -- sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT post_title FROM wordpress.wp_posts WHERE post_status=\"publish\";"'
```

> 💡 Comme tout est dans des fichiers numérotés, `kubectl apply -f .` recrée l'ensemble dans l'ordre (le Secret excepté, créé hors Git).

## Étape 6 : Prouver la persistance

```bash
kubectl delete pod -l app=mysql        # on "perd" le serveur de base de données
kubectl get pods -w                    # un nouveau Pod mysql est créé et remonte le même PVC
```
Rechargez le site : l'article est toujours là. Allez plus loin :
```bash
kubectl delete -f 20-mysql.yaml        # on supprime le Deployment ET le Service...
kubectl get pvc                        # ...le PVC (et donc les données) reste
kubectl apply -f 20-mysql.yaml         # tout revient, avec les données
```

Question : que se passerait-il avec `kubectl delete -f .` ? Et avec `kubectl delete namespace wordpress` ? Quelle précaution prendre en production ?

## Étape 7 : Nettoyage

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace wordpress     # supprime TOUT, y compris le PVC et les données
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Volume réservé | `kubectl get pvc` | `Bound` |
| Secret utilisé, pas de mot de passe en clair | `grep -ri password *.yaml` | uniquement des `secretKeyRef` |
| DNS interne | `kubectl exec deploy/wordpress -- getent hosts mysql` | IP du Pod MySQL |
| Site | URL de `minikube service` | WordPress installé, article publié |
| Persistance | après `delete pod -l app=mysql` | article toujours présent |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| WordPress : « Error establishing a database connection » | MySQL pas encore prêt, ou mauvais mot de passe | `kubectl logs deploy/mysql`, attendre `ready for connections` |
| PVC `Pending` | pas de StorageClass par défaut | `minikube addons enable storage-provisioner default-storageclass` |
| MySQL `CrashLoopBackOff` après changement de mot de passe | la base existante garde l'ancien mot de passe | le mot de passe ne sert qu'à l'**initialisation** : supprimer le PVC pour repartir de zéro |
| `secret "mysql-pass" not found` | Secret créé dans un autre namespace | `kubectl get secret -A \| grep mysql-pass` |

## 🎯 Défi (20 min)

Ajoutez un **phpMyAdmin** (`phpmyadmin:5`) qui se connecte au Service `mysql`, exposé uniquement par `kubectl port-forward` (pas de NodePort : un outil d'administration ne doit pas être public). Connectez-vous avec le compte `root`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels: { app: phpmyadmin }
  template:
    metadata:
      labels: { app: phpmyadmin }
    spec:
      containers:
        - name: phpmyadmin
          image: phpmyadmin:5
          env:
            - name: PMA_HOST
              value: mysql
          ports: [{ containerPort: 80 }]
```
```bash
kubectl apply -f 40-phpmyadmin.yaml
kubectl port-forward deploy/phpmyadmin 8085:80      # http://localhost:8085, root / mot de passe du Secret
```
</details>

➡️ Version en **StatefulSet** (identité et volume stables par réplica) : TP 4.10. Déploiement de cet ensemble **par Ansible** : TP 5.6.
