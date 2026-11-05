# TP 5.3 : Variables, facts, boucles et conditions

**Objectif :** écrire **un seul** playbook qui s'adapte à chaque serveur : variables par groupe et par hôte, facts, boucles, conditions, et résultats de commandes réutilisés.

**Durée :** 1h30 – **Prérequis :** TP 5.2 – **Fichiers :** `lab/variables.yml`, `lab/group_vars/`, `lab/host_vars/`

---

## Étape 1 : Où placer les variables ?

```
lab/
├── inventory.ini
├── group_vars/
│   ├── all.yml      → tous les hôtes
│   ├── web.yml      → hôtes du groupe web
│   └── db.yml       → hôtes du groupe db
└── host_vars/
    └── web2.yml     → uniquement web2 (prioritaire sur group_vars)
```
```bash
cat group_vars/web.yml host_vars/web2.yml
ansible-inventory --host web1 | grep -A2 page_title
ansible-inventory --host web2 | grep -A2 page_title
```
Ordre de priorité simplifié (du plus faible au plus fort) : `defaults` d'un rôle < `group_vars/all` < `group_vars/<groupe>` < `host_vars/<hôte>` < `vars:` du play < `-e` en ligne de commande.

## Étape 2 : Exécuter le playbook

```bash
cat variables.yml
ansible-playbook variables.yml
```
Pour chaque tâche, identifiez la notion utilisée :

| Tâche | Notion |
|---|---|
| Afficher quelques facts | facts (`ansible_facts[...]`) |
| Installer les paquets communs | liste issue de `group_vars` (différente pour web et db) |
| Créer les groupes | `loop` + filtres `map`/`unique` |
| Créer les utilisateurs | `loop` sur une liste de dictionnaires, `loop_control.label` |
| Installer nginx uniquement… | `when` avec plusieurs conditions, `group_names` |
| Mesurer l'espace disque | `register` + `changed_when: false` |
| Publier la page | template + variable surchargée par `host_vars` |

✅ `http://<IP_VM>:8002` affiche le titre **« web2 (canari) »**, `:8001` le titre du groupe.

## Étape 3 : Surcharger en ligne de commande

```bash
ansible-playbook variables.yml -e "page_title='Maintenance ce soir'" --limit web
curl -s localhost:8001 | grep h1
ansible-playbook variables.yml --limit web          # retour au titre normal
```

## Étape 4 : Explorer les facts et les variables magiques

```bash
ansible web1 -m setup | less
ansible web1 -m debug -a "var=group_names"
ansible web1 -m debug -a "var=groups"
ansible web1 -m debug -a "var=hostvars['db1']['ansible_port']"
```

## Étape 5 : Tags

Ajoutez `tags: [utilisateurs]` aux deux tâches de création de groupes/utilisateurs, puis :
```bash
ansible-playbook variables.yml --list-tags
ansible-playbook variables.yml --tags utilisateurs
ansible-playbook variables.yml --skip-tags utilisateurs
```

## Étape 6 : Gestion d'erreurs

Ajoutez à la fin du playbook :
```yaml
    - name: Tenter une opération risquée
      block:
        - name: Commande qui échoue
          ansible.builtin.command: /bin/false
      rescue:
        - name: Réagir à l'échec
          ansible.builtin.debug:
            msg: "Échec intercepté sur {{ inventory_hostname }}, on continue proprement"
      always:
        - name: Toujours exécuté
          ansible.builtin.debug:
            msg: "Fin du bloc"
```
Ce mécanisme `block / rescue` est celui qui fera le **rollback automatique** du TP 5.6 et du projet final.

---

## ✅ Vérifications

- [ ] web1 et web2 affichent des titres différents sans modifier le playbook
- [ ] Les utilisateurs dev1, dev2, ops1 existent sur les 3 nœuds (`ansible lab -m command -a "id dev1"`)
- [ ] nginx n'est **pas** installé sur db1
- [ ] Le bloc `rescue` s'exécute sans faire échouer le playbook

## 🎯 Défi

Créez `utilisateurs.yml` qui lit la liste des utilisateurs dans `group_vars/all.yml` (nom, groupe, `present`/`absent`, clé SSH publique optionnelle) et :
- crée ou supprime chaque utilisateur selon son état ;
- installe sa clé SSH s'il en a une (module `ansible.posix.authorized_key`) ;
- n'ajoute les utilisateurs du groupe `operations` dans `sudo` **que** sur les serveurs `db`.

<details>
<summary>Solution (extrait)</summary>

```yaml
- name: Gestion des comptes
  hosts: lab
  become: true
  tasks:
    - name: Créer / supprimer les comptes
      ansible.builtin.user:
        name: "{{ item.nom }}"
        groups: "{{ item.groupe }}"
        state: "{{ item.etat | default('present') }}"
        remove: "{{ (item.etat | default('present')) == 'absent' }}"
      loop: "{{ comptes }}"

    - name: Installer les clés SSH
      ansible.posix.authorized_key:
        user: "{{ item.nom }}"
        key: "{{ item.cle }}"
      loop: "{{ comptes | selectattr('cle', 'defined') | selectattr('etat', 'ne', 'absent') }}"

    - name: Droits sudo pour les ops sur les serveurs db
      ansible.builtin.copy:
        dest: "/etc/sudoers.d/{{ item.nom }}"
        content: "{{ item.nom }} ALL=(ALL) NOPASSWD: ALL\n"
        mode: "0440"
        validate: visudo -cf %s
      loop: "{{ comptes | selectattr('groupe', 'eq', 'operations') }}"
      when: "'db' in group_names"
```
</details>
