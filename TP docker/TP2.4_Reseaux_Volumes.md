# TP 2.4 : Réseaux et volumes – faire communiquer et persister

**Objectif :** relier deux conteneurs par un **réseau bridge** et leur faire partager des données, d'abord avec un volume **anonyme** (temporaire), puis avec un volume **nommé** (persistant), et comprendre ce qui survit à la suppression d'un conteneur.

**Durée :** 1h – **Prérequis :** TP 2.2

**Énoncé (TAF) :** créer deux conteneurs `B3_jour` et `B3_soir`,
1. les relier par un réseau bridge ;
2. leur faire partager des données via un volume temporaire ;
3. leur faire partager des données via un volume persistant.

---

## Étape 1 : Le réseau par défaut et ses limites

```bash
docker network ls                         # bridge, host, none : les 3 réseaux créés par Docker
docker run -d --name c1 alpine:3.20 sleep infinity
docker run -d --name c2 alpine:3.20 sleep infinity
docker exec c1 ping -c 2 c2                 # ❌ "bad address 'c2'"
docker exec c1 ping -c 2 $(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c2)   # ✅ par IP
docker rm -f c1 c2
```
Question : sur le réseau `bridge` **par défaut**, les conteneurs se joignent par IP mais **pas par nom**. Pourquoi est-ce un problème quand un conteneur est recréé ?

> ℹ️ On utilise ici `alpine` + `sleep infinity` : un conteneur `ubuntu` lancé avec `-d` et sans commande s'arrête immédiatement (bash n'a pas de terminal). Alpine contient déjà `ping`.

## Étape 2 : Un réseau bridge personnalisé

```bash
docker network create -d bridge mon_reseau
docker network inspect mon_reseau --format '{{(index .IPAM.Config 0).Subnet}}'

docker run -d --name B3_jour --network mon_reseau alpine:3.20 sleep infinity
docker run -d --name B3_soir --network mon_reseau alpine:3.20 sleep infinity

docker exec B3_jour ping -c 2 B3_soir     # ✅ résolution par nom : DNS intégré de Docker
docker network inspect mon_reseau --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

Connexion à chaud d'un conteneur existant :
```bash
docker run -d --name visiteur alpine:3.20 sleep infinity     # sur le bridge par défaut
docker exec visiteur ping -c 1 -W 1 B3_jour                  # ❌
docker network connect mon_reseau visiteur
docker exec visiteur ping -c 1 B3_jour                       # ✅
docker network disconnect mon_reseau visiteur && docker rm -f visiteur
```

## Étape 3 : Partager des données via un volume anonyme (« temporaire »)

On recrée les deux conteneurs (un nom de conteneur est unique) :
```bash
docker rm -f B3_jour B3_soir

# B3_jour reçoit un volume anonyme monté sur /partage (Docker lui donne un nom aléatoire)
docker run -d --name B3_jour --network mon_reseau -v /partage alpine:3.20 sleep infinity
# B3_soir récupère les volumes de B3_jour
docker run -d --name B3_soir --network mon_reseau --volumes-from B3_jour alpine:3.20 sleep infinity

docker exec B3_jour sh -c 'echo "Bonjour du jour" > /partage/fichier.txt'
docker exec B3_soir cat /partage/fichier.txt                 # Bonjour du jour
docker exec B3_soir sh -c 'echo "Bonsoir" >> /partage/fichier.txt'
docker exec B3_jour cat /partage/fichier.txt                 # les deux lignes
```

Vérifiez le caractère « temporaire » :
```bash
docker volume ls                                             # notez le volume au nom aléatoire (64 caractères)
docker inspect B3_jour --format '{{range .Mounts}}{{.Name}} -> {{.Destination}}{{end}}'
docker rm -f -v B3_soir B3_jour                              # -v : supprime aussi les volumes anonymes
docker volume ls                                             # le volume a disparu : les données sont perdues
```

## Étape 4 : Partager des données via un volume nommé (persistant)

```bash
docker volume create mon_volume_persistant
docker volume inspect mon_volume_persistant --format '{{.Mountpoint}}'   # où sont réellement stockées les données

docker run -d --name B3_jour --network mon_reseau -v mon_volume_persistant:/mnt alpine:3.20 sleep infinity
docker run -d --name B3_soir --network mon_reseau -v mon_volume_persistant:/mnt:ro alpine:3.20 sleep infinity

