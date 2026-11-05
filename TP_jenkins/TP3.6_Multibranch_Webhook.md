### Projet Multibranch – un pipeline par branche, déclenché par webhook

**Objectif :** reproduire le flux de travail d'une équipe : chaque branche (et chaque *pull request*) est testée automatiquement **dès le push**, et seule la branche `main` publie une image.

**Durée :** 1h30 – **Prérequis :** TP 3.5, TP Git (branches, pull requests)

---

1. Préparer le Jenkinsfile pour les branches
	Dans le Jenkinsfile du TP 3.5, conditionner le push à la branche principale :
	```groovy
	        stage('Push Docker Hub') {
	            when { branch 'main' }
	            steps { ... }
	        }
	```
	et utiliser une version qui identifie la branche :
	```groovy
	    environment {
	        VERSION = "${env.BRANCH_NAME == 'main' ? '1.0.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"
	    }
	```
	*Commit* + *push* sur `main`.

2. Créer le job Multibranch
	**Nouvel item** → nom = **legeni-webapp-mb** → **Multibranch Pipeline** → **OK**
	- **Branch Sources** → **Add source** → **GitHub**
		- Credentials : ajouter un **Username with password** (identifiant GitHub + **Personal Access Token** avec les droits `repo` et `admin:repo_hook`), ID = **github**
		- Repository HTTPS URL : `https://github.com/<Votre_ID_Github>/legeni-webapp.git` → **Validate**
		- Behaviours : *Discover branches* + *Discover pull requests from origin*
	- **Build Configuration** : by Jenkinsfile, Script Path `Jenkinsfile`
	- **Scan Multibranch Pipeline Triggers** : *Periodically if not otherwise run* → **1 minute** (filet de sécurité si le webhook n'arrive pas)
	- **Save** : Jenkins scanne le dépôt et crée un sous-job **main**

3. Travailler sur une branche
	```bash
	git checkout -b feature/couleur
	sed -i 's/#1f6feb/#8250df/' app.py
	git commit -am "Nouvelle couleur par défaut" && git push -u origin feature/couleur
	```
	- Après le scan, un sous-job **feature/couleur** apparaît et s'exécute
	- ✅ Le stage **Push Docker Hub** est *skipped* (gris) sur cette branche

4. Pull request
	- Sur GitHub, ouvrir une *pull request* `feature/couleur` → `main`
	- Un sous-job **PR-1** apparaît dans l'onglet **Pull Requests** de Jenkins
	- Le statut du build (✔ / ✘) s'affiche **directement dans la pull request GitHub**
	- *Merger* la PR : le build de **main** publie l'image

5. Déclenchement instantané par webhook
	GitHub doit pouvoir joindre Jenkins depuis Internet. En salle de TP, utiliser un tunnel :
	```bash
	# Option A : ngrok (compte gratuit)
	ngrok http 8080
	# Option B : smee.io (aucun compte) – https://smee.io/new fournit une URL
	npx smee-client --url https://smee.io/<canal> --target http://localhost:8080/github-webhook/
	```
	- Sur GitHub : **Settings** → **Webhooks** → **Add webhook**
		- Payload URL : `https://<url_publique>/github-webhook/` (avec le `/` final)
		- Content type : `application/json` – événements : *push* et *pull requests*
	- Pousser un commit : le build démarre **en quelques secondes**
	- **Recent Deliveries** (GitHub) doit afficher un code **200**

6. Protéger `main`
	Sur GitHub : **Settings** → **Branches** → **Add rule** → `main` → *Require status checks to pass before merging* → cocher le contexte Jenkins.
	✅ Une PR dont le build est rouge ne peut plus être mergée.

---

##### ✅ Vérifications
- [ ] Sous-jobs `main`, `feature/couleur` et `PR-1` visibles dans Jenkins
- [ ] Push d'image uniquement sur `main`
- [ ] Statut Jenkins affiché dans la pull request (capture)
- [ ] Webhook en 200 dans *Recent Deliveries*

##### Correction des erreurs fréquentes
| Erreur | Solution |
|---|---|
| Aucun sous-job créé | vérifier le Script Path et que le Jenkinsfile est bien sur la branche ; **Scan Repository Now** |
| Webhook en erreur 403 / 404 | URL sans `/github-webhook/` final, ou tunnel arrêté |
| `API rate limit exceeded` | ajouter le credential GitHub (les appels anonymes sont limités à 60/heure) |
