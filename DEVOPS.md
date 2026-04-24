# Partie 1 - Conteneurisation

## Choix 1 — Politique de redémarrage dans Docker Compose

### La taille et les CVE des versions : 

Version 18 alpine : 

Taille : 187,19 MB

Node.js (node-pkg)
==================
Total: 14 (UNKNOWN: 0, LOW: 2, MEDIUM: 1, HIGH: 11, CRITICAL: 0)

taskflow-backend:latest (alpine 3.21.3)
=======================================
Total: 50 (UNKNOWN: 0, LOW: 5, MEDIUM: 26, HIGH: 15, CRITICAL: 4)

Version 18 slim : 

Taille : 284,8 MB

Node.js (node-pkg)
==================
Total: 14 (UNKNOWN: 0, LOW: 2, MEDIUM: 1, HIGH: 11, CRITICAL: 0)

taskflow-backend:18slim

166 vulnérabilités

Version 20 alpine : 

Taille : 199,43 MB

Node.js (node-pkg)
==================
Total: 14 (UNKNOWN: 0, LOW: 2, MEDIUM: 1, HIGH: 11, CRITICAL: 0)

taskflow-backend:20alpine

14 vulnérabilités

### Explication du choix

Par rapport à cette analyse, nous partons donc sur la version 20-alpine. Elle est légèrement plus lourde mais présente beaucoup moins de faille.

## Choix 2 — Politique de redémarrage dans Docker Compose

Nous avons choisi d'utiliser `unless-stopped`.

### Pourquoi ce choix
Cette politique redémarre automatiquement les services s’ils plantent, ce qui améliore la disponibilité de l’application. En revanche, si un arrêt est effectué manuellement, Docker respecte cet arrêt et ne relance pas automatiquement le conteneur. Nous avons choisi cette option car elle offre un bon compromis entre on failure et always.

### Comparaison avec les autres options
- `on-failure` : redémarre uniquement en cas d’erreur.
- `always` : redémarre dans tous les cas, même après un arrêt manuel. Cette option maximise la reprise automatique, mais elle laisse moins de contrôle.

### Si le backend plante à 3h du matin
- avec `unless-stopped`, il redémarre automatiquement ;
- avec `on-failure`, il redémarre uniquement si le conteneur s’est arrêté sur une erreur ;
- avec `always`, il redémarre dans tous les cas, même après un arrêt manuel.

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

**Objectif**: construire les images Docker backend et frontend, scanner leur sécurité, puis pousser les deux images.

- Runner: `ubuntu-latest`
- Dépendances: `needs: [test, lint]`
- Condition d'exécution:
	- uniquement si la référence Git est un tag commençant par `v`
	- condition: `startsWith(github.ref, 'refs/tags/v')`
- Étapes:
	- Checkout du dépôt
	- Lecture du tag Git dans une variable de sortie (`steps.vars.outputs.tag`)
	- Build de l'image backend depuis `backend/Dockerfile`
	- Build de l'image frontend depuis `frontend/Dockerfile`
	- Scan Trivy backend avec rapport JSON (`trivy-report.json`)
		- Étape en `continue-on-error: true` pour toujours publier le rapport
	- Upload de l'artifact `trivy-report`
	- Échec si vulnérabilités `CRITICAL` sur l'image backend
	- Scan Trivy frontend avec rapport JSON (`trivy-report-frontend.json`)
		- Étape en `continue-on-error: true` pour toujours publier le rapport
	- Upload de l'artifact `trivy-report-frontend`
	- Échec si vulnérabilités `CRITICAL` sur l'image frontend
	- Login Docker Hub via secrets GitHub Actions
	- Push de l'image backend sur Docker Hub
	- Push de l'image frontend sur Docker Hub

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
- `trivy-report` (job `build`, image backend)
- `trivy-report-frontend` (job `build`, image frontend)
