### Projet Déploiement continu – staging automatique, production validée, rollback

**Objectif :** prolonger la CI par la **CD** : après chaque build réussi de `main`, l'application est déployée automatiquement en *staging* ; la mise en production attend une **validation humaine**, est vérifiée par un *smoke test*, et revient automatiquement à la version précédente en cas d'échec.

**Durée :** 2h – **Prérequis :** TP 3.6 (job multibranch), TP 2.8 (Docker Compose)

```
 push main ─▶ Tests ─▶ Build ─▶ Deploy staging ─▶ Smoke test ─▶ ⏸ Validation ─▶ Deploy prod ─▶ Smoke test ─┬─▶ ✅
                                  (port 8081)                    (humain)       (port 8082)                └─▶ ❌ rollback
```

Dans ce TP, staging et prod sont deux stacks Docker Compose **sur la machine de Jenkins**. Dans le projet final, ce seront deux namespaces Kubernetes déployés par Ansible : la logique du pipeline reste la même.

---

1. Fichiers de déploiement (déjà présents dans `app-fil-rouge/deploy/`)
	- `docker-compose.deploy.yml` : web + Redis, image et port paramétrés par variables (`IMAGE_REF`, `PORT`, `APP_ENV`, `REDIS_PASSWORD`). Il ne contient **aucun bind mount** : il fonctionne lancé depuis le conteneur Jenkins.
	- `smoke-test.sh` : vérifie que `/ready` répond et que `/api/info` renvoie la version attendue.
	```bash
	cat deploy/smoke-test.sh
	```

2. Créer les secrets Redis dans Jenkins
	**Manage Jenkins** → **Credentials** → **Add Credentials** → Kind : **Secret text**
	- ID **redis-password-staging**, Secret : un mot de passe aléatoire (`openssl rand -hex 16`)
	- ID **redis-password-prod**, Secret : un autre mot de passe

3. Remplacer le `Jenkinsfile` par la version CD :
	```groovy
// TP 3.8 : intégration continue + déploiement continu (staging automatique, prod après validation)
pipeline {
    agent any

    environment {
        IMAGE   = "legeni-webapp"
        VERSION = "1.0.${env.BUILD_NUMBER}"
        COMPOSE = "docker compose -f deploy/docker-compose.deploy.yml"
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()          // jamais deux déploiements en même temps
    }

    stages {
        stage('Tests unitaires') {
            steps {
                sh 'docker build --target test -t $IMAGE:test .'
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION .'
            }
        }

        stage('Déploiement staging') {
            environment {
                REDIS_PASSWORD = credentials('redis-password-staging')
            }
            steps {
                sh '''
                    IMAGE_REF=$IMAGE:$VERSION APP_ENV=staging APP_COLOR='#bf8700' PORT=8081 \
                        $COMPOSE -p legeni-staging up -d --wait
                '''
                sh './deploy/smoke-test.sh legeni-staging $VERSION'
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Déployer la version ${VERSION} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            environment {
                REDIS_PASSWORD = credentials('redis-password-prod')
            }
            steps {
                script {
                    // Image actuellement en production : servira au rollback
                    env.PREVIOUS_IMAGE = sh(
                        script: "docker inspect -f '{{.Config.Image}}' legeni-prod-web-1 2>/dev/null || echo none",
                        returnStdout: true
                    ).trim()
                    echo "Version en production avant déploiement : ${env.PREVIOUS_IMAGE}"
                    try {
                        sh '''
                            IMAGE_REF=$IMAGE:$VERSION APP_ENV=prod APP_COLOR='#2da44e' PORT=8082 \
                                $COMPOSE -p legeni-prod up -d --wait
                            ./deploy/smoke-test.sh legeni-prod $VERSION
                        '''
                    } catch (err) {
                        if (env.PREVIOUS_IMAGE != 'none') {
                            echo "Smoke test KO : retour à ${env.PREVIOUS_IMAGE}"
                            sh '''
                                IMAGE_REF=$PREVIOUS_IMAGE APP_ENV=prod APP_COLOR='#2da44e' PORT=8082 \
                                    $COMPOSE -p legeni-prod up -d --wait
                            '''
                        }
                        error "Déploiement production échoué, rollback effectué"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} : version ${VERSION} déployée"
        }
        failure {
            echo "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} en échec"
            // Avec le plugin Email Extension configuré (Manage Jenkins > System > Extended E-mail) :
            // emailext to: 'equipe@legeni.cm', subject: "Échec ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //          body: "Voir ${env.BUILD_URL}console"
        }
    }
}
	```
	*Commit* + *push* sur `main`. Le job multibranch du TP 3.6 se déclenche.

4. Premier déploiement
	- Le pipeline s'arrête sur **Validation production** : bouton **Déployer** dans *Stage View* (ou dans la console)
	- ✅ `http://<IP_VM>:8081` → bannière orange, environnement *staging*
	- ✅ `http://<IP_VM>:8082` → bannière verte, environnement *prod*, même version

5. Une branche ne va jamais en production
	Poussez un commit sur `feature/...` : seuls les tests, le build et le staging s'exécutent ; les deux derniers stages sont *skipped*.

6. Simuler une mise en production ratée et le rollback
	- Modifiez `smoke-test.sh` pour qu'il échoue en production uniquement, en ajoutant avant le premier `curl_net` :
		```bash
		[ "$PROJECT" = legeni-prod ] && { echo "panne simulée"; exit 1; }
		```
	- *Push*, validez la mise en production
	- ✅ Console : `Smoke test KO : retour à legeni-webapp:1.0.<n-1>` puis build **rouge**
	- ✅ `docker ps` : `legeni-prod-web-1` tourne toujours avec **l'ancienne** image, la prod n'a jamais été coupée longtemps
	- Annulez la modification

7. Notifications
	- **Manage Jenkins** → **System** → **Extended E-mail Notification** : serveur SMTP (ex. `smtp.gmail.com:465`, SSL, mot de passe d'application)
	- Décommentez le bloc `emailext` du `post { failure { ... } }` et provoquez un échec
	- *(Bonus)* notification Slack/Teams/Discord par webhook :
		```groovy
		withCredentials([string(credentialsId: 'webhook-discord', variable: 'HOOK')]) {
		    sh 'curl -s -H "Content-Type: application/json" -d "{\\"content\\": \\"❌ $JOB_NAME #$BUILD_NUMBER\\"}" "$HOOK"'
		}
		```

---

##### ✅ Vérifications
- [ ] Un push sur `main` déploie automatiquement en staging (capture du port 8081)
- [ ] La production attend la validation (capture du bouton)
- [ ] Le rollback automatique a été démontré (capture de la console)
- [ ] Une notification d'échec a été reçue

##### Questions (compte-rendu)
1. Différence entre **livraison continue** (*continuous delivery*) et **déploiement continu** (*continuous deployment*) : lequel ce pipeline implémente-t-il ? Que faudrait-il retirer pour passer à l'autre ?
2. Pourquoi `disableConcurrentBuilds()` ?
3. Le rollback redéploie l'**image** précédente. Quelles données ne seraient pas restaurées si la nouvelle version avait modifié la base de données ?
