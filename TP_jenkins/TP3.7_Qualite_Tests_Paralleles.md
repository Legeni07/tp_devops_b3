### Projet Qualité – tests en parallèle, rapports et quality gates

**Objectif :** transformer le pipeline en **contrôle qualité automatique** : tests unitaires avec rapport et couverture minimale, analyse statique du code et du Dockerfile, scan de vulnérabilités. Chaque contrôle peut faire échouer le build (*quality gate*).

**Durée :** 2h – **Prérequis :** TP 3.5, plugins `junit`, `coverage`, `warnings-ng`, `docker-workflow` (préinstallés par le TP 3.1)

Notions nouvelles :
- **`parallel`** : plusieurs stages en même temps → pipeline plus rapide
- **`agent { docker { ... } }`** : un stage s'exécute dans un conteneur jetable (ici Python 3.12) au lieu d'installer Python dans Jenkins
- **`junit`**, **`recordCoverage`**, **`recordIssues`** : publication des rapports dans l'interface Jenkins

---

1. Ajouter le fichier `Jenkinsfile.qualite` à la racine du dépôt :
	```groovy
// TP 3.7 : contrôles qualité en parallèle, rapports de tests et quality gates
pipeline {
    agent any

    environment {
        IMAGE   = "legeni-webapp"
        VERSION = "1.0.${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {
        stage('Qualité du code') {
            parallel {
                stage('Tests unitaires') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            args '-e HOME=/tmp'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q -r requirements.txt -r requirements-dev.txt
                            mkdir -p reports
                            /tmp/venv/bin/pytest --junitxml=reports/junit.xml \
                                --cov=app --cov-report=xml:reports/coverage.xml --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit 'reports/junit.xml'
                            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                        }
                    }
                }
                stage('Lint Python') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            args '-e HOME=/tmp'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q flake8==7.1.1
                            mkdir -p reports
                            /tmp/venv/bin/flake8 --output-file=reports/flake8.txt --tee .
                        '''
                    }
                    post {
                        always {
                            recordIssues(tools: [flake8(pattern: 'reports/flake8.txt')])
                        }
                    }
                }
                stage('Lint Dockerfile') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint:v2.12.0 < Dockerfile'
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION .'
            }
        }

        stage('Scan de sécurité') {
            steps {
                // --exit-code 1 : le pipeline échoue si une vulnérabilité CRITICAL corrigeable est trouvée
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
                        aquasec/trivy:0.57.1 image --severity CRITICAL --ignore-unfixed --exit-code 1 $IMAGE:$VERSION
                '''
            }
        }
    }

    post {
        cleanup {
            cleanWs()
        }
    }
}
	```

2. Créer un job **Pipeline script from SCM** nommé **legeni-qualite**, Script Path : `Jenkinsfile.qualite`, et le lancer.

3. Observer
	- **Pipeline Graph** : les 3 contrôles s'exécutent en parallèle
	- Page du build : **Test Result** (6 tests), **Coverage Report** (≥ 80 %), **Flake8 Warnings**
	- Après 3 builds, la page du job affiche les **courbes de tendance** des tests et de la couverture

4. Faire échouer chaque quality gate (un à la fois, en annulant à chaque fois)
	| Modification | Stage en échec | Message attendu |
	|---|---|---|
	| ajouter une ligne de 150 caractères dans `app.py` | Lint Python | `E501 line too long` |
	| ajouter dans `app.py` une fonction d'une quinzaine de lignes sans test | Tests unitaires | `Required test coverage of 80% not reached` |
	| `FROM python:3.12-slim` → `FROM python:latest` dans le Dockerfile | Lint Dockerfile | `DL3007 Using latest is prone to errors` |
	| `FROM python:3.12-slim` → `FROM python:3.6-slim` | Scan de sécurité | liste de vulnérabilités `CRITICAL` |

5. Comprendre les rapports
	- Pourquoi les rapports sont-ils publiés dans un bloc `post { always { ... } }` plutôt qu'à la fin de `steps` ?
	- Quelle différence entre un build **UNSTABLE** (jaune) et **FAILURE** (rouge) ? Dans quel cas le plugin JUnit produit-il un build jaune ?

---

##### ✅ Vérifications
- [ ] Les 3 contrôles s'exécutent en parallèle (capture du Pipeline Graph)
- [ ] Rapports de tests, couverture et flake8 visibles dans le build
- [ ] Les 4 quality gates ont été déclenchés volontairement (captures)

##### 🎯 Défi
Ajoutez un stage parallèle **Secrets** qui recherche des secrets commités par erreur avec *gitleaks* :
`docker run --rm -v "$PWD":/repo zricethezav/gitleaks:v8.21.2 detect --source /repo --no-banner`.
Testez-le en commitant un faux jeton `AKIAIOSFODNN7EXAMPLE` dans un fichier.

> ⚠️ Le montage `-v "$PWD":/repo` ne fonctionne pas tel quel car Jenkins tourne dans un conteneur (le chemin du workspace n'existe pas sur l'hôte). Utilisez plutôt `agent { docker { image 'zricethezav/gitleaks:v8.21.2'; args '--entrypoint='; reuseNode true } }` et lancez `gitleaks detect --source . --no-banner` dans `steps`.
