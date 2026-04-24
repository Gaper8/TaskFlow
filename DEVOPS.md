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
