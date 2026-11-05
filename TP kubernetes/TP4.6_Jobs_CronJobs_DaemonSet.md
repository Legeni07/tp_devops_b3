# TP 4.6 : Jobs, CronJobs, DaemonSets et initContainers – les autres charges de travail

**Objectif :** tout ne tourne pas en permanence derrière un Service. Savoir exécuter une **tâche ponctuelle** (Job : migration, import), une **tâche planifiée** (CronJob : sauvegarde, purge, contrôle), un **agent sur chaque nœud** (DaemonSet : collecte de logs, supervision), et **préparer** un Pod avant son démarrage (initContainer).

**Durée :** 1h15 – **Prérequis :** TP 4.3, TP 4.5 (Secret, PVC)

Travaillez dans le dossier `~/k8s-legeni/tp46/` et le namespace `batch` :
```bash
kubectl create namespace batch && kubectl config set-context --current --namespace=batch
```

> ℹ️ Le déploiement de WordPress **par un playbook Ansible** (ancien TP 4.6) est traité, en version complète, au [TP 5.6](../TP%20ansible/TP5.6_Ansible_Kubernetes.md).

| Objet | Le Pod… | Exemple |
|---|---|---|
| Deployment | tourne en permanence, N réplicas interchangeables | site web, API |
| **Job** | s'exécute **jusqu'à la réussite**, puis s'arrête | migration de base, import |
| **CronJob** | crée un Job **selon un horaire** | sauvegarde nocturne |
| **DaemonSet** | tourne en **un exemplaire par nœud** | agent de logs, de supervision |
| StatefulSet (TP 4.10) | tourne en permanence, avec identité et volume stables | base de données |

---

## Étape 1 : Un Job simple

`10-job-pi.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: calcul-pi
spec:
  backoffLimit: 3                    # nombre de nouvelles tentatives en cas d'échec
  ttlSecondsAfterFinished: 600       # le Job et ses Pods sont supprimés 10 min après la fin
  template:
    spec:
      restartPolicy: Never           # obligatoire pour un Job : Never ou OnFailure
      containers:
        - name: pi
          image: perl:5.40-slim
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(500)"]
```
```bash
kubectl apply -f 10-job-pi.yaml
kubectl get jobs,pods -w             # le Pod passe à Completed, le Job à 1/1 (Ctrl+C)
kubectl logs job/calcul-pi
kubectl wait --for=condition=complete job/calcul-pi --timeout=120s   # utilisé dans les pipelines CI/CD
```

## Étape 2 : Un Job qui échoue – nouvelles tentatives

`11-job-echec.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-ratee
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: busybox:1.36
          command: ["sh", "-c", "echo 'Migration du schéma...'; sleep 2; echo 'ERREUR : table verrouillée' >&2; exit 1"]
```
```bash
kubectl apply -f 11-job-echec.yaml
kubectl get pods -l job-name=migration-ratee -w     # 3 Pods en Error (1 essai + 2 nouvelles tentatives)
kubectl describe job migration-ratee | grep -A3 Conditions
kubectl wait --for=condition=failed job/migration-ratee --timeout=120s && echo "→ le pipeline doit s'arrêter ici"
```
Question : dans un pipeline de déploiement, pourquoi lance-t-on la migration de la base dans un Job **avant** de mettre à jour le Deployment de l'application ?

## Étape 3 : Un Job parallèle (traitement par lots)

`12-job-lots.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: traitement-lots
spec:
  completions: 6                    # 6 exécutions réussies au total
  parallelism: 2                    # 2 Pods en même temps
  completionMode: Indexed           # chaque Pod reçoit un index 0..5
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: lot
          image: busybox:1.36
          command: ["sh", "-c", "echo \"Traitement du lot $JOB_COMPLETION_INDEX sur $(hostname)\"; sleep 5"]
```
```bash
kubectl apply -f 12-job-lots.yaml
kubectl get pods -l job-name=traitement-lots -w     # jamais plus de 2 Pods Running à la fois
for p in $(kubectl get pods -l job-name=traitement-lots -o name); do kubectl logs $p; done | sort
```

