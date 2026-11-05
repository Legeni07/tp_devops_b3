# TP 5.5 : Secrets avec Ansible Vault – déploiement de WordPress

**Objectif :** déployer WordPress + MariaDB avec Docker Compose généré par Ansible, **sans jamais stocker un mot de passe en clair** dans Git. C'est la version complète et fonctionnelle de `lab_ansible.md` / `vars.yml`.

**Durée :** 1h30 – **Prérequis :** TP 5.4 (rôle `docker`) – **Fichiers :** `lab/wordpress.yml`, `lab/templates/wordpress-compose.yml.j2`, `lab/vars/`

---

## Étape 1 : Le problème

```bash
cat ../vars.yml
```
Ce fichier contient `mysql_root_password` **en clair**. Une fois poussé sur GitHub, il est public… et reste dans l'historique Git même après suppression.

## Étape 2 : Séparer variables publiques et secrètes

```bash
cd "TP ansible/lab"
cat vars/wordpress.yml                     # non sensible : versionné en clair
cp vars/wordpress_secrets.yml.exemple vars/wordpress_secrets.yml
nano vars/wordpress_secrets.yml            # choisissez vos mots de passe (12 caractères minimum)
```

## Étape 3 : Chiffrer avec Vault

```bash
ansible-vault encrypt vars/wordpress_secrets.yml    # choisir un mot de passe Vault
cat vars/wordpress_secrets.yml                      # $ANSIBLE_VAULT;1.1;AES256 ...
ansible-vault view vars/wordpress_secrets.yml
ansible-vault edit vars/wordpress_secrets.yml
```
Le fichier chiffré **peut** être commité. Seul le mot de passe Vault doit rester secret.

Autres commandes utiles :
```bash
ansible-vault rekey vars/wordpress_secrets.yml                  # changer le mot de passe Vault
ansible-vault encrypt_string 'MonSecret' --name 'api_token'     # chiffrer une seule variable
```

## Étape 4 : Le template Docker Compose

```bash
cat templates/wordpress-compose.yml.j2
```
Les mots de passe y sont des variables Jinja2. Le fichier généré sur le serveur a les droits `0600` : seul root peut le lire.

## Étape 5 : Déployer

```bash
cat wordpress.yml
ansible-playbook wordpress.yml --ask-vault-pass
```
Observez :
- `pre_tasks` + `assert` : on **refuse** de déployer avec des mots de passe trop faibles ;
- le rôle `docker` du TP 5.4 est **réutilisé** tel quel ;
- `community.docker.docker_compose_v2` démarre la stack (équivalent de `docker compose up -d`) ;
- la dernière tâche **attend** (`retries`/`until`) que WordPress réponde : MariaDB met 1 à 2 minutes à s'initialiser.

✅ `http://<IP_VM>:8103` affiche l'assistant d'installation WordPress.

> 🔎 Relancez le playbook : avec la collection `community.docker` 3.7 (celle d'Ubuntu 24.04), la tâche *Démarrer la stack WordPress* reste `changed` à chaque passage alors que rien n'est recréé. Vérifiez-le : `ansible db1 -b -m shell -a "docker inspect wordpress-db-1 | grep -m1 Created"` renvoie toujours la même date. C'est un **faux positif** (les événements *Waiting/Healthy* des healthchecks sont comptés comme des changements), corrigé dans les versions récentes de la collection (`ansible-galaxy collection install community.docker --upgrade`). Bonne occasion de rappeler qu'il faut **vérifier** l'idempotence, pas la supposer.

## Étape 6 : Le mot de passe Vault dans un fichier (pour l'automatisation)

Jenkins ne peut pas taper un mot de passe : on lui fournit un fichier.
```bash
echo 'votre_mot_de_passe_vault' > .vault_pass && chmod 600 .vault_pass
ansible-playbook wordpress.yml --vault-password-file .vault_pass
cat .gitignore            # .vault_pass est bien ignoré par Git
```
Dans Jenkins, ce fichier sera un credential de type **Secret file** (projet final).

## Étape 7 : Vérifier qu'aucun secret ne fuit

```bash
ansible-playbook wordpress.yml --vault-password-file .vault_pass -v | grep -c "AChanger" || echo "aucune fuite"
ansible db1 -b -m command -a "stat -c '%a %U' /opt/wordpress/docker-compose.yml"
```
Question : dans quelles tâches faudrait-il ajouter `no_log: true` si le mot de passe apparaissait dans les logs ?

---

## ✅ Vérifications

- [ ] `vars/wordpress_secrets.yml` commence par `$ANSIBLE_VAULT`
- [ ] WordPress accessible sur le port 8103
- [ ] `docker-compose.yml` sur db1 en `600 root`
- [ ] Le playbook refuse un mot de passe de moins de 12 caractères (testez avec `-e mysql_password=court`)

## 🎯 Défi

Ajoutez une tâche de **sauvegarde** de la base avant chaque redéploiement : `community.docker.docker_container_exec` lance `mariadb-dump` dans le conteneur `db`, le résultat est écrit dans `/opt/wordpress/backups/wordpress_<date>.sql` et seules les 5 dernières sauvegardes sont conservées.

<details>
<summary>Solution (extrait)</summary>

```yaml
    - name: Dump de la base
      community.docker.docker_container_exec:
        container: wordpress-db-1
        command: >-
          sh -c 'mariadb-dump -u root -p"$MARIADB_ROOT_PASSWORD" {{ mysql_database }}'
      register: dump
      no_log: true
      failed_when: false

    - name: Écrire la sauvegarde
      ansible.builtin.copy:
        content: "{{ dump.stdout }}"
        dest: "{{ wordpress_dir }}/backups/wordpress_{{ ansible_date_time.iso8601_basic_short }}.sql"
        mode: "0600"
      when: dump.rc | default(1) == 0

    - name: Lister les sauvegardes
      ansible.builtin.find:
        paths: "{{ wordpress_dir }}/backups"
        patterns: "*.sql"
      register: backups

    - name: Ne garder que les 5 plus récentes
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ (backups.files | sort(attribute='mtime', reverse=true))[5:] }}"
```
</details>
