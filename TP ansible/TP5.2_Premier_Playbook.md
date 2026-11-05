# TP 5.2 : Premier playbook – nginx, templates et handlers

**Objectif :** décrire l'état souhaité d'un serveur web dans un **playbook** versionné, rejouable à l'infini, avec une page générée par template et un rechargement de service uniquement quand c'est nécessaire.

**Durée :** 1h – **Prérequis :** TP 5.1 (lab démarré) – **Fichiers :** `lab/nginx.yml`, `lab/templates/index.html.j2`

---

## Étape 1 : Lire le playbook

```bash
cd "TP ansible/lab"
cat nginx.yml
```
```yaml
- name: Installer et configurer nginx
  hosts: web
  become: true

  tasks:
    - name: Installer nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Publier la page d'accueil
      ansible.builtin.template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
        mode: "0644"
      notify: Recharger nginx

    - name: Démarrer nginx et l'activer au démarrage
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Recharger nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```
Repérez : le **play**, la **cible** (`hosts`), l'**élévation de privilèges**, les **tâches**, le **handler** et le `notify` qui le déclenche.

## Étape 2 : Valider avant d'exécuter

```bash
ansible-playbook nginx.yml --syntax-check
ansible-playbook nginx.yml --list-tasks
ansible-playbook nginx.yml --check            # "dry run" : ce qui serait modifié
```
> `--check` peut échouer sur une tâche qui dépend d'une précédente (le paquet n'est pas réellement installé) : c'est normal la première fois.

## Étape 3 : Exécuter et observer l'idempotence

```bash
ansible-playbook nginx.yml       # 1er passage : changed > 0, le handler s'exécute
ansible-playbook nginx.yml       # 2e passage  : changed=0, le handler ne s'exécute pas
```
Ouvrez `http://<IP_VM>:8001` et `http://<IP_VM>:8002` : chaque page affiche le nom, la distribution, la RAM du serveur (facts) et l'équipe.

## Étape 4 : Le template Jinja2

```bash
cat templates/index.html.j2
```
- `{{ inventory_hostname }}` : nom de l'hôte dans l'inventaire
- `{{ ansible_facts['memtotal_mb'] }}` : fact collecté automatiquement
- `{% if %}` / `{% for %}` : logique dans le template
- `| default('...')` : filtre, valeur par défaut si la variable n'existe pas

Modifiez le texte du pied de page dans le template et relancez : seule la tâche *Publier la page* est `changed`, et le handler s'exécute.

## Étape 5 : Le mode diff

```bash
ansible-playbook nginx.yml --diff                    # affiche les lignes modifiées
ansible-playbook nginx.yml --check --diff --limit web1
```

## Étape 6 : Dérive de configuration

Quelqu'un modifie le serveur « à la main » :
```bash
ansible web1 -b -m shell -a "echo PIRATE > /var/www/html/index.html"
curl -s localhost:8001
ansible-playbook nginx.yml --diff        # Ansible détecte et corrige la dérive
curl -s localhost:8001 | grep h1
```
C'est le principe de l'**Infrastructure as Code** : le playbook dans Git est la référence, le serveur doit s'y conformer.

---

## ✅ Vérifications

- [ ] 2ᵉ exécution : `changed=0` (capture du `PLAY RECAP`)
- [ ] Pages web1/web2 accessibles avec leurs facts
- [ ] Dérive corrigée par une simple ré-exécution

## 🎯 Défi

Ajoutez au playbook :
1. Une tâche qui déploie un fichier de configuration nginx `/etc/nginx/conf.d/status.conf` exposant `/nginx_status` (`stub_status;`) sur le port 8090, avec `notify`.
2. Une tâche qui valide la configuration **avant** rechargement (`validate: nginx -t -c %s` ne fonctionne pas pour un fichier inclus : utilisez plutôt une tâche `command: nginx -t` avec `changed_when: false`).
3. Une dernière tâche `uri` qui vérifie que `http://127.0.0.1/nginx_status` renvoie 200.

<details>
<summary>Solution</summary>

```yaml
    - name: Configurer la page de statut
      ansible.builtin.copy:
        dest: /etc/nginx/conf.d/status.conf
        mode: "0644"
        content: |
          server {
            listen 8090;
            location /nginx_status { stub_status; }
          }
      notify: Recharger nginx

    - name: Vérifier la configuration nginx
      ansible.builtin.command: nginx -t
      changed_when: false

    - name: Appliquer les handlers maintenant
      ansible.builtin.meta: flush_handlers

    - name: Vérifier la page de statut
      ansible.builtin.uri:
        url: http://127.0.0.1:8090/nginx_status
        status_code: 200
```
(sur le port 8090 pour ne pas entrer en conflit avec le site par défaut ni avec l'application du TP 5.4, qui utilise 8080)
</details>
