# TP 5.4 : Les rôles – installer Docker et déployer l'application fil rouge

**Objectif :** organiser le code Ansible en **rôles réutilisables** : un rôle `docker` (installation de Docker CE) et un rôle `webapp` (déploiement de l'application + Redis), puis les assembler dans un playbook de 5 lignes.

**Durée :** 2h – **Prérequis :** TP 5.3, image `legeni-webapp` sur Docker Hub (TP 2.9) – **Fichiers :** `lab/roles/`, `lab/site.yml`

---

## Étape 1 : Anatomie d'un rôle

```bash
ansible-galaxy role init --init-path /tmp demo && tree /tmp/demo     # squelette standard
tree roles/
```
| Dossier | Contenu |
|---|---|
| `tasks/main.yml` | les tâches (point d'entrée) |
| `handlers/main.yml` | les handlers |
| `defaults/main.yml` | variables **par défaut**, faciles à surcharger |
| `vars/main.yml` | variables internes, difficiles à surcharger |
| `templates/`, `files/` | fichiers Jinja2 et fichiers statiques |
| `meta/main.yml` | description, plateformes, **dépendances** |

## Étape 2 : Le rôle `docker`

```bash
cat roles/docker/tasks/main.yml roles/docker/defaults/main.yml
```
Questions :
1. Comment l'URL du dépôt s'adapte-t-elle à la distribution et à l'architecture ?
2. Pourquoi installer `python3-docker` / `python3-requests` sur les nœuds ?
3. Que se passe-t-il au 2ᵉ passage si Docker est déjà installé ?

## Étape 3 : Le rôle `webapp`

```bash
cat roles/webapp/defaults/main.yml roles/webapp/tasks/main.yml roles/webapp/meta/main.yml
```
- Toutes les valeurs (image, version, port, couleur…) sont dans `defaults` : **le rôle est générique**.
- `meta/main.yml` déclare la dépendance au rôle `docker` : il sera exécuté automatiquement avant.
- Les deux dernières tâches **vérifient** le déploiement (`uri` + `until`) : un rôle de déploiement doit prouver qu'il a réussi.
- `no_log: true` empêche le mot de passe Redis d'apparaître dans les logs.

## Étape 4 : Déployer

1. Dans `group_vars/all.yml`, remplacez `webapp_image` par **votre** image Docker Hub.
2. Lancez :
	```bash
	cat site.yml
	ansible-playbook site.yml
	```
3. ✅ `http://<IP_VM>:8101` (web1) et `:8102` (web2) affichent l'application.

> 🐳 Docker tourne **dans** les nœuds du lab (Docker-in-Docker, d'où `privileged: true` dans le `docker-compose.yml` du lab). Sur de vraies VM, rien ne change.

## Étape 5 : Variabiliser par environnement

Créez `host_vars/web2.yml` (complétez le fichier existant) :
```yaml
webapp_env: "preprod"
webapp_color: "#bf8700"
webapp_version: "2.0.0"
```
```bash
ansible-playbook site.yml --diff
curl -s localhost:8102/api/info
```
✅ web2 tourne en 2.0.0 (orange), web1 reste en 1.0.0.

## Étape 6 : Mise à jour applicative depuis la ligne de commande

C'est exactement ainsi que Jenkins appellera Ansible :
```bash
ansible-playbook site.yml -e webapp_version=2.0.0 --limit web1
```
Que se passe-t-il si on demande une version qui n'existe pas (`-e webapp_version=9.9.9`) ? Le service est-il coupé ?

## Étape 7 : Réutiliser des rôles de la communauté (Ansible Galaxy)

`requirements.yml` :
```yaml
roles:
  - name: geerlingguy.docker
    version: 7.4.1
collections:
  - name: community.docker
  - name: kubernetes.core
```
```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy role list
```
Comparez `geerlingguy.docker` (sur https://galaxy.ansible.com) avec notre rôle `docker` : nombre de variables, plateformes supportées.

---

## ✅ Vérifications

- [ ] `ansible-playbook site.yml` 2ᵉ passage → `changed=0`
- [ ] web1 en 1.0.0, web2 en 2.0.0 (captures `/api/info`)
- [ ] Aucun mot de passe visible dans la sortie d'`ansible-playbook`

## 🎯 Défi

Ajoutez au rôle `webapp` :
1. une variable `webapp_state` (`present` / `absent`) qui permet de **désinstaller** proprement l'application (conteneurs + réseau, en gardant le volume Redis) ;
2. un **test du rôle** : un playbook `tests/test_webapp.yml` qui applique le rôle puis vérifie avec `assert` que `/api/info` renvoie la version attendue et que le conteneur `redis` n'a pas de port publié.

<details>
<summary>Solution (extrait)</summary>

```yaml
- name: Supprimer l'application
  when: webapp_state == 'absent'
  block:
    - name: Supprimer les conteneurs
      community.docker.docker_container:
        name: "{{ item }}"
        state: absent
      loop: [webapp, redis]
    - name: Supprimer le réseau
      community.docker.docker_network:
        name: "{{ webapp_network }}"
        state: absent
```
Et entourer les tâches de déploiement d'un `block` avec `when: webapp_state == 'present'`.
</details>
