##### Projet Artefact – publier l'image testée sur Docker Hub

**Objectif :** compléter la chaîne *build → test → publication* : une image qui a passé le test d'acceptance est **taguée de façon traçable** et **poussée sur Docker Hub**, avec des identifiants stockés dans le **gestionnaire de credentials** de Jenkins (jamais dans le job ni dans le code). Enchaîner ensuite les jobs entre eux.

**Durée :** 45 min – **Prérequis :** TP 3.3, compte Docker Hub et **jeton d'accès** (TP 2.3, étape 1)

> L'**artefact** est le produit livrable d'un build : ici une image Docker. Règle d'or de la livraison continue : **on construit une seule fois**, et c'est cette même image, identifiée par un tag unique, qui passe en test, en staging puis en production.

---

1. Enregistrer les identifiants Docker Hub dans Jenkins
	- **Administrer Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → **Add Credentials**
	- **Kind** : `Username with password`
	- **Username** : votre identifiant Docker Hub
	- **Password** : le **jeton d'accès** (pas le mot de passe du compte)
	- **ID** : `dockerhub` (c'est par cet ID que les jobs et le Jenkinsfile du TP 3.5 y feront référence)
	- **Create**

	Question : où Jenkins stocke-t-il ce secret, et sous quelle forme ? *(indice : `docker exec jenkins grep -A3 dockerhub /var/jenkins_home/credentials.xml`)*

2. Créer le job **artefact** en copiant `test-acceptance`
	- **Nouvel item** → nom = **artefact** → **Copier depuis** = `test-acceptance` → **OK**
	- Ajouter un **paramètre String** `DOCKERHUB_ID` (votre identifiant Docker Hub)
	- Changer la valeur par défaut de `IMAGE_TAG` en `1.0` : la version « humaine » de l'application

3. Injecter les credentials dans le build
	- **Environnement de build** → cocher **Use secret text(s) or file(s)** → **Ajouter** → **Username and password (separated)**
	- **Username Variable** : `DOCKERHUB_USER` – **Password Variable** : `DOCKERHUB_TOKEN`
	- **Credentials** : `dockerhub`

4. Modifier la première étape shell : l'image est nommée **avec votre identifiant Docker Hub** et reçoit un tag **unique par build**
	```bash
	#!/bin/bash
	set -euo pipefail
	TEST=${IMAGE_NAME}-test
	IMAGE=${DOCKERHUB_ID}/${IMAGE_NAME}
	UNIQUE_TAG=${IMAGE_TAG}.${BUILD_NUMBER}-${GIT_COMMIT:0:7}     # ex. 1.0.12-3f9c2ab

	cat > Dockerfile <<'EOF'
	FROM nginx:1.27-alpine
	COPY . /usr/share/nginx/html/
	EXPOSE 80
	EOF
	printf '.git\nDockerfile\n' > .dockerignore
	docker build --label org.opencontainers.image.revision=${GIT_COMMIT} \
	             -t ${IMAGE}:${UNIQUE_TAG} -t ${IMAGE_NAME}:${IMAGE_TAG} .

	JENKINS_NET=$(docker inspect jenkins -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}')
	docker rm -f ${TEST} 2>/dev/null || true
	docker run -d --name ${TEST} --network ${JENKINS_NET} ${IMAGE}:${UNIQUE_TAG}
	for i in $(seq 1 10); do curl -sf http://${TEST}/ >/dev/null && break; sleep 1; done

	echo "${UNIQUE_TAG}" > tag.txt          # transmis à l'étape de publication
	```
	Les étapes **HTTP Request** et de vérification `curl` du TP 3.3 restent inchangées.

5. Ajouter, **après** les tests, l'étape de publication : **Exécuter un script shell**
	```bash
	#!/bin/bash
	set -euo pipefail
	IMAGE=${DOCKERHUB_ID}/${IMAGE_NAME}
	UNIQUE_TAG=$(cat tag.txt)

	echo "${DOCKERHUB_TOKEN}" | docker login -u "${DOCKERHUB_USER}" --password-stdin
	docker push ${IMAGE}:${UNIQUE_TAG}
	# "latest" ne pointe que vers une image qui a passé les tests
	docker tag ${IMAGE}:${UNIQUE_TAG} ${IMAGE}:latest
	docker push ${IMAGE}:latest
	docker logout
	echo "Image publiée : ${IMAGE}:${UNIQUE_TAG}"
	```
	puis l'étape de nettoyage `docker rm -f ${IMAGE_NAME}-test` en dernier.

6. Archiver la trace du build : **Actions à la suite du build** → **Archiver des artefacts** → `tag.txt`
	- Sur la page du build, `tag.txt` est téléchargeable : on sait **quelle image** ce build a produite.

7. Lancer **Build avec des paramètres**, puis vérifier :
	- La console affiche `docker login` → `Login Succeeded`, et le jeton apparaît sous la forme `****` : Jenkins **masque** les secrets dans les logs
	- Sur Docker Hub : le dépôt `restaurant` contient le tag `1.0.<n>-<sha>` et `latest`
	- Preuve de traçabilité :
		```bash
		docker pull <DOCKERHUB_ID>/restaurant:latest
		docker inspect <DOCKERHUB_ID>/restaurant:latest --format '{{index .Config.Labels "org.opencontainers.image.revision"}}'
		```
		Le hash affiché est celui du commit visible sur GitHub.

8. Faire échouer la porte de qualité (comme au TP 3.3, titre `Pizzeria`) : le job est rouge **et aucun nouveau tag n'apparaît sur Docker Hub**. Remettez le titre d'origine.

9. Enchaîner les jobs (vision « chaîne de jobs » d'avant le pipeline)
	- Créez un job freestyle **deploy** (sans paramètre) dont l'étape shell relance le site en production locale à partir de Docker Hub :
		```bash
		docker rm -f restaurant-prod 2>/dev/null || true
		docker run -d --name restaurant-prod -p 8099:80 <DOCKERHUB_ID>/restaurant:latest
		```
	- Dans le job **artefact** : **Actions à la suite du build** → **Construire d'autres projets** → `deploy` → *Déclencher seulement si la construction est stable*
	- Lancez `artefact` : `deploy` démarre automatiquement après lui, le site répond sur `http://localhost:8099`

---

##### ✅ Vérifications
- [ ] Le credential `dockerhub` existe et n'apparaît jamais en clair dans la console (`****`)
- [ ] Docker Hub contient un tag `1.0.<n>-<sha>` par build réussi, et `latest`
- [ ] Le label `org.opencontainers.image.revision` de l'image correspond au commit GitHub
- [ ] Un test en échec n'entraîne **aucune** publication
- [ ] `deploy` est déclenché automatiquement après un `artefact` vert

##### Correction des erreurs fréquentes
| Erreur | Cause | Solution |
|---|---|---|
| `denied: requested access to the resource is denied` | image non préfixée par l'ID Docker Hub, ou jeton en lecture seule | `DOCKERHUB_ID/...`, jeton *Read & Write* |
| `unauthorized: incorrect username or password` | mot de passe du compte au lieu du jeton, ou faute de frappe | recréer le credential avec le jeton |
| `DOCKERHUB_TOKEN: unbound variable` | case **Use secret text(s) or file(s)** non cochée | étape 3 |
| `cat: tag.txt: No such file` | étape de publication placée avant l'étape de build | réordonner les étapes (glisser-déposer) |

##### Questions (compte-rendu)
1. Pourquoi un tag unique (`1.0.12-3f9c2ab`) plutôt que de toujours pousser `latest` ? Que se passe-t-il en production si `latest` change pendant un redéploiement ?
2. Combien de **jobs**, d'**étapes** et d'**écrans de configuration** avez-vous manipulés depuis le TP 3.2 ? Si Jenkins est perdu, combien de temps faut-il pour tout reconfigurer ? → c'est la motivation du **pipeline as code** (TP 3.5), qui regroupe tout cela dans un fichier `Jenkinsfile` versionné.

##### 🎯 Défi (15 min)
Faites en sorte que le job **artefact** refuse de publier si l'image contient une vulnérabilité **CRITICAL** corrigeable, à l'aide de Trivy lancé en conteneur (voir TP 2.9).

<details>
<summary>Solution</summary>

Étape shell à insérer entre les tests et la publication :
```bash
#!/bin/bash
set -euo pipefail
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.57.1 \
  image --quiet --severity CRITICAL --ignore-unfixed --exit-code 1 \
  ${DOCKERHUB_ID}/${IMAGE_NAME}:$(cat tag.txt)
```
`--exit-code 1` fait échouer l'étape (et donc le job) si une vulnérabilité critique corrigeable est trouvée.
</details>
