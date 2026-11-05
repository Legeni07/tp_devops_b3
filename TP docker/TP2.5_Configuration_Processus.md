# TP 2.5 : Configurer un conteneur – variables, ENTRYPOINT/CMD et processus n°1

**Objectif :** construire **une seule image** qui fonctionne dans tous les environnements (dev, staging, prod) grâce à la configuration par variables d'environnement, maîtriser la différence entre `ARG`/`ENV` et `ENTRYPOINT`/`CMD`, et comprendre pourquoi un conteneur mal écrit met 10 secondes à s'arrêter.

**Durée :** 1h30 – **Prérequis :** TP 2.3 – **Application :** `app-fil-rouge/` (à la racine du dépôt)

> Principe (méthode *12-factor app*) : **le code est dans l'image, la configuration est dans l'environnement.** On ne reconstruit pas une image pour changer une URL, un mot de passe ou une couleur.

---

## Étape 1 : Configurer l'application fil rouge par l'environnement

```bash
cd app-fil-rouge
docker build -t legeni-webapp:1.0.0 --build-arg APP_VERSION=1.0.0 .

docker run -d --name dev  -p 5001:5000 legeni-webapp:1.0.0
docker run -d --name prod -p 5002:5000 -e APP_ENV=prod -e APP_COLOR='#d1242f' legeni-webapp:1.0.0
curl -s localhost:5001/api/info; echo
curl -s localhost:5002/api/info; echo
```
Ouvrez les deux URL dans le navigateur : **même image**, deux bannières de couleur différente.

