# TP Ansible – Parcours pratique

Tous les TPs utilisent le **lab** du dossier [`lab/`](lab/) : 3 serveurs Ubuntu (conteneurs avec systemd et SSH) administrés depuis votre machine. Les playbooks et rôles du dossier `lab/` sont les **corrigés de référence** : les étudiants les lisent, les exécutent, puis les modifient dans les défis.

| TP | Fichier | Contenu | Durée |
|---|---|---|---|
| 5.1 | [TP5.1_Lab_Inventaire_AdHoc.md](TP5.1_Lab_Inventaire_AdHoc.md) | Installation, lab, inventaire INI/YAML, `ansible.cfg`, commandes ad-hoc, idempotence | 1h30 |
| 5.2 | [TP5.2_Premier_Playbook.md](TP5.2_Premier_Playbook.md) | Playbook nginx, template Jinja2, handlers, `--check --diff`, dérive de configuration | 1h |
| 5.3 | [TP5.3_Variables_Conditions_Boucles.md](TP5.3_Variables_Conditions_Boucles.md) | group_vars/host_vars, facts, `loop`, `when`, `register`, tags, `block/rescue` | 1h30 |
| 5.4 | [TP5.4_Roles_Docker_Webapp.md](TP5.4_Roles_Docker_Webapp.md) | Rôles `docker` et `webapp`, dépendances, Ansible Galaxy | 2h |
| 5.5 | [TP5.5_Vault_WordPress.md](TP5.5_Vault_WordPress.md) | Ansible Vault, WordPress par Docker Compose généré | 1h30 |
| 5.6 | [TP5.6_Ansible_Kubernetes.md](TP5.6_Ansible_Kubernetes.md) | `kubernetes.core.k8s`, staging/prod, smoke test, rollback automatique | 1h30 |

Contenu de `lab/` :

```
lab/
├── docker-compose.yml, node/Dockerfile   # les 3 nœuds (web1, web2, db1)
├── ansible.cfg, inventory.ini            # configuration et inventaire
├── group_vars/, host_vars/, vars/        # variables (vars/wordpress_secrets.yml à chiffrer avec Vault)
├── templates/                            # index.html.j2, wordpress-compose.yml.j2, k8s/webapp.yml.j2
├── roles/docker, roles/webapp            # TP 5.4
└── nginx.yml, variables.yml, site.yml, wordpress.yml, k8s_deploy.yml
```

Ports exposés sur la machine hôte :

| Nœud | SSH | port 80 du nœud | port 8080 du nœud |
|---|---|---|---|
| web1 | 2221 | 8001 | 8101 |
| web2 | 2222 | 8002 | 8102 |
| db1 | 2223 | 8003 | 8103 |

Fichiers historiques : [`Lab.md`](Lab.md) (exemple pour CentOS/`yum`), [`lab_ansible.md`](lab_ansible.md), [`hosts.ini`](hosts.ini) et [`vars.yml`](vars.yml) (version complète dans le TP 5.5), [`../Ansible_conf.md`](../Ansible_conf.md) (configuration sur VM réelles).

Aide-mémoire : [`../Support de cours/Fiches memo/ansible.md`](../Support%20de%20cours/Fiches%20memo/ansible.md)
