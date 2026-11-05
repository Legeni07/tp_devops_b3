#### Projet test-acceptance – vérifier automatiquement que l'application répond

**Objectif :** ajouter au job de build un **test d'acceptance** : après la construction, l'image est lancée dans un conteneur de test et Jenkins vérifie par une requête HTTP que le site répond **et** affiche le bon contenu. Si le test échoue, le build est **rouge** et l'image ne va pas plus loin.

**Durée :** 45 min – **Prérequis :** TP 3.2

> Le principe : « une image qui se construit » n'est pas « une image qui fonctionne ». Le test d'acceptance est la **porte de qualité** (*quality gate*) qui sépare les deux.

---

1. Vérifier que le plugin **HTTP Request** est installé
	- **Administrer Jenkins** → **Plugins** → **Installed plugins** → rechercher `HTTP Request` (il est dans `plugins.txt` de l'image du TP 3.1 ; sinon : **Available plugins** → l'installer)

2. Créer le job **test-acceptance** en copiant le précédent
	- **Nouvel item** → nom = **test-acceptance** → en bas : **Copier depuis** = `build` → **OK**
	- Garder les paramètres `IMAGE_NAME`, `IMAGE_TAG`, `Votre_ID_Github` et la gestion de code source Git

3. Remplacer l'étape shell par la construction **et** le lancement d'un conteneur de **test**
	```bash
	#!/bin/bash
	set -euo pipefail
	TEST=${IMAGE_NAME}-test

	cat > Dockerfile <<'EOF'
	FROM nginx:1.27-alpine
	COPY . /usr/share/nginx/html/
	EXPOSE 80
	EOF
	printf '.git\nDockerfile\n' > .dockerignore
	docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

	# Jenkins tourne lui-même dans un conteneur : pour qu'il joigne le conteneur de test
	# par son nom, on place ce dernier sur le même réseau Docker que Jenkins.
	JENKINS_NET=$(docker inspect jenkins -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}')
	docker rm -f ${TEST} 2>/dev/null || true
	docker run -d --name ${TEST} --network ${JENKINS_NET} ${IMAGE_NAME}:${IMAGE_TAG}

	# Attendre que nginx accepte les connexions (au plus 10 s)
	for i in $(seq 1 10); do curl -sf http://${TEST}/ >/dev/null && break; sleep 1; done
	```

	Question : pourquoi `curl http://localhost/` ne fonctionnerait-il **pas** ici ? *(indice : où s'exécute le script ? où tourne le conteneur ?)*

4. Ajouter l'étape de test avec le plugin : **Ajouter une étape** → **HTTP Request**
	- **URL** : `http://${IMAGE_NAME}-test/`
	- **HTTP mode** : `GET`
	- **Avancé** :
		- **Response codes expected** : `200`
		- **Response content expected** : `Restaurant` (texte qui doit figurer dans la page, ici dans le `<title>`)
		- **Connection timeout** : `10`

5. Ajouter une étape shell de vérification **sans plugin** (plus portable : c'est ce qu'on utilisera dans le Jenkinsfile)
	```bash
	#!/bin/bash
	set -euo pipefail
	TEST=${IMAGE_NAME}-test
	# 1. La page d'accueil contient le titre attendu
	curl -sf http://${TEST}/ | grep -qi "<title>Restaurant" && echo "✅ titre OK"
	# 2. Les ressources statiques sont servies
	curl -sf -o /dev/null -w "%{http_code} %{content_type}\n" http://${TEST}/1.jpg
	# 3. Une page inexistante renvoie bien 404 (le serveur ne masque pas les erreurs)
	test "$(curl -s -o /dev/null -w '%{http_code}' http://${TEST}/nexiste-pas)" = "404" && echo "✅ 404 OK"
	```

6. Nettoyer le conteneur de test : ajoutez une **dernière** étape shell
	```bash
	docker rm -f ${IMAGE_NAME}-test
	```
	Question : si l'étape 4 échoue, cette dernière étape est-elle exécutée ? Quelle conséquence ? *(c'est pour cela que l'étape 3 commence par `docker rm -f ${TEST}` ; on règlera ce problème proprement avec le bloc `post { always { } }` du Jenkinsfile, TP 3.5)*

7. Lancer **Build avec des paramètres** → le build doit être **vert**. Dans **Console Output**, repérez la ligne du plugin : `Success: Status code 200 is in the accepted range: 200`.

8. **Casser le test** volontairement
	- Sur GitHub, dans votre fork, remplacez le titre de `index.html` par `<title>Pizzeria</title>` et *commit*
	- Relancez le job : il est **rouge** (`Fail: Response doesn't contain expected content 'Restaurant'`)
	- Constatez que l'image a été construite mais que **rien n'a été publié ni déployé** : c'est le rôle de la porte de qualité
	- Remettez le titre d'origine, relancez : vert

---

##### ✅ Vérifications
- [ ] Le job `test-acceptance` est vert, la console montre `Status code 200` et `✅ titre OK`
- [ ] Un changement de titre fait passer le job au **rouge**
- [ ] Après un build vert, `docker ps -a --filter name=restaurant-test` ne montre plus de conteneur

##### Correction des erreurs fréquentes
| Erreur | Cause | Solution |
|---|---|---|
| `UnknownHostException: restaurant-test` | conteneur de test pas sur le réseau de Jenkins | vérifier `JENKINS_NET` : `docker inspect jenkins -f '{{json .NetworkSettings.Networks}}'` |
| `Error: No such object: jenkins` | le conteneur Jenkins porte un autre nom | adapter `docker inspect <nom_du_conteneur>` (`docker ps`) |
| `Connection refused` | nginx pas encore démarré | boucle d'attente de l'étape 3 |
| Le conteneur de test reste après un échec | la dernière étape n'est pas exécutée si une étape précédente échoue | `docker rm -f` en début de script (déjà présent), puis `post { always }` au TP 3.5 |

##### Questions (compte-rendu)
1. Quelle est la différence entre un **test unitaire** (TP 3.5, `pytest`) et un **test d'acceptance** ? Lequel doit s'exécuter en premier, et pourquoi ?
2. Le plugin HTTP Request ou `curl` dans un script : avantages et inconvénients de chaque approche ?

##### 🎯 Défi (15 min)
Ajoutez un **test de performance minimal** : la page d'accueil doit répondre en moins de 500 ms (moyenne de 10 requêtes), sinon le build échoue.

<details>
<summary>Solution</summary>

```bash
#!/bin/bash
set -euo pipefail
TEMPS=$(for i in $(seq 1 10); do curl -s -o /dev/null -w '%{time_total}\n' http://${IMAGE_NAME}-test/; done)
MOY=$(echo "$TEMPS" | awk '{s+=$1} END {printf "%.3f", s/NR}')
echo "Temps moyen : ${MOY} s"
awk -v m="$MOY" 'BEGIN { exit !(m < 0.5) }' || { echo "❌ trop lent"; exit 1; }
```
</details>
