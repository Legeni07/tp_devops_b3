# TP Jenkins – Parcours pratique

Progression : on commence par des jobs *freestyle* configurés à la main (pour comprendre ce que fait Jenkins), puis on passe au **pipeline as code** versionné, testé, et qui déploie.

| TP | Fichier | Contenu | Durée |
|---|---|---|---|
| 3.1 | [TP3.1_Installation.md](TP3.1_Installation.md) | Jenkins en Docker Compose, image avec Docker/kubectl/Ansible, persistance | 1h |
| 3.2 | [TP3.2_Build.md](TP3.2_Build.md) | Job freestyle paramétré : Git, build et run du site restaurant, workspace, variables, Poll SCM | 1h |
| 3.3 | [TP3.3_Test_acceptance.md](TP3.3_Test_acceptance.md) | Test d'acceptance (plugin HTTP Request et `curl`) sur le réseau de Jenkins, porte de qualité cassée volontairement | 45 min |
| 3.4 | [TP3.4_Artefact.md](TP3.4_Artefact.md) | Credentials Jenkins, tag unique traçable, push Docker Hub, archivage, chaînage vers un job deploy | 45 min |
| 3.5 | [TP3.5_Jenkinsfile.md](TP3.5_Jenkinsfile.md) | Pipeline déclaratif depuis Git : tests → build → acceptance → push | 2h |
| 3.6 | [TP3.6_Multibranch_Webhook.md](TP3.6_Multibranch_Webhook.md) | Multibranch, pull requests, webhook GitHub, protection de `main` | 1h30 |
| 3.7 | [TP3.7_Qualite_Tests_Paralleles.md](TP3.7_Qualite_Tests_Paralleles.md) | Stages parallèles, agents Docker, JUnit, couverture, flake8, hadolint, Trivy | 2h |
| 3.8 | [TP3.8_Deploiement_Continu.md](TP3.8_Deploiement_Continu.md) | Staging auto, validation humaine, prod, smoke test, rollback, notifications | 2h |

Ressources :
- [`jenkins-docker/`](jenkins-docker/) : Dockerfile, plugins et docker-compose de Jenkins (TP 3.1)
- [`../app-fil-rouge/`](../app-fil-rouge/) : application, `Jenkinsfile` de référence (TP 3.5), `deploy/` (TP 3.8)
- [`../odoo18-jenkins-pipeline-updated/`](../odoo18-jenkins-pipeline-updated/) : exemple de pipelines séparés test/build/release pour un projet Odoo
- [`../Support de cours/Fiches memo/jenkins.md`](../Support%20de%20cours/Fiches%20memo/jenkins.md) : aide-mémoire syntaxe déclarative

> Les pipelines des TPs 3.5, 3.7 et 3.8 (y compris l'approbation manuelle et le rollback automatique) ont été exécutés de bout en bout sur l'image Jenkins du TP 3.1.
