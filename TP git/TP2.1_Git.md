# TP 2.1 : Git en équipe – branches, pull requests, conflits, tags

**Objectif :** créer le dépôt de l'application fil rouge, y travailler à deux avec le workflow **GitHub Flow** (branche → pull request → revue → merge), résoudre un conflit et marquer une version. Ce dépôt sera utilisé par Jenkins à partir du TP 3.5 et dans le projet final.

**Durée :** 2h – **Prérequis :** TP 1 (poste préparé), un compte GitHub par étudiant – **Travail en binôme** (étudiant A et étudiant B)

---

## Étape 1 : Configurer Git (chacun)

```bash
git config --global user.name "Prénom Nom"
git config --global user.email "vous@exemple.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --list | grep -E 'user|alias'
```

Authentification GitHub : créez une clé SSH et ajoutez-la dans *GitHub → Settings → SSH and GPG keys*.
```bash
ssh-keygen -t ed25519 -C "vous@exemple.com"
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com          # "Hi <login>! You've successfully authenticated"
```

## Étape 2 : Créer le dépôt (étudiant A)

1. Sur GitHub : **New repository** → `legeni-webapp`, public, **sans** README.
2. En local :
	```bash
	cp -r devops_b3/app-fil-rouge ~/legeni-webapp && cd ~/legeni-webapp
	git init
	git status                        # tout est "Untracked"
	```
3. Créez un `.gitignore` **avant** le premier commit :
	```gitignore
	__pycache__/
	.venv/
	.pytest_cache/
	reports/
	.env
	```
4. Premier commit et publication :
	```bash
	git add .
	git status                        # tout est "Changes to be committed"
	git commit -m "Application fil rouge : version initiale"
	git remote add origin git@github.com:<login_A>/legeni-webapp.git
	git push -u origin main
	```
5. Sur GitHub : **Settings → Collaborators** → invitez l'étudiant B.

## Étape 3 : Cloner (étudiant B)

```bash
git clone git@github.com:<login_A>/legeni-webapp.git ~/legeni-webapp && cd ~/legeni-webapp
git lg
```

## Étape 4 : Une fonctionnalité dans une branche (étudiant B)

```bash
git switch -c feature/titre
sed -i 's/<h1>Legeni WebApp<\/h1>/<h1>Legeni WebApp – B3<\/h1>/' app.py
git diff                              # ce qui a changé
git add -p                            # indexer morceau par morceau (répondre y)
git commit -m "Ajoute la promotion dans le titre"
git push -u origin feature/titre
```
Sur GitHub : **Compare & pull request** → décrivez le *pourquoi* du changement → **Create pull request**.

## Étape 5 : La revue (étudiant A)

1. Onglet **Files changed** : lisez le diff, ajoutez un commentaire sur une ligne.
2. **Review changes → Approve**, puis **Merge pull request** → **Delete branch**.
3. En local :
	```bash
	git switch main && git pull
	git lg                            # le commit de B est sur main
	```

## Étape 6 : Provoquer et résoudre un conflit (les deux)

Les deux étudiants partent de `main` à jour et modifient **la même ligne** : la couleur par défaut dans `app.py` (`os.getenv("APP_COLOR", "#1f6feb")`).

- **A** : branche `feature/couleur-verte`, couleur `#2da44e`, commit, push, PR, **merge**.
- **B** : branche `feature/couleur-violette`, couleur `#8250df`, commit, push, PR → GitHub indique *This branch has conflicts*.

Résolution en local (étudiant B) :
```bash
git switch feature/couleur-violette
git pull origin main                  # rebase : le conflit apparaît
git status                            # "both modified: app.py"
```
Ouvrez `app.py` :
```text
<<<<<<< HEAD
            color=os.getenv("APP_COLOR", "#2da44e"),
=======
            color=os.getenv("APP_COLOR", "#8250df"),
>>>>>>> Couleur violette
```
Choisissez (ensemble) la bonne valeur, **supprimez les trois marqueurs**, puis :
```bash
git add app.py
git rebase --continue
docker build --target test .          # les tests passent-ils toujours ?
git push --force-with-lease           # la branche a été réécrite par le rebase (branche personnelle uniquement !)
```
La PR redevient fusionnable : faites-la relire et fusionner.

## Étape 7 : Annuler proprement un commit partagé

```bash
git switch main && git pull
echo "ligne de trop" >> requirements.txt
git commit -am "Erreur volontaire" && git push
git revert HEAD --no-edit             # crée un commit inverse
git push
git lg
```
Pourquoi `git revert` et pas `git reset --hard HEAD~1 && git push --force` sur `main` ?

## Étape 8 : Marquer la version

```bash
git tag -a v1.0.0 -m "Première version livrable"
git push origin v1.0.0
```
Sur GitHub : **Releases → Draft a new release** → tag `v1.0.0` → **Publish release**.

## Étape 9 : Protéger `main`

*Settings → Branches → Add branch ruleset* (ou *Add rule*) → branche `main` :
- *Require a pull request before merging* (1 approbation)
- *Block force pushes*

Essayez un `git push` direct sur `main` : il est refusé. (Au TP 3.6, on exigera en plus un build Jenkins vert.)

---

## ✅ Vérifications

- [ ] Dépôt `legeni-webapp` avec un `.gitignore` et au moins 5 commits aux messages explicites
- [ ] Une pull request relue (commentaire + approbation) et fusionnée
- [ ] Un conflit résolu (capture des marqueurs et de la PR fusionnée)
- [ ] Un commit annulé par `git revert`
- [ ] Release `v1.0.0` publiée et `main` protégée

## Questions (compte-rendu)

1. Quelle différence entre `git pull --rebase` et un `git pull` avec merge ? Dessinez l'historique obtenu dans les deux cas.
2. Pourquoi `--force-with-lease` est-il acceptable sur une branche personnelle et dangereux sur `main` ?
3. Un mot de passe a été commité il y a 3 commits. Que faites-vous ?

## 🎯 Défi

Créez un alias `git undo` qui annule le dernier commit **local non poussé** en gardant les modifications dans le répertoire de travail, et un hook `pre-commit` qui refuse un commit si `flake8` signale une erreur.

<details>
<summary>Solution</summary>

```bash
git config --global alias.undo "reset --soft HEAD~1"

cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
docker run --rm -v "$PWD":/src -w /src python:3.12-slim \
  sh -c "pip install -q flake8==7.1.1 && flake8 ." || {
  echo "flake8 a trouvé des erreurs : commit refusé"; exit 1; }
EOF
chmod +x .git/hooks/pre-commit
```
Réponse à la question 3 : changer (révoquer) immédiatement le mot de passe ; le supprimer du code ne suffit pas, il reste dans l'historique (nettoyage éventuel avec `git filter-repo`).
</details>
