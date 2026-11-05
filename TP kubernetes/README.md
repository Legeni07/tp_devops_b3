# TP Kubernetes – Parcours pratique

Fil conducteur : on part d'un pod isolé et on arrive à l'application fil rouge déployée en **staging et prod**, avec mises à jour sans coupure, autoscaling, persistance et nom de domaine.

| TP | Fichier | Contenu | Durée |
|---|---|---|---|
| 4.1 | [TP4.1_Installation_cluster.md](TP4.1_Installation_cluster.md) | Minikube (driver docker), exploration du cluster, addons | 1h |
| 4.2 | [TP4.2_Pod.md](TP4.2_Pod.md) | Premier Pod en YAML, labels, `exec`/`logs`/`port-forward`, Pod multi-conteneurs | 45 min |
| 4.3 | [TP4.3_Deployment_Service.md](TP4.3_Deployment_Service.md) | Deployment/ReplicaSet, auto-réparation, scaling, Service et répartition de charge, DNS interne | 1h |
| 4.4 | [TP4.4_Webapp_fil_rouge.md](TP4.4_Webapp_fil_rouge.md) | Application fil rouge sur le port 8180, configuration par variables, image cassée et `rollout undo` | 45 min |
| 4.5 | [TP4.5_WordPress_MySQL.md](TP4.5_WordPress_MySQL.md) | WordPress + MySQL : Secret, PVC, Service headless, preuve de persistance | 1h |
| 4.6 | [TP4.6_Jobs_CronJobs_DaemonSet.md](TP4.6_Jobs_CronJobs_DaemonSet.md) | Jobs (migration, lots parallèles), CronJobs (contrôle de santé, sauvegarde), DaemonSet, initContainers | 1h15 |
| 4.7 | [TP4.7_Namespace_ConfigMap_Secret.md](TP4.7_Namespace_ConfigMap_Secret.md) | Namespace, ConfigMap, Secret, DNS interne | 1h30 |
| 4.8 | [TP4.8_Scaling_RollingUpdate_Rollback.md](TP4.8_Scaling_RollingUpdate_Rollback.md) | Scaling, rolling update sans coupure, rollback, HPA | 1h30 |
| 4.9 | [TP4.9_Probes_Ressources.md](TP4.9_Probes_Ressources.md) | Liveness/readiness, requests/limits, OOMKilled, quotas | 1h |
| 4.10 | [TP4.10_Stockage_StatefulSet.md](TP4.10_Stockage_StatefulSet.md) | PV/PVC, StorageClass, StatefulSet | 1h |
| 4.11 | [TP4.11_Ingress.md](TP4.11_Ingress.md) | Ingress nginx, noms d'hôtes, HTTPS | 1h |
| 4.12 | [TP4.12_Kustomize_Environnements.md](TP4.12_Kustomize_Environnements.md) | base/overlays staging & prod, promotion d'une version | 1h30 |
| 4.13 | [TP4.13_Depannage.md](TP4.13_Depannage.md) | Méthode de diagnostic, 6 pannes à résoudre | 1h30 |

Ressources :
- [`../minikube/`](../minikube/) : script d'installation et erreurs fréquentes
- [`../Support de cours/Fiches memo/kubernetes.md`](../Support%20de%20cours/Fiches%20memo/kubernetes.md) : aide-mémoire kubectl
- [`../app-fil-rouge/`](../app-fil-rouge/) : application utilisée à partir du TP 4.4

> Organisation conseillée des fichiers : un dossier `~/k8s-legeni/` versionné dans Git, un fichier par ressource, préfixés par un numéro (`00-namespace.yaml`, `10-configmap.yaml`…) pour que `kubectl apply -f .` les applique dans l'ordre.
