# TP 5.1 : Mise en place du lab, inventaire et commandes ad-hoc

**Objectif :** disposer d'une machine de contrôle Ansible et de **3 serveurs cibles**, écrire un inventaire, et administrer les serveurs à distance sans aucun agent installé.

**Durée :** 1h30 – **Prérequis :** Docker (TP 2.2), notions SSH

## Architecture du lab

```
 Machine de contrôle (WSL / VM)                 Nœuds gérés (conteneurs Ubuntu 24.04 + systemd + SSH)
 ┌──────────────────────────┐   SSH :2221   ┌────────┐  ports web : 8001 (80) / 8101 (8080)
 │ ansible, ansible-playbook│ ────────────▶ │  web1  │
 │ inventory.ini            │   SSH :2222   ├────────┤  ports web : 8002 (80) / 8102 (8080)
 │ ansible.cfg              │ ────────────▶ │  web2  │
 │ keys/id_ansible          │   SSH :2223   ├────────┤  ports web : 8003 (80) / 8103 (8080)
 └──────────────────────────┘ ────────────▶ │  db1   │
                                            └────────┘
```
Les nœuds sont des conteneurs **qui se comportent comme des VM** (systemd, services, apt, sudo) : on les administre exactement comme de vrais serveurs. Sur des VM VirtualBox ou dans le cloud, seul l'inventaire change.

---

## Étape 1 : Installer Ansible sur la machine de contrôle

```bash
sudo apt update && sudo apt install -y ansible     # ou : pipx install --include-deps ansible
ansible --version
ansible-galaxy collection list | grep -E 'community.docker|kubernetes.core'
```
Si les collections manquent :
```bash
ansible-galaxy collection install community.docker kubernetes.core
```

## Étape 2 : Démarrer le lab

```bash
cd "TP ansible/lab"
ssh-keygen -t ed25519 -f keys/id_ansible -N ""     # paire de clés dédiée au lab
docker compose up -d --build                       # 3 à 5 min la première fois
docker compose ps
ssh -i keys/id_ansible -p 2221 ansible@127.0.0.1 hostname    # test SSH manuel (taper "yes")
```

## Étape 3 : Comprendre `ansible.cfg` et l'inventaire

```bash
cat ansible.cfg inventory.ini
ansible-inventory --graph
ansible-inventory --host web1
```
Questions :
1. Quels sont les groupes ? Que contient le groupe `lab` ?
2. Où est définie la clé SSH utilisée ? Et l'utilisateur ?
3. Pourquoi `host_key_checking = False` est acceptable en lab mais pas en production ?

Écrivez le même inventaire au **format YAML** dans `inventory.yml` et vérifiez qu'il est équivalent :
```bash
ansible-inventory -i inventory.yml --graph
```
<details>
<summary>Solution</summary>

```yaml
lab:
  vars:
    ansible_user: ansible
    ansible_ssh_private_key_file: keys/id_ansible
    ansible_python_interpreter: /usr/bin/python3
  children:
    web:
      hosts:
        web1: {ansible_host: 127.0.0.1, ansible_port: 2221}
        web2: {ansible_host: 127.0.0.1, ansible_port: 2222}
    db:
      hosts:
        db1: {ansible_host: 127.0.0.1, ansible_port: 2223}
wordpress_servers:
  hosts:
    db1:
```
</details>

## Étape 4 : Commandes ad-hoc

Syntaxe : `ansible <cible> -m <module> -a "<arguments>" [-b pour sudo]`

```bash
ansible all -m ping                                    # connexion + Python OK ?
ansible web -m command -a "uptime"
ansible db1 -m shell -a "df -h / | tail -1"
ansible all -m setup -a "filter=ansible_distribution*" # facts
ansible all -m setup -a "filter=ansible_memtotal_mb"
```

Administration (avec élévation de privilèges `-b`) :
```bash
ansible web -b -m apt -a "name=tree state=present update_cache=yes"
ansible web -m command -a "tree -L 1 /etc/nginx" ; echo "→ normal, nginx n'est pas encore installé"
ansible all -b -m user -a "name=stagiaire shell=/bin/bash"
ansible all -m command -a "id stagiaire"
ansible all -b -m copy -a "content='Serveur géré par Ansible\n' dest=/etc/motd mode=0644"
ansible web1 -m command -a "cat /etc/motd"
```

## Étape 5 : L'idempotence

Relancez **deux fois** la commande `apt` et la commande `user` ci-dessus :
- 1ʳᵉ fois : `CHANGED` (jaune) – 2ᵉ fois : `SUCCESS` (vert), rien n'a été refait.

Comparez avec :
```bash
ansible web1 -m shell -a "echo ligne >> /tmp/fichier.txt"    # toujours CHANGED
ansible web1 -m shell -a "wc -l /tmp/fichier.txt"
```
Question : pourquoi préférer les modules (`apt`, `user`, `copy`, `lineinfile`…) aux modules `command`/`shell` ?

## Étape 6 : Cibler finement

```bash
ansible 'web:!web2' -m ping          # web sauf web2
ansible 'lab:&web' -m ping           # intersection
ansible all -m ping --limit db1
ansible all -m ping -f 1 -v          # 1 hôte à la fois, mode verbeux
```

---

## ✅ Vérifications

- [ ] `ansible all -m ping` → 3 `pong`
- [ ] `inventory.yml` équivalent à `inventory.ini`
- [ ] L'utilisateur `stagiaire` et le fichier `/etc/motd` existent sur les 3 nœuds
- [ ] Idempotence observée et expliquée

## 🎯 Défi

Avec **une seule** commande ad-hoc par point :
1. Afficher la version du noyau des 3 nœuds.
2. Supprimer l'utilisateur `stagiaire` et son dossier personnel.
3. Ajouter la ligne `alias ll='ls -alF'` dans `/home/ansible/.bashrc` de tous les nœuds, de façon idempotente.

<details>
<summary>Solution</summary>

```bash
ansible all -m setup -a "filter=ansible_kernel"
ansible all -b -m user -a "name=stagiaire state=absent remove=yes"
ansible all -m lineinfile -a "path=/home/ansible/.bashrc line=\"alias ll='ls -alF'\""
```
</details>

## Arrêter / réinitialiser le lab

```bash
docker compose stop                 # conserve l'état des nœuds
docker compose down -v              # repart de serveurs vierges
```
