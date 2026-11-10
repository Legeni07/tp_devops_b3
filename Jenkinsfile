// Pipeline du TP 3.5 (branche jenkinsfile) : tests -> build -> test d'acceptance -> push
// Job Jenkins : Pipeline script from SCM
//   Repository URL : https://github.com/Legeni07/tp_devops_b3.git
//   Branch Specifier : */jenkinsfile   -   Script Path : Jenkinsfile
// L'application se trouve dans app-fil-rouge/ : chaque étape s'y place avec dir(APP_DIR).
pipeline {
    agent any

    parameters {
        string(name: 'DOCKERHUB_ID', defaultValue: 'legeni', description: 'Votre identifiant Docker Hub (le credential dockerhub doit appartenir à ce compte)')
        booleanParam(name: 'PUSH', defaultValue: true, description: "Pousser l'image sur Docker Hub")
        booleanParam(name: 'DEPLOY', defaultValue: true, description: "Lancer l'application sur la machine Docker (http://localhost:<APP_PORT>)")
        string(name: 'APP_PORT', defaultValue: '8095', description: "Port publié sur la machine hôte pour ouvrir l'application")
    }

    environment {
        IMAGE          = "${params.DOCKERHUB_ID}/legeni-webapp"
        VERSION        = "1.0.${env.BUILD_NUMBER}"
        TEST_CONTAINER = "webapp-ci-${env.BUILD_NUMBER}"
        CI_NETWORK     = "ci-net-${env.BUILD_NUMBER}"
        APP_DIR        = "app-fil-rouge"
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
                dir(env.APP_DIR) {
                    sh 'docker build --target test -t legeni-webapp:test .'
                }
            }
        }

        stage('Build image') {
            steps {
                dir(env.APP_DIR) {
                    sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION -t $IMAGE:latest .'
                }
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

        stage('Déploiement local') {
            when { expression { params.DEPLOY } }
            steps {
                // Jenkins pilote le Docker de l'hôte (socket monté) : le conteneur tourne à côté de Jenkins,
                // et -p publie son port 5000 sur la machine hôte -> http://localhost:APP_PORT
                sh '''
                    docker rm -f legeni-webapp || true
                    docker run -d --name legeni-webapp --restart unless-stopped \
                        -p $APP_PORT:5000 $IMAGE:$VERSION
                '''
                echo "Application disponible sur http://localhost:${params.APP_PORT}"
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
