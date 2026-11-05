# INITIATION AU DEVOPS  Par Thomas BOGNI-DANCHI

Cours **B3COM0105 – 20 h**, orienté pratique : une même application (`app-fil-rouge/`) est conteneurisée, testée, intégrée, déployée et automatisée tout au long du cours, jusqu'au projet final de chaîne CI/CD complète.

➡️ Programme détaillé, planning des séances et modalités d'évaluation : [`Support de cours/Programme_pratique.md`](Support%20de%20cours/Programme_pratique.md)

## Plan du cours

Chaque module dispose de diapositives (`Support de cours/Diapos/M*.pptx` et `.pdf`) et d'un polycopié (`Support de cours/Polycopie/M*.md`). Chaque module se termine par un chapitre **Travaux pratiques** et un chapitre **Synthèse** (à retenir + quiz).

➢ MODULE 1 : Introduction au DevOps
    1. Historique – 2. Définition et contexte – 3. Principes fondamentaux
    4. Le cycle de vie DevOps – 5. Les outils du DevOps – 6. Ce que vous allez construire
    - TP : préparation de l'environnement (WSL/VM, VS Code, Git, Docker)

➢ MODULE 2 : Versioning et conteneurisation
    - Partie A – Git : 1. Notions sur les SCM et SVC – 2. Le workflow du versioning – 3. Introduction à Git
    - Partie B – Docker : 1. Introduction à la conteneurisation – 2. Les composants de Docker – 3. Dockerfile et commandes
      – 4. Stockage et réseau – 5. Images et registres – 6. Docker Compose – 7. Exploiter et dépanner
    - TP 2.1 → [`TP git/`](TP%20git/TP2.1_Git.md) ; TP 2.2 à 2.10 → [`TP docker/`](TP%20docker/README.md)

➢ MODULE 3 : Intégration et déploiement continus
    1. L'intégration continue – 2. Intégration continue et compilation – 3. Introduction à Jenkins
    4. Architecture de Jenkins – 5. Le pipeline Jenkins – 6. Déploiement continu
    - TP 3.1 à 3.8 → [`TP_jenkins/`](TP_jenkins/README.md)

➢ MODULE 4 : Orchestration avec Kubernetes
    1. Introduction – 2. Architecture et composants – 3. Les commandes kubectl – 4. Le fichier YAML et les objets
    5. Faire vivre l'application – 6. Exposer et gérer des environnements – 7. Diagnostiquer
    - TP 4.1 à 4.13 (dont 4.6 : Jobs, CronJobs, DaemonSets) → [`TP kubernetes/`](TP%20kubernetes/README.md)

➢ MODULE 5 : Gestion de configuration avec Ansible
    1. Introduction – 2. Architecture – 3. Le playbook – 4. Les commandes Ansible
    5. Rôles et secrets – 6. Ansible et Kubernetes
    - TP 5.1 à 5.6 → [`TP ansible/`](TP%20ansible/README.md)

➢ MODULE 6 : PROJET FINAL (par l'étudiant)
    - Chaîne CI/CD complète Git → Jenkins → Docker → Ansible → Kubernetes → [`Projet CICD/`](Projet%20CICD/README.md)

## Contenu du dépôt

| Dossier | Contenu |
|---|---|
| [`app-fil-rouge/`](app-fil-rouge/) | Application Flask + Redis avec tests, Dockerfile multi-stage, compose, Jenkinsfile de référence, fichiers de déploiement |
| [`TP git/`](TP%20git/) | TP 2.1 (Git en équipe) |
| [`TP docker/`](TP%20docker/) | TPs 2.2 – 2.10 (un fichier par TP) |
| [`TP_jenkins/`](TP_jenkins/) | TPs 3.x et image Jenkins prête à l'emploi (`jenkins-docker/`) |
| [`TP kubernetes/`](TP%20kubernetes/) | TPs 4.x |
| [`TP ansible/`](TP%20ansible/) | TPs 5.x et lab de 3 serveurs (`lab/`) avec playbooks et rôles |
| [`Projet CICD/`](Projet%20CICD/) | Énoncé du projet final, grille, corrigé de référence (`corrige/`, réservé à l'enseignant) |
| [`Support de cours/`](Support%20de%20cours/README.md) | Diapositives `.pptx` par section (avec notes de l'orateur), polycopiés, fiches mémo, programme pratique, sources de génération |
| [`Projet final/`](Projet%20final/) | Ancien sujet IC GROUP (Odoo + pgAdmin), utilisable comme sujet alternatif |
| `jenkins-install/`, `minikube/`, `legeni_static_website/`, `odoo18-jenkins-pipeline-updated/` | Ressources historiques et exemples |
