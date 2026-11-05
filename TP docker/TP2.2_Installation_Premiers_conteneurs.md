# TP 2.2 : Installer Docker et maîtriser le cycle de vie d'un conteneur

**Objectif :** installer Docker Engine et le plugin Compose proprement, utiliser Docker **sans `sudo`**, puis piloter des conteneurs à la main (lancer, observer, entrer, arrêter, supprimer) et déployer une première application multi-conteneurs avec Compose.

**Durée :** 1h – **Prérequis :** une VM Ubuntu 24.04 ou WSL2 (Ubuntu), un accès Internet

---

## Étape 1 : Installer Docker Engine et Compose

On installe depuis le **dépôt officiel Docker** (le paquet `docker.io` d'Ubuntu est souvent en retard de plusieurs versions).

```bash
cd "TP docker"
cat docker_install.sh          # lisez TOUJOURS un script avant de l'exécuter avec sudo
bash docker_install.sh
```

Le script : ajoute la clé GPG dans un *keyring* (`signed-by=`, et non `apt-key` qui est déprécié), ajoute le dépôt, installe `docker-ce`, `docker-ce-cli`, `containerd.io` et `docker-compose-plugin`.

> ℹ️ Méthode alternative rapide (environnement jetable uniquement) : `curl -fsSL https://get.docker.com | sh`.
> Sous **Docker Desktop** (Windows/macOS avec intégration WSL), Docker est déjà installé : passez à l'étape 2.

## Étape 2 : Utiliser Docker sans `sudo`

```bash
sudo usermod -aG docker $USER      # ajoute votre utilisateur au groupe docker
newgrp docker                      # recharge les groupes (ou déconnexion / reconnexion)
docker version                     # doit afficher une section "Client" ET une section "Server"
docker info | grep -E 'Server Version|Storage Driver|Cgroup'
```

Question : `docker version` affiche deux parties, **Client** et **Server**. Que se passe-t-il réellement quand vous tapez une commande `docker` ? *(indice : `ls -l /var/run/docker.sock`)*

> ⚠️ Être dans le groupe `docker` revient à être **root** sur la machine : `docker run -v /:/host ...` donne accès à tout le disque. On ne l'accorde qu'aux administrateurs (voir aussi TP 3.1, où l'on donne ce droit à Jenkins).

## Étape 3 : Premier conteneur

```bash
docker run hello-world
```
Lisez le message affiché : il décrit les 4 étapes réalisées (client → démon → pull depuis Docker Hub → création du conteneur → sortie vers votre terminal).

```bash
docker images                 # l'image hello-world est maintenant en cache local
docker ps                     # aucun conteneur en cours...
docker ps -a                  # ...mais le conteneur existe, à l'état "Exited (0)"
```

## Étape 4 : Le cycle de vie d'un conteneur

Un conteneur passe par les états : `created` → `running` → (`paused`) → `exited` → supprimé.

```bash
# 1. Lancer en arrière-plan (-d), lui donner un nom, publier un port hôte:conteneur
docker run -d --name web -p 8081:80 nginx:1.27-alpine
docker ps
curl -s http://localhost:8081 | head -5

# 2. Observer
docker logs web                 # sortie standard du processus principal
docker logs -f --tail 5 web     # suivre en direct (Ctrl+C pour quitter), dans un 2e terminal : curl localhost:8081
docker top web                  # processus qui tournent dans le conteneur
docker stats --no-stream web    # CPU / mémoire
docker inspect web --format '{{.State.Status}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# 3. Entrer dans le conteneur
docker exec -it web sh          # un shell DANS le conteneur
	cat /etc/os-release         # Alpine, pas Ubuntu : le conteneur a son propre système de fichiers
	ps                          # nginx est le processus n°1
	exit

# 4. Modifier "à chaud" (et constater que c'est une mauvaise idée)
docker exec web sh -c 'echo "<h1>Modifié à la main</h1>" > /usr/share/nginx/html/index.html'
curl -s http://localhost:8081

# 5. Arrêter, redémarrer, supprimer
docker stop web && docker ps -a --filter name=web     # Exited
docker start web && curl -s http://localhost:8081     # la modification est toujours là...
docker rm -f web
docker run -d --name web -p 8081:80 nginx:1.27-alpine
curl -s http://localhost:8081 | grep -i title         # ...mais disparaît quand on RECRÉE le conteneur
docker rm -f web
```

Questions :
1. Quelle différence entre `docker stop` et `docker rm` ? Entre `docker rm` et `docker rmi` ?
2. Pourquoi la modification de l'étape 4.4 a-t-elle survécu à `stop/start` mais pas à `rm` + `run` ? Qu'en concluez-vous sur l'endroit où il faut mettre le contenu d'un site (→ TP 2.3) et les données (→ TP 2.4) ?

## Étape 5 : Conteneurs interactifs et jetables

```bash
docker run -it --rm ubuntu:24.04 bash       # -it : terminal interactif ; --rm : supprimé à la sortie
	apt-get update -qq && apt-get install -y -qq curl >/dev/null
	curl -sI https://www.docker.com | head -1
	exit
docker ps -a | grep ubuntu                  # rien : --rm a supprimé le conteneur
docker run --rm alpine:3.20 echo "Je vis 1 seconde"
```

> 💡 `--rm` est idéal pour les outils ponctuels : `docker run --rm -v "$PWD":/src -w /src python:3.12-slim python script.py` exécute un script sans installer Python sur la machine.

## Étape 6 : Première application multi-conteneurs avec Compose (Odoo + PostgreSQL)

1. Créez un dossier `~/odoo-compose` et le fichier `compose.yml` :
	```yaml
	services:
	  db:
	    image: postgres:16
	    environment:
	      POSTGRES_DB: postgres
	      POSTGRES_USER: odoo
	      POSTGRES_PASSWORD: odoo
	    volumes:
	      - db-data:/var/lib/postgresql/data
	    networks: [odoo_network]

	  odoo:
	    image: odoo:18.0
	    depends_on: [db]
	    environment:
	      HOST: db
	      USER: odoo
	      PASSWORD: odoo
	    ports:
	      - "8069:8069"
	    volumes:
	      - odoo-data:/var/lib/odoo
	    networks: [odoo_network]

	networks:
	  odoo_network:

	volumes:
	  db-data:
	  odoo-data:
	```
2. Démarrez et observez :
	```bash
	docker compose up -d           # plugin v2 : "docker compose", sans tiret
	docker compose ps
	docker compose logs -f odoo    # attendre "HTTP service (werkzeug) running"
	```
3. Ouvrez `http://localhost:8069` (ou `http://<IP_VM>:8069`) et créez une base de démonstration.
4. Arrêtez **sans perdre les données**, puis redémarrez : la base est toujours là.
	```bash
	docker compose down            # supprime conteneurs et réseau, PAS les volumes
	docker compose up -d
	docker compose down -v         # -v supprime aussi les volumes : à n'utiliser qu'en connaissance de cause
	```

> ℹ️ L'ancien binaire autonome `docker-compose` (avec tiret, Compose v1) n'est plus maintenu. Toutes les commandes du cours utilisent `docker compose`.

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Docker sans sudo | `docker ps` | pas de `permission denied` |
| Compose plugin | `docker compose version` | `Docker Compose version v2.x` (ou plus) |
| Cycle de vie | `docker ps -a` après l'étape 4 | aucun conteneur `web` restant |
| Compose | `docker compose ps` (étape 6) | `db` et `odoo` à l'état `running` |
| Persistance Compose | base Odoo présente après `down` / `up` | ✅ |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| `permission denied while trying to connect to the Docker daemon socket` | groupe `docker` pas encore pris en compte | `newgrp docker` ou se reconnecter |
| `Cannot connect to the Docker daemon` | démon arrêté (fréquent sous WSL sans systemd) | `sudo systemctl start docker` ou `sudo service docker start` |
| `Bind for 0.0.0.0:8081 failed: port is already allocated` | un autre conteneur/service utilise le port | `docker ps`, changer le port hôte (`-p 8082:80`) |
| `docker: 'compose' is not a docker command` | plugin non installé | `sudo apt-get install docker-compose-plugin` |

## 🎯 Défi (10 min)

Sans regarder la documentation, lancez un conteneur `redis:7.4-alpine` nommé `cache`, écrivez une clé avec `redis-cli` **depuis l'intérieur du conteneur**, relisez-la, puis affichez **uniquement** l'adresse IP du conteneur avec `docker inspect` et un format Go. Enfin, supprimez-le en une seule commande.

<details>
<summary>Solution</summary>

```bash
docker run -d --name cache redis:7.4-alpine
docker exec cache redis-cli SET cours devops
docker exec cache redis-cli GET cours                 # "devops"
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cache
docker rm -f cache
```
</details>
