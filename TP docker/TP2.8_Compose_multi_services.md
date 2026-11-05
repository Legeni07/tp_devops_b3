# TP 2.8 : Docker Compose – une stack multi-services prête pour la production

**Objectif :** décrire toute une application (proxy + application + cache) dans **un seul fichier** versionné, avec healthchecks, ordre de démarrage, variables d'environnement, persistance et mise à l'échelle.

**Durée :** 2h – **Prérequis :** TP 2.6, TP 2.7 – **Application :** `app-fil-rouge/`

Architecture cible :

```
 navigateur ──:8080──▶ proxy (nginx) ──réseau frontend──▶ web ×N (Flask/gunicorn) ──réseau backend──▶ redis (volume redis_data)
```

---

## Étape 1 : Écrire le fichier `compose.yml`

Dans un dossier de travail `~/tp28`, copiez le code de l'application (`app.py`, `requirements*.txt`, `Dockerfile`, `tests/`) puis créez `compose.yml` **en complétant les `???`** :

```yaml
name: legeni

services:
  web:
    build: ???                       # construire depuis le Dockerfile du dossier courant
    environment:
      APP_ENV: dev
      REDIS_HOST: ???                # nom du service redis
    depends_on:
      redis:
        condition: ???               # attendre que redis soit "healthy"
    networks: [???, ???]

  redis:
    image: redis:7.4-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - ???:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5
    networks: [backend]

  proxy:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    networks: [frontend]

volumes:
  redis_data:

networks:
  frontend:
  backend:
```

Créez `nginx/default.conf` (le `resolver` permet à nginx de découvrir les nouveaux réplicas) :

```nginx
resolver 127.0.0.11 valid=10s ipv6=off;

server {
    listen 80;
    location / {
        set $backend http://web:5000;
        proxy_pass $backend;
        proxy_set_header Host $host;
    }
}
```

<details>
<summary>Solution des ???</summary>

`build: .` – `REDIS_HOST: redis` – `condition: service_healthy` – `networks: [backend, frontend]` – `redis_data:/data`.
La version complète se trouve dans `app-fil-rouge/docker-compose.yml`.
</details>

## Étape 2 : Valider puis démarrer

```bash
docker compose config            # valide la syntaxe et affiche le fichier "résolu"
docker compose up -d --build
docker compose ps                # web et redis doivent être (healthy)
curl -s localhost:8080/api/info
```

Ouvrez `http://<IP_VM>:8080` et rafraîchissez : le compteur de visites augmente.

## Étape 3 : Persistance

```bash
docker compose down              # supprime conteneurs et réseaux, PAS les volumes
docker compose up -d
curl -s localhost:8080 | grep -A1 visites   # le compteur continue
docker compose down -v           # -v supprime aussi les volumes
docker compose up -d
curl -s localhost:8080 | grep -A1 visites   # repart à 1
```

## Étape 4 : Mise à l'échelle et répartition de charge

```bash
docker compose up -d --scale web=3
for i in $(seq 10); do curl -s localhost:8080/api/info | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```
✅ Plusieurs `hostname` différents apparaissent, mais le compteur reste **unique** car il est partagé dans Redis.

Question : que se passerait-il si on supprimait `REDIS_HOST` (compteur en mémoire) avec 3 réplicas ?

## Étape 5 : Variables d'environnement et fichier `.env`

1. Créez un fichier `.env` à côté de `compose.yml` :
	```dotenv
	APP_VERSION=1.1.0
	APP_COLOR=#2da44e
	PROXY_PORT=8090
	```
2. Utilisez ces variables dans `compose.yml` : `${APP_VERSION:-dev}` en *build arg*, `${APP_COLOR}` dans `environment`, `"${PROXY_PORT:-8080}:80"` dans `ports`.
3. `docker compose config | grep -E 'APP_|8090'` puis `docker compose up -d --build`. La bannière devient verte et la version affiche 1.1.0.

> ⚠️ Le fichier `.env` contient souvent des secrets : ajoutez-le au `.gitignore` et versionnez plutôt un `.env.example`.

## Étape 6 : Exploitation au quotidien

```bash
docker compose logs -f web               # suivre les logs d'un service
docker compose exec redis redis-cli get visits
docker compose top
docker compose restart web
docker compose stop proxy && docker compose start proxy
```

## Étape 7 : Simuler une panne

```bash
docker compose stop redis
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/ready    # 503 : l'appli signale qu'elle n'est pas prête
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/health   # 200 : le processus est vivant
docker compose start redis
```
Expliquez la différence entre `/health` (liveness) et `/ready` (readiness) : elle sera essentielle dans Kubernetes (TP 4.9).

---

## ✅ Vérifications

- [ ] `docker compose ps` : tous les services `running`, `web` et `redis` `(healthy)`
- [ ] Le compteur survit à `docker compose down` mais pas à `down -v`
- [ ] Avec `--scale web=3`, au moins 2 hostnames différents répondent
- [ ] Le port et la couleur sont pilotés par le `.env`

## 🎯 Défi : déployer WordPress (ancien TP des diapos)

Écrivez un `compose.yml` WordPress + MariaDB avec : volumes nommés pour la base **et** pour `wp-content`, healthcheck sur MariaDB, `depends_on: condition: service_healthy`, et mots de passe dans un `.env`.

<details>
<summary>Solution</summary>

```yaml
services:
  db:
    image: mariadb:11.4
    environment:
      MARIADB_DATABASE: wordpress
      MARIADB_USER: wp
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      retries: 5
    restart: unless-stopped

  wordpress:
    image: wordpress:6-apache
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - wp_content:/var/www/html/wp-content
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  db_data:
  wp_content:
```
</details>
