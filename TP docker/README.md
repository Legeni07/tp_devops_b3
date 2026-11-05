# TP Docker – Parcours pratique

Les TPs se font dans l'ordre. Chaque TP se termine par une liste de **vérifications** (à cocher dans le compte-rendu) et un **défi** dont la solution est masquée.

| TP | Fichier | Contenu | Durée |
|---|---|---|---|
| 2.1 | [../TP git/TP2.1_Git.md](../TP%20git/TP2.1_Git.md) | Git en équipe : branches, PR, conflit, tags (dépôt `legeni-webapp`) | 2h |
| 2.2 | [TP2.2_Installation_Premiers_conteneurs.md](TP2.2_Installation_Premiers_conteneurs.md) | Installation de Docker et Compose, cycle de vie d'un conteneur (`run`, `logs`, `exec`, `stop`, `rm`), première stack Odoo | 1h |
| 2.3 | [TP2.3_Dockerfile_Image_DockerHub.md](TP2.3_Dockerfile_Image_DockerHub.md) | Premier Dockerfile (site restaurant), couches et cache, optimisation, push sur Docker Hub avec jeton | 1h |
| 2.4 | [TP2.4_Reseaux_Volumes.md](TP2.4_Reseaux_Volumes.md) | Réseau bridge et DNS, volumes anonymes et nommés (B3_jour / B3_soir), base PostgreSQL persistante | 1h |
| 2.5 | [TP2.5_Configuration_Processus.md](TP2.5_Configuration_Processus.md) | Configuration par l'environnement, `ARG`/`ENV`, `ENTRYPOINT`/`CMD`, PID 1 et signaux, script d'entrée | 1h30 |
| 2.6 | [TP2.6_Dockerfile_avance.md](TP2.6_Dockerfile_avance.md) | Cache, `.dockerignore`, multi-stage, tests dans le build, non-root, HEALTHCHECK | 1h30 |
| 2.7 | [TP2.7_Reseaux_Volumes_avances.md](TP2.7_Reseaux_Volumes_avances.md) | DNS des réseaux, isolation 3 tiers, bind mount, tmpfs, sauvegarde/restauration | 1h30 |
| 2.8 | [TP2.8_Compose_multi_services.md](TP2.8_Compose_multi_services.md) | Stack proxy + app + Redis, healthchecks, `.env`, scale, panne ; WordPress | 2h |
| 2.9 | [TP2.9_Registre_Tags_Securite.md](TP2.9_Registre_Tags_Securite.md) | Stratégie de tags, registre privé, scan Trivy, lint hadolint | 1h30 |
| 2.10 | [TP2.10_Exploitation_Depannage.md](TP2.10_Exploitation_Depannage.md) | Logs, inspect, limites, restart policies, ménage, 5 pannes à résoudre | 1h30 |

Ressources :
- [struture_dockerfile.md](struture_dockerfile.md) : les instructions du Dockerfile
- [`../app-fil-rouge/`](../app-fil-rouge/) : l'application utilisée à partir du TP 2.6 (et dans les TPs Jenkins, Kubernetes, Ansible et le projet final)
- [`../Support de cours/Fiches memo/docker.md`](../Support%20de%20cours/Fiches%20memo/docker.md) : aide-mémoire

> Le compte-rendu de chaque TP contient : les commandes passées, les captures des vérifications, les réponses aux questions et la solution du défi.
