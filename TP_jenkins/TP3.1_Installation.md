### Projet installation – Jenkins en conteneur, prêt pour le CI/CD

**Objectif :** installer Jenkins avec Docker Compose de façon **reproductible** : données persistées, accès à Docker sans `chmod 666`, outils (Docker, kubectl, Ansible) et plugins préinstallés.

**Durée :** 1h – **Prérequis :** Docker et Docker Compose (TP 2.2)

> Pourquoi une image personnalisée ? Dans l'ancienne version des TPs, on installait Docker **à la main** dans le conteneur (`apt-get install docker.io`) et on ouvrait le socket à tous (`chmod 776/666 /var/run/docker.sock`). Ces modifications disparaissent dès que le conteneur est recréé, et le `chmod` est une faille de sécurité. Ici tout est décrit dans un `Dockerfile` versionné.

---

1. Récupérer les fichiers du dossier [`jenkins-docker/`](jenkins-docker/) :
	- `Dockerfile` : part de `jenkins/jenkins:lts-jdk21`, ajoute Docker CLI, kubectl, Ansible
	- `plugins.txt` : liste des plugins installés automatiquement
	- `docker-compose.yml` : service Jenkins, volume `jenkins_home`, socket Docker

2. Donner à Jenkins le droit d'utiliser Docker **proprement** : on ajoute l'utilisateur `jenkins` du conteneur au groupe qui possède le socket sur l'hôte.
	```bash
	cd TP_jenkins/jenkins-docker
	echo "DOCKER_GID=$(stat -c %g /var/run/docker.sock)" > .env
	cat .env
	```

3. Construire et démarrer
	```bash
	docker compose up -d --build        # 5 à 10 min la première fois
	docker compose logs -f jenkins      # attendre "Jenkins is fully up and running"
	```

4. Premier accès
	- Récupérer le mot de passe initial :
		```bash
		docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
		```
	- Ouvrir `http://<IP_VM>:8080`, coller le mot de passe
	- **Install suggested plugins** (les plugins de `plugins.txt` sont déjà là)
	- Créer le compte administrateur, garder l'URL proposée

5. Vérifier que Jenkins pilote Docker et dispose des outils
	- **Tableau de bord** → **Nouvel item** → nom = **verif-outils** → **Pipeline** → **OK**
	- Dans **Pipeline script**, coller :
		```groovy
		pipeline {
		    agent any
		    stages {
		        stage('Outils') {
		            steps {
		                sh 'docker version --format "Docker serveur : {{.Server.Version}}"'
		                sh 'docker compose version'
		                sh 'kubectl version --client'
		                sh 'ansible --version | head -1'
		                sh 'id'
		            }
		        }
		    }
		}
		```
	- **Save** → **Lancer un build** → **Console Output** : toutes les commandes doivent réussir.

6. Prouver la persistance
	```bash
	docker compose down          # supprime le conteneur…
	docker compose up -d         # …le job verif-outils et son historique sont toujours là
	docker volume ls | grep jenkins_home
	```

7. Sauvegarder Jenkins (à faire avant chaque mise à jour)
	```bash
	docker run --rm -v jenkins-docker_jenkins_home:/source:ro -v "$PWD":/backup alpine \
	  tar czf /backup/jenkins_home_$(date +%F).tar.gz -C /source .
	```

---

##### ✅ Vérifications
- [ ] Le job `verif-outils` est vert et affiche les versions de Docker, kubectl et Ansible
- [ ] `id` dans la console montre le groupe du socket Docker (ex. `989`)
- [ ] Le job survit à `docker compose down && docker compose up -d`

##### Correction des erreurs fréquentes
| Erreur | Cause | Solution |
|---|---|---|
| `permission denied ... /var/run/docker.sock` | `DOCKER_GID` absent ou faux | refaire l'étape 2 puis `docker compose up -d --force-recreate` |
| Port 8080 déjà utilisé | autre service (Odoo, Jenkins précédent…) | changer `"8080:8080"` en `"8186:8080"` |
| Jenkins très lent / redémarre | mémoire insuffisante | prévoir 2 Go de RAM libres pour la VM |
| Plugins non installés | réseau / proxy pendant le build | relancer `docker compose build --no-cache` |

> ℹ️ Le dossier [`../jenkins-install/`](../jenkins-install/) contient l'ancienne installation (image Ubuntu + systemd). Elle reste utilisable, mais cette version est celle utilisée par les TPs 3.5 à 3.8 et le projet final.