docker exec B3_jour sh -c 'echo "Bonjour" > /mnt/fichier.txt'
docker exec B3_soir cat /mnt/fichier.txt                     # Bonjour
docker exec B3_soir sh -c 'echo test > /mnt/x.txt'           # ❌ Read-only file system (option :ro)
```

Supprimez les conteneurs et prouvez que les données ont survécu :
```bash
docker rm -f -v B3_jour B3_soir                              # même avec -v, un volume NOMMÉ n'est pas supprimé
docker volume ls | grep mon_volume_persistant
docker run --rm -v mon_volume_persistant:/mnt alpine:3.20 cat /mnt/fichier.txt   # Bonjour
```

## Étape 5 : Un cas réel – une base de données qui survit à sa suppression

```bash
docker run -d --name pg --network mon_reseau -e POSTGRES_PASSWORD=secret \
  -v pg_data:/var/lib/postgresql/data postgres:16-alpine
until docker exec pg pg_isready -h 127.0.0.1 -U postgres -q; do sleep 1; done      # attendre que la base accepte les connexions
docker exec pg psql -U postgres -c "CREATE TABLE eleves(nom text); INSERT INTO eleves VALUES ('B3 jour'),('B3 soir');"

docker rm -f pg                                              # on "perd" le serveur...
docker run -d --name pg --network mon_reseau -e POSTGRES_PASSWORD=secret \
  -v pg_data:/var/lib/postgresql/data postgres:16-alpine
until docker exec pg pg_isready -h 127.0.0.1 -U postgres -q; do sleep 1; done
docker exec pg psql -U postgres -c "SELECT * FROM eleves;"   # ...les données sont toujours là
```

Un client sur le même réseau joint la base **par son nom** :
```bash
docker run --rm --network mon_reseau -e PGPASSWORD=secret postgres:16-alpine \
  psql -h pg -U postgres -c "SELECT count(*) FROM eleves;"
```

## Étape 6 : Nettoyage

```bash
docker rm -f pg
docker volume rm mon_volume_persistant pg_data
docker network rm mon_reseau
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| DNS du réseau personnalisé | `docker exec B3_jour ping -c1 B3_soir` | réponse, alors qu'elle échoue sur le bridge par défaut |
| Volume anonyme partagé | `docker exec B3_soir cat /partage/fichier.txt` | `Bonjour du jour` |
| Volume anonyme supprimé | `docker volume ls` après `rm -v` | le volume aléatoire a disparu |
| Volume nommé persistant | `docker run --rm -v mon_volume_persistant:/mnt alpine cat /mnt/fichier.txt` | `Bonjour` après suppression des conteneurs |
| Base persistante | `SELECT * FROM eleves` après recréation de `pg` | 2 lignes |

## Synthèse à compléter (compte-rendu)

| | Volume anonyme | Volume nommé | Bind mount (TP 2.7) |
|---|---|---|---|
| Syntaxe `-v` | `-v /chemin` | `-v nom:/chemin` | `-v /dossier/hote:/chemin` |
| Créé par | ? | ? | ? |
| Supprimé par `docker rm -v` | ? | ? | ? |
| Usage typique | ? | ? | ? |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| `bad address 'B3_soir'` | conteneurs sur le bridge **par défaut** | les mettre sur un réseau créé avec `docker network create` |
| `Conflict. The container name "/B3_jour" is already in use` | nom déjà pris par un conteneur arrêté | `docker rm -f B3_jour` |
| `volume is in use` au `docker volume rm` | un conteneur (même arrêté) l'utilise encore | `docker ps -a --filter volume=<nom>` puis `docker rm` |
| Le conteneur `ubuntu` s'arrête aussitôt | pas de processus au premier plan | ajouter `sleep infinity` ou utiliser `-it` |

## 🎯 Défi (15 min)

Créez un réseau `backend` et un réseau `frontend`. Placez un conteneur `api` (alpine) sur les **deux** réseaux, un conteneur `db` (alpine) uniquement sur `backend` et un conteneur `web` (alpine) uniquement sur `frontend`. Prouvez que `web` joint `api` mais **pas** `db`, et que `api` joint les deux.

<details>
<summary>Solution</summary>

```bash
docker network create backend && docker network create frontend
docker run -d --name db  --network backend  alpine:3.20 sleep infinity
docker run -d --name web --network frontend alpine:3.20 sleep infinity
docker run -d --name api --network backend  alpine:3.20 sleep infinity
docker network connect frontend api
docker exec web ping -c1 api          # ✅
docker exec web ping -c1 -W1 db       # ❌ bad address
docker exec api ping -c1 db           # ✅
docker rm -f db web api && docker network rm backend frontend
```
C'est le principe de l'**isolation 3 tiers** approfondi au TP 2.7 : la base n'est jamais joignable depuis la couche exposée.
</details>

➡️ Pour aller plus loin : TP 2.7 (bind mounts, tmpfs, sauvegarde/restauration de volume, isolation réseau).
