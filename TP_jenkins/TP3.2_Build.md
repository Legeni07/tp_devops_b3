### Projet build – un premier job *freestyle* paramétré

**Objectif :** comprendre ce que fait Jenkins en construisant à la main un job *freestyle* qui récupère le code depuis GitHub, construit une image Docker et lance le site restaurant. Explorer l'espace de travail, la console, les paramètres, les variables d'environnement et les déclencheurs.

**Durée :** 1h – **Prérequis :** TP 3.1 (Jenkins avec accès à Docker), TP 2.3 (fork du dépôt `restaurant`)

> ℹ️ Avec l'image du TP 3.1, Docker est **déjà** utilisable par Jenkins. Les anciennes manipulations (`apt-get install docker.io` dans le conteneur, `chmod 666 /var/run/docker.sock`) ne sont plus nécessaires et sont **à proscrire** : elles disparaissent à la recréation du conteneur et ouvrent Docker à tous les utilisateurs de la machine.

---

1. Préparer le dépôt
	- Si ce n'est pas fait au TP 2.3, **forker** [Legeni07/restaurant](https://github.com/Legeni07/restaurant) sur votre compte GitHub.

2. Créer le job
	- **Tableau de bord** → **Nouvel item** → nom = **build** → **Projet free-style** → **OK**
	- **Description** : `Build et lancement du site restaurant`
	- Cocher **Ce build a des paramètres** → **Ajouter un paramètre** → **Paramètre String**, trois fois :

	| Nom | Valeur par défaut | Description |
	|---|---|---|
	| `IMAGE_NAME` | `restaurant` | nom de l'image |
	| `IMAGE_TAG` | `latest` | tag de l'image |
	| `Votre_ID_Github` | `Legeni07` | propriétaire du dépôt (mettez votre identifiant) |

3. Récupérer le code : **Gestion de code source** → **Git**
	- **Repository URL** : `https://github.com/${Votre_ID_Github}/restaurant.git`
	- **Branch Specifier** : `*/main` (vérifiez le nom de la branche par défaut de votre fork : `main` ou `master`)

	> 💡 C'est Jenkins qui clone le dépôt dans l'**espace de travail** (*workspace*) du job. On ne fait plus de `git clone` dans le Dockerfile (TP 2.6) : on **copie** le code récupéré par la CI, ce qui garantit que l'image contient exactement le commit testé.

4. Étape de build : **Build Steps** → **Ajouter une étape** → **Exécuter un script shell**
	```bash
	#!/bin/bash
	set -euo pipefail                       # arrêter le job à la première erreur

	echo "== Build n°${BUILD_NUMBER} du commit ${GIT_COMMIT:0:7} dans ${WORKSPACE}"

	# Dockerfile généré à partir du code récupéré par Jenkins
	cat > Dockerfile <<'EOF'
	FROM nginx:1.27-alpine
	LABEL maintainer="Legeni"
	COPY . /usr/share/nginx/html/
	EXPOSE 80
	EOF
	printf '.git\nDockerfile\n' > .dockerignore

	docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:build-${BUILD_NUMBER} .
	docker images ${IMAGE_NAME}

	# Relancer le conteneur : on supprime d'abord celui du build précédent
	docker rm -f legeni_website 2>/dev/null || true
	docker run --name legeni_website -d -p 8092:80 ${IMAGE_NAME}:${IMAGE_TAG}
	docker ps --filter name=legeni_website
	```
	- **Enregistrer**

5. Lancer et lire le résultat
	- **Lancer un build avec des paramètres** → **Build**
	- Cliquer sur le n° du build → **Console Output** : retrouvez le `git checkout`, chaque couche du `docker build`, le `docker run`
	- Ouvrir `http://localhost:8092` (ou `http://<IP_VM>:8092`) : le site restaurant s'affiche
	- **Espace de travail** (menu du job) : le code cloné, le `Dockerfile` et le `.dockerignore` générés y sont

6. Le rendre idempotent : relancez le build **une seconde fois**.
	- Il doit être vert. Supprimez temporairement la ligne `docker rm -f legeni_website ...` et relancez : quel message d'erreur obtenez-vous ? Remettez la ligne.
	- Question : pourquoi un job de CI doit-il pouvoir être relancé N fois avec le même résultat ?

7. Les variables d'environnement de Jenkins
	- Ouvrir `http://localhost:8080/env-vars.html` : liste des variables disponibles (`BUILD_NUMBER`, `JOB_NAME`, `WORKSPACE`, `GIT_COMMIT`, `BUILD_URL`…)
	- Ajoutez au script : `echo "Job ${JOB_NAME}, URL ${BUILD_URL}"` et relancez

8. Les déclencheurs (**Ce qui déclenche le build**)
	- Cocher **Scrutation de l'outil de gestion de version** (*Poll SCM*) avec le planning `H/2 * * * *` (vérification toutes les 2 minutes)
	- Sur GitHub, modifier le `<title>` de `index.html` de votre fork et *commit*
	- Attendre au plus 2 minutes : un build démarre seul (**Started by an SCM change**), le titre a changé sur `http://localhost:8092`
	- Décocher ensuite le Poll SCM (il sera remplacé par un **webhook** au TP 3.6, instantané et sans interrogation permanente)

---

##### ✅ Vérifications
- [ ] Le job `build` est vert et la console montre `docker build` puis `docker run`
- [ ] `http://localhost:8092` affiche le site restaurant
- [ ] `docker images restaurant` montre les tags `latest` et `build-<n>`
- [ ] Le job peut être relancé deux fois de suite sans erreur
- [ ] Un commit sur GitHub déclenche un build automatiquement (Poll SCM)

##### Correction des erreurs fréquentes
| Erreur | Cause | Solution |
|---|---|---|
| `docker: not found` | ancienne image Jenkins sans Docker CLI | utiliser l'image du TP 3.1 (`TP_jenkins/jenkins-docker/`) |
| `permission denied ... /var/run/docker.sock` | `DOCKER_GID` absent ou faux dans `.env` | refaire l'étape 2 du TP 3.1, puis `docker compose up -d --force-recreate` |
| `Couldn't find any revision to build` | branche `*/master` alors que le dépôt utilise `main` (ou l'inverse) | corriger le **Branch Specifier** |
| `Conflict. The container name "/legeni_website" is already in use` | conteneur du build précédent | `docker rm -f legeni_website` en début de script |
| `port is already allocated` | 8092 utilisé par un autre service | changer le port hôte (`-p 8094:80`) |

##### Questions (compte-rendu)
1. Où se trouve physiquement l'espace de travail du job ? *(indice : `docker exec jenkins ls /var/jenkins_home/workspace`)* Et sur quelle machine l'image est-elle construite ?
2. Citez trois inconvénients d'un job configuré uniquement dans l'interface (indice : historique des modifications, revue, reconstruction de Jenkins).

##### 🎯 Défi (10 min)
Ajoutez un paramètre **Choix** (*Choice Parameter*) `PORT` avec les valeurs `8092`, `8093`, `8094` et un paramètre **Booléen** `NETTOYER` : s'il est coché, le job supprime les images `restaurant:build-*` sauf les 3 plus récentes.

<details>
<summary>Solution</summary>

Dans le script : `-p ${PORT}:80` à la place de `-p 8092:80`, puis à la fin :
```bash
if [ "${NETTOYER}" = "true" ]; then
  docker images ${IMAGE_NAME} --format '{{.Tag}}' | grep '^build-' | sort -t- -k2 -n -r | tail -n +4 \
    | xargs -r -I{} docker rmi ${IMAGE_NAME}:{}
fi
```
</details>
