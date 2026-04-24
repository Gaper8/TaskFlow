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

## Job kubernetees NON FAIT

Puisque nous utilisons minicube qui tourne en local sur la machine, on ne peut pas faire de job github qui met à jour les settings local de la machine. 

## Artifacts générés

- `audit-report` (job `audit`)
- `trivy-report` (job `build`)