## Étape 4 : Un CronJob – contrôle planifié de l'application

On déploie l'application fil rouge, puis un CronJob qui vérifie sa santé **toutes les minutes** (un « smoke test » permanent).

```bash
kubectl create deployment webapp --image=<votre_id_dockerhub>/legeni-webapp:1.0.0 --port=5000
kubectl expose deployment webapp --port=80 --target-port=5000
```
`20-cronjob-sante.yaml`
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: controle-sante
spec:
  schedule: "*/1 * * * *"            # format cron : minute heure jour mois jour-semaine
  concurrencyPolicy: Forbid          # ne pas lancer un contrôle si le précédent tourne encore
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: curl
              image: curlimages/curl:8.11.1
              command: ["sh", "-c", "date; curl -sf --max-time 5 http://webapp/health && echo ' → OK'"]
```
```bash
kubectl apply -f 20-cronjob-sante.yaml
kubectl get cronjob controle-sante
kubectl get jobs -w                      # un nouveau Job chaque minute (attendre 2-3 min)
kubectl logs job/$(kubectl get jobs -o name | grep controle | tail -1 | cut -d/ -f2)
```
Provoquez une panne et observez :
```bash
kubectl scale deployment webapp --replicas=0
# attendre la minute suivante
kubectl get jobs | grep controle         # le dernier Job est en échec (0/1, Failed)
kubectl scale deployment webapp --replicas=1
```
Lancement manuel immédiat, sans attendre l'horaire (pratique pour tester une sauvegarde) :
```bash
kubectl create job --from=cronjob/controle-sante controle-manuel
kubectl logs job/controle-manuel
```

## Étape 5 : Un CronJob de sauvegarde avec volume

`21-cronjob-sauvegarde.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sauvegardes
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sauvegarde
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: sauvegarde
              image: busybox:1.36
              command:
                - sh
                - -c
                - |
                  F=/backup/info-$(date +%Y%m%d-%H%M).json
                  wget -qO "$F" http://webapp/api/info && echo "Sauvegarde : $F"
                  ls -1t /backup/*.json | tail -n +6 | xargs -r rm     # ne garder que les 5 dernières
                  ls -l /backup
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: sauvegardes
```
```bash
kubectl apply -f 21-cronjob-sauvegarde.yaml
kubectl create job --from=cronjob/sauvegarde sauvegarde-1 && sleep 15 && kubectl logs job/sauvegarde-1
```
Mettez le CronJob en pause sans le supprimer : `kubectl patch cronjob sauvegarde -p '{"spec":{"suspend":true}}'`.

## Étape 6 : Un DaemonSet – un agent par nœud

`30-daemonset-agent.yaml`
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent-noeud
spec:
  selector:
    matchLabels:
      app: agent-noeud
  template:
    metadata:
      labels:
        app: agent-noeud
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane     # tourner aussi sur le nœud du plan de contrôle
          effect: NoSchedule
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo \"$(date) nœud=$NODE_NAME pods_logs=$(ls /var/log/pods | wc -l)\"; sleep 30; done"]
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName      # l'API "descendante" injecte le nom du nœud
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
          resources:
            requests: { cpu: 10m, memory: 16Mi }
            limits:   { cpu: 50m, memory: 32Mi }
      volumes:
        - name: varlog
          hostPath:
            path: /var/log                      # les logs du nœud lui-même
```
```bash
kubectl apply -f 30-daemonset-agent.yaml
kubectl get daemonset agent-noeud              # DESIRED = nombre de nœuds
kubectl logs ds/agent-noeud
```
Bonus (si votre machine a 8 Go de RAM libres) : ajoutez un nœud et constatez qu'un agent y est **automatiquement** déployé.
```bash
minikube node add
kubectl get pods -l app=agent-noeud -o wide    # un Pod par nœud
minikube node delete minikube-m02
```
Question : citez deux outils réels déployés en DaemonSet dans un cluster (indice : `kubectl get ds -A`).

## Étape 7 : Les initContainers – préparer avant de démarrer

Un initContainer s'exécute **jusqu'au bout avant** les conteneurs principaux. Usage typique : attendre une dépendance, télécharger une configuration, appliquer une migration.

`40-webapp-init.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-redis
spec:
  replicas: 1
  selector:
    matchLabels: { app: webapp-redis }
  template:
    metadata:
      labels: { app: webapp-redis }
    spec:
      initContainers:
        - name: attendre-redis
          image: busybox:1.36
          command: ["sh", "-c", "until nc -z redis 6379; do echo 'Redis indisponible, attente...'; sleep 2; done; echo 'Redis OK'"]
      containers:
        - name: webapp
          image: <votre_id_dockerhub>/legeni-webapp:1.0.0
          env:
            - name: REDIS_HOST
              value: redis
          ports: [{ containerPort: 5000 }]
```
```bash
kubectl apply -f 40-webapp-init.yaml
kubectl get pods -l app=webapp-redis            # Init:0/1 : bloqué tant que Redis n'existe pas
kubectl logs deploy/webapp-redis -c attendre-redis
kubectl create deployment redis --image=redis:7.4-alpine --port=6379 && kubectl expose deployment redis --port=6379
kubectl get pods -l app=webapp-redis -w         # Init:0/1 → PodInitializing → Running
```

## Étape 8 : Nettoyage

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace batch
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Job réussi | `kubectl get job calcul-pi` | `COMPLETIONS 1/1` |
| Échec détecté | `kubectl wait --for=condition=failed job/migration-ratee` | `condition met` après 3 essais |
| Parallélisme | Pods de `traitement-lots` | 6 logs, index 0 à 5, jamais plus de 2 simultanés |
| CronJob | `kubectl get jobs` après 3 min | 3 Jobs `controle-sante` réussis, 1 échec pendant la panne |
| DaemonSet | `kubectl get ds agent-noeud` | `DESIRED = READY = nombre de nœuds` |
| initContainer | `kubectl get pods -l app=webapp-redis` | `Init:0/1` puis `Running` après création de Redis |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| `spec.template.spec.restartPolicy: Unsupported value: "Always"` | `restartPolicy` absent dans un Job | `Never` ou `OnFailure` |
| `field is immutable` en modifiant un Job | la spec d'un Job ne se modifie pas | `kubectl delete job ...` puis `apply` (ou `ttlSecondsAfterFinished`) |
| Le CronJob ne se déclenche jamais | erreur de syntaxe cron, ou `suspend: true` | `kubectl describe cronjob`, tester sur [crontab.guru](https://crontab.guru) |
| Heure de déclenchement décalée | le cron est en UTC par défaut | `spec.timeZone: "Africa/Douala"` |
| Pod bloqué en `Init:0/1` | l'initContainer n'aboutit jamais | `kubectl logs <pod> -c <init>` |

## 🎯 Défi (20 min)

Écrivez un **Job de migration** pour l'application fil rouge : il attend Redis (initContainer), puis initialise le compteur de visites à `1000` (`redis-cli -h redis SET visits 1000`). Enchaînez ensuite, en une seule ligne de commande, « lancer le Job → attendre sa réussite → redémarrer `webapp-redis` », comme le ferait un pipeline.

<details>
<summary>Solution</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-compteur
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      initContainers:
        - name: attendre-redis
          image: busybox:1.36
          command: ["sh", "-c", "until nc -z redis 6379; do sleep 2; done"]
      containers:
        - name: migration
          image: redis:7.4-alpine
          command: ["redis-cli", "-h", "redis", "SET", "visits", "1000"]
```
```bash
kubectl apply -f 50-migration.yaml \
 && kubectl wait --for=condition=complete job/migration-compteur --timeout=120s \
 && kubectl rollout restart deployment/webapp-redis \
 && kubectl rollout status deployment/webapp-redis
kubectl port-forward deploy/webapp-redis 5000:5000 &  sleep 2; curl -s localhost:5000 | grep -A1 visites
```
</details>
