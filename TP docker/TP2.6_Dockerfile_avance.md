# TP 2.6 : Dockerfile avancé – optimiser, sécuriser, tester dans le build

**Objectif :** passer d'un Dockerfile « qui marche » (TP2) à un Dockerfile **de production** : petite image, build rapide grâce au cache, utilisateur non-root, healthcheck et tests exécutés pendant le build.

**Durée :** 1h30 – **Prérequis :** TP 2.2 à 2.5 – **Application :** `app-fil-rouge/` (à la racine du dépôt)

---

## Étape 1 : Mesurer le point de départ

1. Reconstruisez l'image du TP 2.3 (site restaurant sur `ubuntu:22.04`) et notez sa taille :
	```bash
	cd legeni_static_website
	docker build -t restaurant:ubuntu .
	docker images restaurant
	docker history restaurant:ubuntu      # une ligne = une couche
	```
2. Questions :
	- Combien de couches ? Laquelle est la plus lourde ?
	- Pourquoi `git` et la liste des paquets `apt` se retrouvent-ils dans l'image finale alors qu'ils ne servent plus à l'exécution ?

## Étape 2 : Choisir la bonne image de base

1. Créez un `Dockerfile.alpine` pour le même site, basé sur nginx officiel :
	```dockerfile
	FROM nginx:1.27-alpine
	LABEL maintainer="Legeni"
	COPY restaurant/ /usr/share/nginx/html/
	EXPOSE 80
	```
2. Construisez et comparez :
	```bash
	git clone https://github.com/Legeni07/restaurant.git
	docker build -f Dockerfile.alpine -t restaurant:alpine .
	docker images restaurant
	```
3. ✅ L'image `alpine` doit être **nettement plus petite** (ordre de grandeur : ~75 Mo contre ~250 Mo, dont l'essentiel est constitué des images du site). Notez les deux tailles dans votre compte-rendu.

> 💡 Règle : on copie le code dans l'image avec `COPY` (reproductible, versionné) plutôt que de faire un `git clone` dans le `RUN` (dépend du réseau et de l'état du dépôt au moment du build).

## Étape 3 : Le cache de build et l'ordre des instructions

1. Placez-vous dans `app-fil-rouge/` et observez le `Dockerfile` : `requirements.txt` est copié **avant** le reste du code.
2. Construisez deux fois et chronométrez :
	```bash
	cd app-fil-rouge
	time docker build -t legeni-webapp:cache .
	echo "# commentaire" >> app.py
	time docker build -t legeni-webapp:cache .     # les couches pip sont "CACHED"
	git checkout app.py
	```
3. Expérience : déplacez `COPY . .` **avant** `RUN pip install` dans une copie du Dockerfile, modifiez `app.py`, reconstruisez. Que constatez-vous sur la durée ? Expliquez.

## Étape 4 : Le fichier `.dockerignore`

1. Affichez `app-fil-rouge/.dockerignore`.
2. Créez un gros fichier inutile et observez la taille du *contexte* envoyé au démon :
	```bash
	dd if=/dev/zero of=gros_fichier.bin bs=1M count=200
	docker build -t legeni-webapp:ctx . 2>&1 | grep -i "transferring context"
	echo "gros_fichier.bin" >> .dockerignore
	docker build -t legeni-webapp:ctx . 2>&1 | grep -i "transferring context"
	rm gros_fichier.bin && git checkout .dockerignore
	```

## Étape 5 : Le build multi-étapes (multi-stage)

Le `Dockerfile` de l'application contient **3 étapes** : `base`, `test`, `runtime`.

1. Lancez uniquement l'étape de test (lint flake8 + tests pytest) :
	```bash
	docker build --target test -t legeni-webapp:test .
	```
2. Cassez volontairement un test (par exemple changez `"ok"` en `"ko"` dans `/health` de `app.py`), relancez : **le build doit échouer**. Remettez le code d'origine.
3. Construisez l'image finale en injectant la version :
	```bash
	docker build --build-arg APP_VERSION=1.0.0 -t legeni-webapp:1.0.0 .
	docker images legeni-webapp
	```
	La cible `runtime` ne contient ni pytest ni flake8 : comparez la taille avec `legeni-webapp:test`.

> 💡 C'est ce mécanisme que Jenkins utilisera (TP 3.5) : **si les tests échouent, aucune image n'est produite.**

## Étape 6 : Sécurité – utilisateur non-root

```bash
docker run --rm legeni-webapp:1.0.0 id
docker run --rm --entrypoint id restaurant:ubuntu   # --entrypoint remplace nginx par la commande id
```
✅ L'application tourne avec l'utilisateur `appuser` (uid 10001), le site ubuntu tourne en `root`. Expliquez le risque.

## Étape 7 : HEALTHCHECK

```bash
docker run -d --name webapp -p 5000:5000 legeni-webapp:1.0.0
watch -n2 docker ps        # la colonne STATUS passe de "health: starting" à "healthy"
docker inspect --format '{{json .State.Health}}' webapp | python3 -m json.tool
curl http://localhost:5000/api/info
docker rm -f webapp
```

---

## ✅ Vérifications

| Contrôle | Commande | Résultat attendu |
|---|---|---|
| Tests dans le build | `docker build --target test .` | `6 passed` |
| Version injectée | `docker run --rm legeni-webapp:1.0.0 env \| grep APP_VERSION` | `APP_VERSION=1.0.0` |
| Non-root | `docker run --rm legeni-webapp:1.0.0 whoami` | `appuser` |
| Santé | `docker ps` | `(healthy)` |

## 🎯 Défi

Réécrivez le Dockerfile du site restaurant en **multi-stage** : une première étape `alpine/git` clone le dépôt, la seconde (`nginx:1.27-alpine`) ne récupère que les fichiers HTML/CSS/images avec `COPY --from=`. Ajoutez un `HEALTHCHECK` avec `wget`.

<details>
<summary>Solution</summary>

```dockerfile
FROM alpine/git:v2.52.0 AS source
RUN git clone --depth 1 https://github.com/Legeni07/restaurant.git /src && rm -rf /src/.git

FROM nginx:1.27-alpine
LABEL maintainer="Legeni"
COPY --from=source /src/ /usr/share/nginx/html/
EXPOSE 80
HEALTHCHECK --interval=15s --timeout=3s CMD wget -q -O /dev/null http://127.0.0.1/ || exit 1
```
</details>
