# TP 5.6 : Ansible pilote Kubernetes – déploiement multi-environnement avec rollback

**Objectif :** utiliser Ansible pour **générer et appliquer** les manifests Kubernetes de l'application fil rouge en *staging* ou en *prod*, attendre la fin du rolling update, tester, et **revenir automatiquement en arrière** en cas d'échec. Ce playbook est la brique « déploiement » du projet final : Jenkins n'aura plus qu'à l'appeler.

**Durée :** 1h30 – **Prérequis :** TP 4.7 à 4.9 (Kubernetes), TP 5.3 (block/rescue), cluster Minikube démarré – **Fichiers :** `lab/k8s_deploy.yml`, `lab/templates/k8s/webapp.yml.j2`, `lab/vars/k8s_*.yml`

> Ce TP remplace et complète l'ancien TP 4.6 (WordPress par playbook). Le TP 4.6 traite désormais des Jobs, CronJobs et DaemonSets.

---

## Étape 1 : Préparer la machine de contrôle

Le module `kubernetes.core.k8s` parle directement à l'API Kubernetes avec la librairie Python `kubernetes` et le fichier `~/.kube/config` :
```bash
sudo apt install -y python3-kubernetes        # ou : pip install kubernetes
ansible-galaxy collection install kubernetes.core
kubectl get nodes                             # le contexte courant doit être minikube
```

## Étape 2 : Lire le playbook et le template

```bash
cd "TP ansible/lab"
cat vars/k8s_staging.yml vars/k8s_prod.yml
cat templates/k8s/webapp.yml.j2
cat k8s_deploy.yml
```
Repérez :
- les variables qui changent entre staging et prod (namespace, couleur, réplicas, port) ;
- l'annotation `checksum/config` : elle change quand la configuration change, ce qui **force** un rolling update (sinon les pods ne relisent pas la ConfigMap, cf. TP 4.7) ;
- `wait: true` : Ansible attend que **tous** les pods de la nouvelle version soient prêts ;
- le *smoke test* exécuté **dans** un pod (`k8s_exec`) : il vérifie la version renvoyée par le Service ;
- `block` / `rescue` + `k8s_rollback` : retour à la révision précédente si une étape échoue.

## Étape 3 : Déployer en staging

```bash
export REDIS_PASSWORD=$(openssl rand -hex 16)        # en vrai : Ansible Vault ou credential Jenkins
ansible-playbook k8s_deploy.yml -e target_env=staging \
  -e webapp_image=<votre_id_dockerhub>/legeni-webapp -e app_version=1.0.0
kubectl get all -n legeni-staging
minikube service webapp -n legeni-staging --url
```

## Étape 4 : Promouvoir en production, puis mettre à jour

```bash
ansible-playbook k8s_deploy.yml -e target_env=prod -e webapp_image=<votre_id>/legeni-webapp -e app_version=1.0.0
ansible-playbook k8s_deploy.yml -e target_env=prod -e webapp_image=<votre_id>/legeni-webapp -e app_version=2.0.0
kubectl rollout history deployment/webapp -n legeni-prod
```

## Étape 5 : Déploiement raté → rollback automatique

```bash
ansible-playbook k8s_deploy.yml -e target_env=prod -e webapp_image=<votre_id>/legeni-webapp -e app_version=9.9.9
```
✅ Attendu (≈ 3 minutes, le temps du `wait_timeout`) :
- la tâche *Appliquer les manifests* échoue (pods en `ImagePullBackOff`) ;
- le `rescue` exécute *Retour à la version précédente* ;
- le playbook termine en **échec** (`failed=1`) : c'est voulu, Jenkins doit afficher un build rouge ;
- `kubectl get deploy webapp -n legeni-prod -o wide` → l'image est revenue en **2.0.0**, et grâce à `maxUnavailable: 0` le service n'a jamais été interrompu.

## Étape 6 : Idempotence

```bash
ansible-playbook k8s_deploy.yml -e target_env=prod -e webapp_image=<votre_id>/legeni-webapp -e app_version=2.0.0
```
Quelles tâches sont `changed` ? Pourquoi le smoke test est-il marqué `changed_when: false` ?

## Étape 7 : Le TP 4.6 revisité – WordPress par Ansible

Réécrivez le playbook `wordpress-playbook.yaml` du TP 4.6 pour qu'il :
1. crée le namespace `blog` et le Secret `mysql-pass` à partir d'une variable chiffrée Vault ;
2. applique les manifests du TP 4.5 (corrigés) via `kubernetes.core.k8s` avec `src:` et `namespace: blog` ;
3. attende que les deux Deployments soient prêts (`wait: true`).

<details>
<summary>Solution</summary>

```yaml
- name: WordPress sur Kubernetes
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - vars/wordpress_secrets.yml        # chiffré (TP 5.5)
  tasks:
    - name: Namespace
      kubernetes.core.k8s:
        definition: {apiVersion: v1, kind: Namespace, metadata: {name: blog}}

    - name: Secret MySQL
      kubernetes.core.k8s:
        definition:
          apiVersion: v1
          kind: Secret
          metadata: {name: mysql-pass, namespace: blog}
          stringData: {password: "{{ mysql_root_password }}"}
      no_log: true

    - name: Manifests MySQL, WordPress et Service
      kubernetes.core.k8s:
        src: "{{ item }}"
        namespace: blog
        wait: true
        wait_timeout: 300
      loop:
        - k8s-wordpress/mysql-deployment.yaml
        - k8s-wordpress/wordpress-deployment.yaml
        - k8s-wordpress/wordpress-service.yaml
```
</details>

---

## ✅ Vérifications

- [ ] staging (orange, 2 réplicas) et prod (vert, 3 réplicas) déployés par le même playbook
- [ ] Mise à jour 1.0.0 → 2.0.0 en prod sans interruption
- [ ] Rollback automatique démontré avec la version 9.9.9 (capture du `PLAY RECAP` et de `kubectl get deploy -o wide`)
- [ ] Aucun mot de passe dans les manifests ni dans les logs

## Pour aller plus loin

Le playbook génère les manifests avec Jinja2. Une alternative est de garder les manifests en **Kustomize** (TP 4.12) et de les appliquer avec `kubernetes.core.k8s` + `definition: "{{ lookup('kubernetes.core.kustomize', dir='k8s/overlays/prod') }}"`. Discutez des avantages de chaque approche dans le projet final.
