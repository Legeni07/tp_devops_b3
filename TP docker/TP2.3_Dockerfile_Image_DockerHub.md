# TP 2.3 : Dockerfile, image, conteneur et Docker Hub – le site restaurant

**Objectif :** écrire un premier `Dockerfile`, construire une image, comprendre ses **couches**, lancer un conteneur à partir de cette image, puis publier l'image sur **Docker Hub** pour qu'elle soit utilisable sur n'importe quelle machine.

**Durée :** 1h – **Prérequis :** TP 2.2, un compte GitHub et un compte Docker Hub – **Application :** site statique [restaurant](https://github.com/Legeni07/restaurant)

---

## Étape 1 : Préparer les comptes

1. **GitHub** : créez un compte sur [github.com](https://github.com) si vous n'en avez pas, puis **forkez** le dépôt [Legeni07/restaurant](https://github.com/Legeni07/restaurant) (vous en aurez besoin au TP 3.2).
2. **Docker Hub** : créez un compte sur [hub.docker.com](https://hub.docker.com), puis un **jeton d'accès** : *Account settings → Personal access tokens → Generate new token* (droits *Read & Write*). Copiez-le dans un gestionnaire de mots de passe : il ne sera plus affiché.

> ⚠️ On n'utilise **jamais** son mot de passe Docker Hub dans un terminal partagé ni dans un script : un jeton se révoque en un clic, un mot de passe compromis donne accès à tout le compte.

## Étape 2 : Écrire le Dockerfile

```bash
mkdir -p ~/legeni_static_website && cd ~/legeni_static_website
```

Créez le fichier `Dockerfile` (sans extension) :
```dockerfile
FROM ubuntu:24.04
LABEL maintainer="Legeni"
RUN apt-get update
RUN apt-get install -y nginx git
RUN rm -Rf /var/www/html/*
RUN git clone https://github.com/Legeni07/restaurant.git /var/www/html/
EXPOSE 80
ENTRYPOINT ["/usr/sbin/nginx", "-g", "daemon off;"]
```

Pour chaque ligne, notez dans votre compte-rendu **ce qu'elle fait** et **à quel moment** elle s'exécute (pendant le *build* ou au *démarrage* du conteneur). Aide : [struture_dockerfile.md](struture_dockerfile.md).

Questions :
1. Pourquoi `DEBIAN_FRONTEND=noninteractive` ? *(essayez sans : le build reste bloqué sur une question de fuseau horaire)*
2. Pourquoi nginx est-il lancé avec `daemon off;` ? Que se passerait-il sans ?
3. `EXPOSE 80` publie-t-il le port sur la machine hôte ?

## Étape 3 : Construire l'image et observer ses couches

```bash
docker build -t legeni_static_website:v1 .
docker images legeni_static_website
docker history legeni_static_website:v1        # une ligne par instruction
docker image inspect legeni_static_website:v1 --format '{{len .RootFS.Layers}} couches'
```

Relancez exactement la même commande `docker build` : elle se termine en une seconde. Repérez les mots `CACHED` dans la sortie et expliquez.

Puis modifiez le `LABEL` (par exemple `maintainer="Legeni B3"`) et reconstruisez : **quelles étapes sont rejouées ?** Concluez sur l'ordre dans lequel il faut écrire les instructions d'un Dockerfile.

## Étape 4 : Lancer et tester le conteneur

```bash
docker run --name legeni_static_website -d -p 80:80 legeni_static_website:v1
docker ps
curl -s http://localhost | grep -i '<title>'
```
Ouvrez `http://localhost` (WSL) ou `http://<IP_de_votre_VM>` dans le navigateur.

> Si le port 80 est déjà occupé : `-p 8090:80` puis `http://localhost:8090`.

Prouvez que le site est **dans l'image** (et non sur votre disque) :
```bash
docker exec legeni_static_website ls /var/www/html
ls ~/legeni_static_website        # seul le Dockerfile est sur votre machine
```

## Étape 5 : Améliorer le Dockerfile (version 2)

La version 1 fonctionne mais crée une couche par `RUN` et laisse le cache `apt` dans l'image. Créez `Dockerfile.v2` :
```dockerfile
FROM ubuntu:22.04
LABEL maintainer="Legeni"
# Une seule couche : mise à jour + installation + nettoyage du cache apt
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends nginx git ca-certificates \
 && rm -rf /var/lib/apt/lists/* /var/www/html/*
RUN git clone --depth 1 https://github.com/Legeni07/restaurant.git /var/www/html/ \
 && rm -rf /var/www/html/.git
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
```bash
docker build -f Dockerfile.v2 -t legeni_static_website:v2 .
docker images legeni_static_website          # comparez les tailles v1 / v2
```

Questions :
1. Pourquoi faut-il faire `apt-get update` et `apt-get install` **dans le même `RUN`** ?
2. On a remplacé `ENTRYPOINT` par `CMD`. Testez `docker run --rm legeni_static_website:v2 cat /etc/os-release` puis la même chose avec `:v1`. Quelle différence ? *(on y revient en détail au TP 2.5)*

## Étape 6 : Publier l'image sur Docker Hub

```bash
export DOCKERHUB_ID=<votre_id_dockerhub>
read -s DOCKERHUB_TOKEN                     # collez le jeton (rien ne s'affiche), puis Entrée
echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_ID" --password-stdin

# Une image ne peut être poussée que si son nom est préfixé par votre identifiant
docker tag legeni_static_website:v2 $DOCKERHUB_ID/legeni_static_website:v2
docker push $DOCKERHUB_ID/legeni_static_website:v2
```

Sur Docker Hub, ouvrez **Repositories** : l'image `legeni_static_website` est présente avec le tag `v2`.

Preuve de portabilité : supprimez **toutes** les copies locales, puis relancez depuis le registre.
```bash
docker rm -f legeni_static_website
docker rmi legeni_static_website:v1 legeni_static_website:v2 $DOCKERHUB_ID/legeni_static_website:v2
docker run -d --name resto -p 80:80 $DOCKERHUB_ID/legeni_static_website:v2   # "Unable to find image locally" → pull
curl -sI http://localhost | head -1
docker rm -f resto
```

> 💡 Gardez le dossier `~/legeni_static_website` et le `Dockerfile` v1 : ils servent de point de départ au TP 2.6 (optimisation) et au TP 3.2 (build par Jenkins).

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Image construite | `docker images legeni_static_website` | tags `v1` et `v2`, `v2` plus légère |
| Couches | `docker history legeni_static_website:v1` | 1 ligne par instruction du Dockerfile |
| Site accessible | `curl -sI http://localhost` | `HTTP/1.1 200 OK` |
| Image publiée | page Docker Hub du dépôt | tag `v2` visible |
| Portabilité | `docker run` après `docker rmi` | l'image est téléchargée puis le site répond |

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| `denied: requested access to the resource is denied` au push | image non préfixée par votre ID, ou pas connecté | `docker tag ... $DOCKERHUB_ID/...` et `docker login` |
| Le build reste bloqué sur `Please select the geographic area` | installation interactive de `tzdata` | `DEBIAN_FRONTEND=noninteractive` |
| Le conteneur s'arrête aussitôt (`Exited (0)`) | nginx part en arrière-plan, PID 1 se termine | `daemon off;` |
| `fatal: could not read Username` dans `git clone` | URL du dépôt erronée ou dépôt privé | vérifier l'URL, dépôt public |
| Ancien contenu après un `git push` sur le site | la couche `git clone` est en **cache** | `docker build --no-cache` (et voir TP 2.6 : `COPY` plutôt que `git clone`) |

## 🎯 Défi (15 min)

Votre site restaurant doit afficher la mention « Version : v3 – <date du build> » en bas de la page d'accueil, **sans modifier le dépôt Git**. Écrivez un `Dockerfile.v3` qui reçoit la version par `--build-arg`, l'injecte dans `index.html` pendant le build, et poussez `v3` sur Docker Hub.

<details>
<summary>Solution</summary>

```dockerfile
FROM ubuntu:22.04
ARG VERSION=dev
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends nginx git ca-certificates \
 && rm -rf /var/lib/apt/lists/* /var/www/html/*
RUN git clone --depth 1 https://github.com/Legeni07/restaurant.git /var/www/html/ \
 && rm -rf /var/www/html/.git \
 && sed -i "s|</body>|<p style=\"text-align:center\">Version : ${VERSION} – $(date +%F)</p></body>|" /var/www/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
```bash
docker build -f Dockerfile.v3 --build-arg VERSION=v3 -t $DOCKERHUB_ID/legeni_static_website:v3 .
docker run --rm -d --name v3 -p 8093:80 $DOCKERHUB_ID/legeni_static_website:v3
curl -s localhost:8093 | grep Version ; docker rm -f v3
docker push $DOCKERHUB_ID/legeni_static_website:v3
```
</details>
