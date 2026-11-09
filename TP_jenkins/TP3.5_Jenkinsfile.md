### Projet Jenkinsfile – le pipeline as code

**Objectif :** remplacer les jobs *freestyle* des TPs 3.2 à 3.4 (configurés à la souris, non versionnés) par **un fichier `Jenkinsfile` stocké dans le dépôt Git**, qui enchaîne : tests unitaires → build de l'image → test d'acceptance → push sur Docker Hub.

**Durée :** 2h – **Prérequis :** TP 3.1 (Jenkins avec Docker), compte GitHub et Docker Hub (jeton d'accès, TP 2.9)

| Freestyle (TP 3.2–3.4) | Pipeline (ce TP) |
|---|---|
| Configuration dans l'interface Jenkins | Configuration dans Git, relue en *pull request* |
| 3 jobs séparés (build, test, artefact) | 1 pipeline, des *stages* visibles |
| Perdue si Jenkins est perdu | Reconstruite en 1 clic depuis le dépôt |

---

1. Créer le dépôt de l'application
	- Sur GitHub, créer un dépôt public **legeni-webapp**
	- Y pousser le contenu du dossier [`app-fil-rouge/`](../app-fil-rouge/) :
		```bash
		cp -r app-fil-rouge ~/legeni-webapp && cd ~/legeni-webapp
		git init -b main && git add . && git commit -m "Application fil rouge"
		git remote add origin https://github.com/<Votre_ID_Github>/legeni-webapp.git
		git push -u origin main
		```

2. Lire le `Jenkinsfile` fourni à la racine du dépôt et répondre (compte-rendu) :
	- À quoi servent les blocs `parameters`, `environment`, `options`, `post` ?
	- Pourquoi le test d'acceptance utilise-t-il un conteneur `curlimages/curl` sur un réseau dédié plutôt que `curl http://localhost:5000` ? *(indice : Jenkins tourne lui-même dans un conteneur)*
	- Que se passe-t-il pour les étapes suivantes si `Tests unitaires` échoue ?

```groovy
// Pipeline de référence du TP 3.5 : tests -> build -> test d'acceptance -> push
pipeline {
    agent any

    parameters {
        string(name: 'DOCKERHUB_ID', defaultValue: 'legeni07', description: 'Votre identifiant Docker Hub')
        booleanParam(name: 'PUSH', defaultValue: true, description: "Pousser l'image sur Docker Hub")
    }

    environment {
        IMAGE          = "${params.DOCKERHUB_ID}/legeni-webapp"
        VERSION        = "1.0.${env.BUILD_NUMBER}"
        TEST_CONTAINER = "webapp-ci-${env.BUILD_NUMBER}"
        CI_NETWORK     = "ci-net-${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Tests unitaires') {
            steps {
                // L'étape "test" du Dockerfile lance flake8 + pytest : si un test échoue, le build s'arrête ici
                sh 'docker build --target test -t legeni-webapp:test .'
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION -t $IMAGE:latest .'
            }
        }

        stage("Test d'acceptance") {
            steps {
                sh '''
                    docker network create $CI_NETWORK
                    docker run -d --name $TEST_CONTAINER --network $CI_NETWORK $IMAGE:$VERSION
                    # On interroge le conteneur depuis un conteneur curl placé sur le même réseau
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf --retry 10 --retry-connrefused --retry-delay 1 http://$TEST_CONTAINER:5000/health
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf http://$TEST_CONTAINER:5000/ | grep -q "Version $VERSION"
                '''
            }
        }

        stage('Push Docker Hub') {
            when { expression { params.PUSH } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$VERSION
                        docker push $IMAGE:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f $TEST_CONTAINER || true; docker network rm $CI_NETWORK || true'
        }
        success {
            echo "Image ${IMAGE}:${VERSION} construite et validée"
        }
        failure {
            echo "Échec du pipeline : consultez la console de l'étape en rouge"
        }
    }
}
```

3. Créer le credential Docker Hub
	**Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
	- Kind : **Username with password**
	- Username : votre identifiant Docker Hub – Password : votre **jeton d'accès** (pas le mot de passe)
	- ID : **dockerhub** *(c'est l'ID utilisé dans `withCredentials`)*

4. Créer le job pipeline
	**Nouvel item** → nom = **legeni-webapp** → **Pipeline** → **OK**
	- **Pipeline** → Definition : **Pipeline script from SCM**
	- SCM : **Git** – Repository URL : `https://github.com/<Votre_ID_Github>/legeni-webapp.git`
	- Branch Specifier : `*/main` – Script Path : `Jenkinsfile`
	- **Save**

	> ⚠️ Ne collez **pas** le Jenkinsfile dans le champ *Pipeline script* : dans ce mode, Jenkins ne clone pas le dépôt, l'espace de travail reste vide et le premier `docker build` échoue (`open Dockerfile: no such file or directory`). C'est le mode *from SCM* qui récupère le code **et** le Jenkinsfile depuis Git.

5. Exécuter
	- Premier **Lancer un build** : Jenkins découvre les paramètres définis dans le Jenkinsfile
	- Ensuite **Build with Parameters** → `DOCKERHUB_ID` = votre identifiant → **Build**
	- Suivre l'exécution dans **Stage View** (ou **Pipeline Graph** / **Console Output**)
	- ✅ Vérifier sur Docker Hub les tags `1.0.<numéro_de_build>` et `latest`

6. Casser le build volontairement
	- Dans `app.py`, remplacer `status="ok"` par `status="ko"`, *commit* et *push*
	- Relancer : l'étape **Tests unitaires** est rouge, **aucune image n'est construite ni poussée**
	- Annuler la modification (`git revert HEAD && git push`) et relancer : retour au vert

7. Déclencher automatiquement
	Dans le job : **Configurer** → **Build Triggers** → **Scrutation de l'outil de gestion de version** (*Poll SCM*) → `H/2 * * * *`
	Poussez un commit : un build démarre dans les 2 minutes. *(Le déclenchement instantané par webhook est vu au TP 3.6.)*

---

##### ✅ Vérifications
- [ ] Les 4 stages sont verts dans **Stage View** (capture)
- [ ] L'image `1.0.<n>` est sur Docker Hub
- [ ] Un test cassé bloque le pipeline avant le build de l'image (capture)
- [ ] Aucun mot de passe en clair dans le Jenkinsfile ni dans la console (Jenkins masque `****`)

##### Correction des erreurs fréquentes
| Erreur | Cause | Solution |
|---|---|---|
| `failed to read dockerfile: open Dockerfile: no such file or directory` (et `transferring dockerfile: 2B`) | job en mode **Pipeline script** : aucun code récupéré | Definition = **Pipeline script from SCM** (étape 4) ; vérifier que `Dockerfile` est à la racine du dépôt |
| `MissingPropertyException: No such property: groovy` | la ligne ```` ```groovy ```` du Markdown a été copiée avec le script | ne copier que de `pipeline {` à la dernière `}` |
| `Couldn't find any revision to build` | branche `*/master` alors que le dépôt utilise `main` | corriger le **Branch Specifier** |
| `unauthorized: incorrect username or password` | credential `dockerhub` absent ou mot de passe au lieu du jeton | étape 3 |
| `network ci-net-<n> not found` dans *Post Actions* | le pipeline a échoué avant de créer le réseau | sans gravité : `\|\| true` ignore l'erreur |

##### 🎯 Défi
Ajoutez un paramètre `choice` nommé `APP_COLOR` (`#1f6feb`, `#2da44e`, `#d1242f`) transmis au conteneur de test d'acceptance (`-e APP_COLOR=...`), et vérifiez dans le stage d'acceptance que la couleur apparaît dans la page.

<details>
<summary>Solution</summary>

```groovy
parameters {
    choice(name: 'APP_COLOR', choices: ['#1f6feb', '#2da44e', '#d1242f'], description: 'Couleur de la bannière')
}
// dans le stage d'acceptance :
sh '''
    docker run -d --name $TEST_CONTAINER --network $CI_NETWORK -e APP_COLOR="$APP_COLOR" $IMAGE:$VERSION
    ...
    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 -sf http://$TEST_CONTAINER:5000/ | grep -q "$APP_COLOR"
'''
```
Les paramètres sont exposés comme variables d'environnement dans `sh`.
</details>
