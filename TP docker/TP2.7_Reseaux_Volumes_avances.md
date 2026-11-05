# TP 2.7 : Réseaux et stockage en profondeur

**Objectif :** comprendre *pourquoi* deux conteneurs se voient (ou pas), et choisir le bon type de stockage : volume nommé, bind mount ou tmpfs. Savoir sauvegarder et restaurer les données d'un volume.

**Durée :** 1h30 – **Prérequis :** TP 3 (volumes et réseau)

---

## Partie A – Réseaux

### Étape 1 : Le réseau `bridge` par défaut n'a pas de DNS

```bash
docker run -d --name a1 alpine sleep infinity
docker run -d --name a2 alpine sleep infinity
docker exec a1 ping -c1 a2                 # ❌ bad address 'a2'
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' a2
docker exec a1 ping -c1 <IP_de_a2>         # ✅ fonctionne par IP
docker rm -f a1 a2
```

### Étape 2 : Un réseau personnalisé apporte la résolution de noms

```bash
docker network create front
docker run -d --name a1 --network front alpine sleep infinity
docker run -d --name a2 --network front alpine sleep infinity
docker exec a1 ping -c1 a2                 # ✅ résolu par le DNS interne 127.0.0.11
docker exec a1 cat /etc/resolv.conf
```

### Étape 3 : Isolation entre réseaux (architecture 3 tiers)

On reproduit l'architecture de l'application fil rouge : `proxy` ↔ `web` ↔ `redis`. Le proxy **ne doit pas** pouvoir joindre Redis.

```bash
docker network create back
docker run -d --name redis --network back redis:7.4-alpine
docker run -d --name web --network back alpine sleep infinity
docker network connect front web           # web est sur les 2 réseaux
docker run -d --name proxy --network front alpine sleep infinity

docker exec web   ping -c1 redis           # ✅
docker exec proxy ping -c1 web             # ✅
docker exec proxy ping -c1 redis           # ❌ isolé
docker network inspect back --format '{{range .Containers}}{{.Name}} {{end}}'
```

Faites un schéma des 3 conteneurs et des 2 réseaux dans votre compte-rendu.

### Étape 4 : Publication de ports

```bash
docker run -d --name n1 -p 8081:80 nginx:1.27-alpine           # toutes les interfaces
docker run -d --name n2 -p 127.0.0.1:8082:80 nginx:1.27-alpine # uniquement en local
docker run -d --name n3 -P nginx:1.27-alpine                    # port hôte aléatoire
docker port n3
ss -lntp | grep -E '8081|8082'
```
Question : lequel de ces modes choisir pour une base de données qui ne doit pas être exposée sur Internet ?

```bash
docker rm -f a1 a2 web redis proxy n1 n2 n3 && docker network rm front back
```

## Partie B – Stockage

### Étape 5 : Bind mount – développer sans reconstruire l'image

```bash
mkdir -p ~/site && echo "<h1>Version 1</h1>" > ~/site/index.html
docker run -d --name dev -p 8083:80 -v ~/site:/usr/share/nginx/html:ro nginx:1.27-alpine
curl localhost:8083
echo "<h1>Version 2</h1>" > ~/site/index.html
curl localhost:8083                        # modifié à chaud, sans rebuild
docker exec dev sh -c 'echo test > /usr/share/nginx/html/x'   # ❌ Read-only file system
docker rm -f dev
```

### Étape 6 : tmpfs – données en mémoire, jamais écrites sur disque

```bash
docker run -d --name secret --tmpfs /run/secrets:size=1m alpine sleep infinity
docker exec secret sh -c 'echo motdepasse > /run/secrets/token && df -h /run/secrets'
docker restart secret
docker exec secret ls /run/secrets         # vide : tmpfs est volatile
docker rm -f secret
```

### Étape 7 : Sauvegarder et restaurer un volume

1. Créez des données dans un volume Redis :
	```bash
	docker volume create redis_data
	docker run -d --name redis -v redis_data:/data redis:7.4-alpine redis-server --appendonly yes
	docker exec redis redis-cli set cours devops
	docker exec redis redis-cli save
	```
2. Sauvegarde dans une archive sur l'hôte (conteneur jetable) :
	```bash
	docker run --rm -v redis_data:/source:ro -v "$PWD":/backup alpine \
	  tar czf /backup/redis_data.tar.gz -C /source .
	ls -lh redis_data.tar.gz
	```
3. Simulez une perte de données puis restaurez :
	```bash
	docker rm -f redis && docker volume rm redis_data
	docker volume create redis_data
	docker run --rm -v redis_data:/cible -v "$PWD":/backup alpine \
	  tar xzf /backup/redis_data.tar.gz -C /cible
	docker run -d --name redis -v redis_data:/data redis:7.4-alpine redis-server --appendonly yes
	docker exec redis redis-cli get cours      # ✅ "devops"
	docker rm -f redis
	```

---

## ✅ Vérifications

- [ ] Je sais expliquer pourquoi `ping a2` échoue sur le bridge par défaut.
- [ ] Le proxy ne joint pas Redis (isolation réseau démontrée).
- [ ] J'ai restauré la clé `cours` après suppression du volume.

## 🎯 Défi

Écrivez un script `backup_volume.sh <nom_volume>` qui produit `backup_<nom_volume>_<date>.tar.gz` et ne garde que les 7 dernières sauvegardes. (Il sera réutilisé dans le projet final, étape de sauvegarde avant déploiement.)

<details>
<summary>Solution</summary>

```bash
#!/bin/bash
set -euo pipefail
VOLUME=${1:?usage: $0 <nom_volume>}
DEST=${BACKUP_DIR:-$HOME/backups}
mkdir -p "$DEST"
FILE="backup_${VOLUME}_$(date +%Y%m%d_%H%M%S).tar.gz"
docker run --rm -v "${VOLUME}":/source:ro -v "${DEST}":/backup alpine tar czf "/backup/${FILE}" -C /source .
ls -1t "$DEST"/backup_"${VOLUME}"_*.tar.gz | tail -n +8 | xargs -r rm --
echo "Sauvegarde : ${DEST}/${FILE}"
```
</details>
