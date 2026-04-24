# Mini doc CI/CD - Jobs actuels

Ce document résume les jobs définis dans `.github/workflows/ci.yml`.

## Déclenchement du workflow

- Sur `push`
- Sur `pull_request`

## Job `test`

**Objectif**: exécuter les tests backend sur plusieurs versions de Node.js.

- Runner: `ubuntu-latest`
- Stratégie matrix: Node.js `18` et `20`
- Étapes:
	- Checkout du dépôt
	- Setup Node.js (version matrix)
	- Installation des dépendances backend (`npm ci`)
	- Lancement des tests backend (`npm test`)

## Job `lint`

**Objectif**: vérifier le style et la qualité du code backend.

- Runner: `ubuntu-latest`
- Dépendance: `needs: test`
- Node.js: version `18`
- Étapes:
	- Checkout du dépôt
	- Setup Node.js 18
	- Installation des dépendances backend (`npm ci`)
	- Exécution du lint (`npm run lint`)

## Job `audit`

**Objectif**: analyser les vulnérabilités des dépendances backend et conserver un rapport.

- Runner: `ubuntu-latest`
- Dépendance: `needs: test`
- Node.js: version `18`
- Étapes:
	- Checkout du dépôt
	- Setup Node.js 18
	- Installation des dépendances backend (`npm ci`)
	- Génération d'un rapport audit JSON (`npm audit --json > audit-report.json`)
		- Cette étape est en `continue-on-error: true` pour permettre l'upload du rapport même en cas de vulnérabilités
	- Upload de l'artifact `audit-report`
	- Échec du job si vulnérabilités de niveau `high` ou plus (`npm audit --audit-level=high`)

## Job `build` (Build, Scan & Push)

**Objectif**: construire l'image Docker, scanner la sécurité, puis pousser l'image.

- Runner: `ubuntu-latest`
- Dépendances: `needs: [test, lint]`
- Condition d'exécution:
	- uniquement si la référence Git est un tag commençant par `v`
	- condition: `startsWith(github.ref, 'refs/tags/v')`
- Étapes:
	- Checkout du dépôt
	- Lecture du tag Git dans une variable de sortie (`steps.vars.outputs.tag`)
	- Build de l'image Docker taggée
	- Scan Trivy avec rapport JSON (`trivy-report.json`)
		- Étape en `continue-on-error: true` pour toujours publier le rapport
	- Upload de l'artifact `trivy-report`
	- Échec si vulnérabilités `CRITICAL`
	- Login Docker Hub via secrets GitHub Actions
	- Push de l'image Docker sur Docker Hub

## Jobs Kubernetes (préparation à la mise en production)

Nous utilisons actuellement Minikube en local. Dans ce contexte, GitHub Actions ne peut pas piloter automatiquement la configuration Kubernetes locale d'une machine développeur.

Les jobs Kubernetes ont donc été préparés comme une base prête à l'emploi pour la bascule vers un vrai cluster (staging/production) dès que l'infrastructure distante sera disponible.

## Job `deploy-staging`

**Objectif**: déployer en environnement de staging après le build.

- Runner: `ubuntu-latest`
- Dépendance: `needs: build`
- Condition d'exécution:
	- uniquement sur `push` de la branche `staging`
- Étapes:
	- Checkout du dépôt
	- Installation de `kubectl`
	- Chargement du kubeconfig staging via secret (`KUBECONFIG_STAGING`)
	- Mise à jour de l'image du Deployment Kubernetes (`kubectl set image`)

## Job `smoke-test`

**Objectif**: vérifier que le déploiement staging est réellement fonctionnel.

- Runner: `ubuntu-latest`
- Dépendance: `needs: deploy-staging`
- Condition d'exécution:
	- uniquement sur `push` de la branche `staging`
- Vérifications effectuées:
	- Attente du rollout du Deployment (`kubectl rollout status`)
	- Test HTTP sur `GET /health` via `kubectl port-forward`
	- Échec si le code HTTP est différent de `200`
	- Échec si la réponse ne contient pas `"status": "ok"`

## Job `deploy-production`

**Objectif**: déployer en environnement de production à partir d'un tag.

- Runner: `ubuntu-latest`
- Dépendance: `needs: build`
- Condition d'exécution:
	- uniquement sur les tags Git (`refs/tags/*`)
- Étapes:
	- Checkout du dépôt
	- Installation de `kubectl`
	- Chargement du kubeconfig production via secret (`KUBECONFIG_PRODUCTION`)
	- Mise à jour du Deployment dans le namespace `production`
	- Attente du rollout (`kubectl rollout status`)

## Remarque importante

Le job `build` actuel ne s'exécute que sur les tags commençant par `v`. Par conséquent, les jobs `deploy-staging` et `smoke-test` ne pourront se lancer sur `staging` que si ce flux est adapté (ou si un build dédié staging est ajouté).

## Artifacts générés

- `audit-report` (job `audit`)
- `trivy-report` (job `build`)