Regroupez la configuration dans un fichier (jamais versionné s'il contient des secrets) :
```bash
cat > staging.env <<'EOF'
APP_ENV=staging
APP_COLOR=#9a6700
EOF
docker run -d --name staging -p 5003:5000 --env-file staging.env legeni-webapp:1.0.0
docker exec staging env | grep APP_
docker rm -f dev prod staging && rm staging.env
```

Question : lisez le début de `app.py`. Quelles variables l'application lit-elle, et quelles valeurs prend-elle si on ne les fournit pas ?

## Étape 2 : `ARG` (build) ou `ENV` (exécution) ?

Créez un dossier de travail `~/tp25` et le fichier `Dockerfile.args` :
```dockerfile
FROM alpine:3.20
ARG BUILD_VERSION=dev
ENV APP_MESSAGE="Bonjour"
RUN echo "Construit en version ${BUILD_VERSION}" > /version.txt
CMD ["sh", "-c", "cat /version.txt; echo \"BUILD_VERSION=$BUILD_VERSION APP_MESSAGE=$APP_MESSAGE\""]
```
```bash
mkdir -p ~/tp25 && cd ~/tp25          # (créez Dockerfile.args dans ce dossier)
docker build -f Dockerfile.args --build-arg BUILD_VERSION=2.0 -t demo:args .
docker run --rm demo:args
docker run --rm -e APP_MESSAGE="Salut" demo:args
docker run --rm -e BUILD_VERSION=9.9 demo:args
```

Complétez le tableau :

| | `ARG` | `ENV` |
|---|---|---|
| Disponible pendant le `docker build` | ? | ? |
| Disponible dans le conteneur | ? | ? |
| Modifiable au `docker run` | ? | ? |
| Visible dans `docker history` / `docker inspect` | ? | ? |

> ⚠️ Un `ARG` ou un `ENV` contenant un mot de passe reste **lisible** dans l'image (`docker history --no-trunc`, `docker inspect`). Les secrets se passent **au lancement** (`-e`, `--env-file`, secrets Compose/Kubernetes), jamais au build.

## Étape 3 : `CMD`, `ENTRYPOINT` et leur combinaison

`Dockerfile.cmd`
```dockerfile
FROM alpine:3.20
CMD ["echo", "Je suis la commande par défaut"]
```
`Dockerfile.entry`
```dockerfile
FROM alpine:3.20
ENTRYPOINT ["echo", "Préfixe fixe :"]
CMD ["argument par défaut"]
```
```bash
docker build -f Dockerfile.cmd   -t demo:cmd .
docker build -f Dockerfile.entry -t demo:entry .

docker run --rm demo:cmd                         # commande par défaut
docker run --rm demo:cmd ls /                    # les arguments REMPLACENT le CMD
docker run --rm demo:entry                       # ENTRYPOINT + CMD
docker run --rm demo:entry "mon argument"        # les arguments remplacent seulement le CMD
docker run --rm --entrypoint ls demo:entry /     # on force un autre ENTRYPOINT
```

Règle à retenir : **ENTRYPOINT = le programme**, **CMD = ses arguments par défaut**. C'est pour cela qu'au TP 2.3, `docker run legeni_static_website:v1 cat /etc/os-release` passait `cat ...` en argument à nginx.

## Étape 4 : Forme *exec* ou forme *shell* – le processus n°1 et les signaux

Dans un conteneur, le processus principal a le **PID 1**. `docker stop` lui envoie `SIGTERM`, attend 10 secondes, puis envoie `SIGKILL`.

`app.sh` (un « serveur » qui sait s'arrêter proprement) :
```sh
#!/bin/sh
trap 'echo "SIGTERM reçu : fermeture propre des connexions"; exit 0' TERM
echo "Démarrage (PID $$)"
while true; do sleep 1; done
```
`Dockerfile.shell` (forme shell : une chaîne de caractères)
```dockerfile
FROM alpine:3.20
COPY app.sh /app.sh
RUN chmod +x /app.sh
CMD /app.sh | cat
```
`Dockerfile.exec` (forme exec : un tableau JSON)
```dockerfile
FROM alpine:3.20
COPY app.sh /app.sh
RUN chmod +x /app.sh
CMD ["/app.sh"]
```
```bash
docker build -f Dockerfile.shell -t demo:shell .
docker build -f Dockerfile.exec  -t demo:exec .

docker run -d --name shell demo:shell && docker run -d --name exec demo:exec
docker exec shell ps -o pid,args       # PID 1 = /bin/sh -c ..., le script n'est PAS le PID 1
docker exec exec ps -o pid,args       # PID 1 = /app.sh

time docker stop shell                 # ~10 s : le signal n'atteint pas le script → SIGKILL
time docker stop exec                  # ~1 s : le script reçoit SIGTERM et s'arrête proprement
docker logs exec | tail -1            # "SIGTERM reçu : fermeture propre..."
docker logs shell | tail -1            # pas de message : arrêt brutal
docker rm shell exec
```

Questions :
1. Pourquoi un arrêt brutal (`SIGKILL`) est-il dangereux pour une base de données ou une application qui traite des requêtes ?
2. Quelle forme utilise le `CMD` du Dockerfile de `app-fil-rouge` ? Vérifiez : `docker run -d --name webtest legeni-webapp:1.0.0 && docker exec webtest ps aux | head -3; time docker stop webtest; docker rm webtest`.

## Étape 5 : Le script d'entrée (*entrypoint script*)

Cas réel : avant de lancer l'application, on veut vérifier la configuration et préparer un fichier. On écrit un script d'entrée qui **termine par `exec "$@"`**, pour que l'application devienne le PID 1.

`entrypoint.sh`
```sh
#!/bin/sh
set -e
: "${APP_ENV:?La variable APP_ENV est obligatoire}"          # échoue si absente
echo "[entrypoint] environnement = $APP_ENV"
echo "<h1>Environnement : $APP_ENV</h1>" > /usr/share/nginx/html/index.html
exec "$@"                                                    # remplace le shell par la commande (CMD)
```
`Dockerfile.nginx`
```dockerfile
FROM nginx:1.27-alpine
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```
```bash
docker build -f Dockerfile.nginx -t demo:entrypoint .
docker run --rm demo:entrypoint                              # ❌ "APP_ENV: La variable APP_ENV est obligatoire"
docker run -d --name ep -p 8095:80 -e APP_ENV=staging demo:entrypoint
curl -s localhost:8095                                       # <h1>Environnement : staging</h1>
docker exec ep ps -o pid,args | head -3                      # PID 1 = nginx, grâce à exec
docker run --rm -e APP_ENV=test demo:entrypoint nginx -v     # le CMD reste remplaçable
docker rm -f ep
```

> 💡 C'est exactement ce que font les images officielles `postgres`, `mysql` ou `odoo` : leur `docker-entrypoint.sh` initialise la base selon les variables `POSTGRES_PASSWORD`, `MYSQL_DATABASE`… puis fait `exec "$@"`. Allez lire celui de PostgreSQL : `docker run --rm --entrypoint cat postgres:16-alpine /usr/local/bin/docker-entrypoint.sh | tail -20`.

## Étape 6 : Nettoyage

```bash
docker rmi demo:args demo:cmd demo:entry demo:shell demo:exec demo:entrypoint
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Une image, plusieurs configurations | `curl localhost:5001/api/info` et `:5002` | `"env":"dev"` et `"env":"prod"` |
| ARG absent à l'exécution | `docker run --rm demo:args` | `BUILD_VERSION=` vide, mais `/version.txt` contient `2.0` |
| ENTRYPOINT + CMD | `docker run --rm demo:entry "x"` | `Préfixe fixe : x` |
| Forme exec | `time docker stop exec` | environ 1 s, message `SIGTERM reçu` |
| Script d'entrée | `docker run --rm demo:entrypoint` | échec explicite sur `APP_ENV` |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| `docker stop` met toujours 10 s | forme shell (`CMD cmd args`) ou script sans `exec` | forme exec `CMD ["cmd","args"]`, `exec "$@"` dans le script |
| `exec /entrypoint.sh: no such file or directory` alors que le fichier existe | fins de ligne Windows (CRLF) dans le script | `sed -i 's/\r$//' entrypoint.sh` (ou réglage de l'éditeur sur LF) |
| `permission denied` sur le script | pas de droit d'exécution | `RUN chmod +x` ou `COPY --chmod=755` |
| La variable n'est pas remplacée dans `CMD ["echo", "$VAR"]` | la forme exec ne lance pas de shell | `CMD ["sh", "-c", "echo $VAR"]` |
| Mot de passe visible dans `docker history` | secret passé en `ARG`/`ENV` dans le Dockerfile | le passer au `run` (`-e`, `--env-file`, secrets) |

## 🎯 Défi (20 min)

Créez une image `demo:wait` dont le script d'entrée **attend qu'un hôte et un port soient joignables** (variables `WAIT_HOST` et `WAIT_PORT`, 30 tentatives maximum) avant de lancer la commande. Testez-la avec un conteneur Redis sur un réseau dédié : `docker run --rm --network ... -e WAIT_HOST=redis -e WAIT_PORT=6379 demo:wait echo "Redis est prêt"`.

<details>
<summary>Solution</summary>

`wait.sh`
```sh
#!/bin/sh
set -e
if [ -n "$WAIT_HOST" ]; then
  for i in $(seq 1 30); do
    nc -z "$WAIT_HOST" "${WAIT_PORT:-80}" && break
    echo "[wait] $WAIT_HOST:${WAIT_PORT:-80} indisponible ($i/30)"; sleep 1
  done
  nc -z "$WAIT_HOST" "${WAIT_PORT:-80}" || { echo "[wait] abandon"; exit 1; }
fi
exec "$@"
```
```dockerfile
FROM alpine:3.20
COPY --chmod=755 wait.sh /wait.sh
ENTRYPOINT ["/wait.sh"]
CMD ["echo", "prêt"]
```
```bash
docker build -f Dockerfile.wait -t demo:wait .
docker network create tp25
docker run --rm --network tp25 -e WAIT_HOST=redis -e WAIT_PORT=6379 demo:wait echo "Redis est prêt" &
sleep 5; docker run -d --name redis --network tp25 redis:7.4-alpine
wait; docker rm -f redis; docker network rm tp25
```
En Compose, on obtient le même résultat plus proprement avec un `healthcheck` et `depends_on: condition: service_healthy` (TP 2.8).
</details>
