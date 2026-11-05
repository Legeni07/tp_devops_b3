# TP 2.9 : Registre privé, stratégie de tags et scan de sécurité

**Objectif :** publier ses images de façon **traçable** (quelle image contient quel commit ?), héberger un registre privé et bloquer les images vulnérables avant qu'elles n'arrivent en production.

**Durée :** 1h30 – **Prérequis :** TP 2.6, compte Docker Hub – **Application :** `app-fil-rouge/`

---

## Étape 1 : Une stratégie de tags

Un tag `latest` seul ne dit **rien** de ce qu'il y a dans l'image. En entreprise on pousse plusieurs tags pour la même image :

| Tag | Exemple | Usage |
|---|---|---|
| Version sémantique | `1.4.2` | Ce qu'on déploie en production |
| Commit Git court | `sha-3f9c2ab` | Traçabilité : retrouver le code exact |
| Numéro de build CI | `build-57` | Lien avec l'exécution Jenkins |
| `latest` | `latest` | Pratique en dev, **jamais** en prod |

```bash
cd app-fil-rouge
VERSION=1.0.0
SHA=$(git rev-parse --short HEAD)
docker build --build-arg APP_VERSION=$VERSION -t legeni-webapp:$VERSION .
docker tag legeni-webapp:$VERSION legeni-webapp:sha-$SHA
docker images legeni-webapp          # même IMAGE ID pour les deux tags
```

## Étape 2 : Pousser sur Docker Hub

1. Créez un **jeton d'accès** sur Docker Hub (*Account settings → Personal access tokens*) : on n'utilise jamais son mot de passe dans un script.
2. Connexion non interactive (méthode utilisée par Jenkins) :
	```bash
	export DOCKERHUB_ID=<votre_id>
	read -s DOCKERHUB_TOKEN
	echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_ID" --password-stdin
	```
3. Taguez et poussez :
	```bash
	for t in $VERSION sha-$SHA; do
	  docker tag legeni-webapp:$VERSION $DOCKERHUB_ID/legeni-webapp:$t
	  docker push $DOCKERHUB_ID/legeni-webapp:$t
	done
	```
4. Récupérez le **digest** : c'est l'identifiant immuable de l'image (un tag peut être déplacé, un digest non).
	```bash
	docker inspect --format '{{index .RepoDigests 0}}' $DOCKERHUB_ID/legeni-webapp:$VERSION
	```

## Étape 3 : Héberger un registre privé

```bash
docker run -d --name registry -p 5000:5000 -v registry_data:/var/lib/registry --restart unless-stopped registry:3
docker tag legeni-webapp:$VERSION localhost:5000/legeni-webapp:$VERSION
docker push localhost:5000/legeni-webapp:$VERSION
curl -s localhost:5000/v2/_catalog
curl -s localhost:5000/v2/legeni-webapp/tags/list
```

Testez le *pull* depuis zéro :
```bash
docker rmi localhost:5000/legeni-webapp:$VERSION
docker pull localhost:5000/legeni-webapp:$VERSION
```

> 💡 Depuis une autre machine, le registre doit être servi en HTTPS, ou déclaré dans `/etc/docker/daemon.json` → `"insecure-registries": ["<IP>:5000"]` (acceptable uniquement en lab).

## Étape 4 : Scanner les vulnérabilités avec Trivy

Trivy est lancé lui-même dans un conteneur, rien à installer :

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy_cache:/root/.cache aquasec/trivy:0.57.1 \
  image --severity HIGH,CRITICAL legeni-webapp:$VERSION
```

Comparez avec une vieille image :
```bash
docker pull python:3.6-alpine
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy_cache:/root/.cache aquasec/trivy:0.57.1 \
  image --severity CRITICAL python:3.6-alpine | tail -20
```
Question : pourquoi l'image `python:3.6-alpine` de l'ancien projet final est-elle à proscrire ?

## Étape 5 : Faire de la sécurité un *quality gate*

`--exit-code 1` fait échouer la commande si des vulnérabilités sont trouvées : c'est ce qui fera passer le pipeline Jenkins **au rouge** (TP 3.7).

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
  aquasec/trivy:0.57.1 image --severity CRITICAL --ignore-unfixed --exit-code 1 legeni-webapp:$VERSION
echo "Code retour : $?"
```

## Étape 6 : Analyser le Dockerfile (lint)

```bash
docker run --rm -i hadolint/hadolint:v2.12.0 < Dockerfile
docker run --rm -i hadolint/hadolint:v2.12.0 < ../legeni_static_website/Dockerfile
```
Corrigez au moins deux avertissements du Dockerfile `legeni_static_website` (ex. `DL3008` versions de paquets, `DL3009` nettoyage du cache apt).

---

## ✅ Vérifications

- [ ] Image visible sur Docker Hub avec au moins 2 tags (`version` et `sha-…`)
- [ ] `curl localhost:5000/v2/_catalog` liste `legeni-webapp`
- [ ] Rapport Trivy de `legeni-webapp` : aucune vulnérabilité CRITICAL corrigeable
- [ ] `hadolint` sans erreur sur le Dockerfile de l'application

## 🎯 Défi

Écrivez un script `release.sh <version>` qui : construit l'image avec tests (`--target test` puis `runtime`), la scanne (échec si CRITICAL), la tague `version` + `sha-<commit>` et la pousse. Ce script préfigure le pipeline Jenkins du TP 3.5.

<details>
<summary>Solution</summary>

```bash
#!/bin/bash
set -euo pipefail
VERSION=${1:?usage: $0 <version>}
IMAGE=${DOCKERHUB_ID:?définir DOCKERHUB_ID}/legeni-webapp
SHA=sha-$(git rev-parse --short HEAD)

docker build --target test -t legeni-webapp:test .
docker build --build-arg APP_VERSION="$VERSION" -t "$IMAGE:$VERSION" -t "$IMAGE:$SHA" .
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
  aquasec/trivy:0.57.1 image --severity CRITICAL --ignore-unfixed --exit-code 1 "$IMAGE:$VERSION"
docker push "$IMAGE:$VERSION"
docker push "$IMAGE:$SHA"
```
</details>
