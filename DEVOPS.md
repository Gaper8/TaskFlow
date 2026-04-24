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

Par rapport à cette analyse, nous partons donc sur la version 20-alpine. Elle est légèrement plus lourde mais présente beaucoup moins de faiblesse. Trivy nous a renvoyé les données que nous voyons plus haut et ducoup nous avons géré ce retour en choisissant la version qui était la deuxième moins lourde et qui contenait le moins d'erreur.

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

# Partie 2 CI/CD - Jobs actuels - Choix 3

Nous avons choisi deux stratégies différentes pour staging et production.

### Staging
Nous avons retenu :

- `maxUnavailable: 1`
- `maxSurge: 1`

Cette stratégie offre un bon équilibre entre vitesse de déploiement et disponibilité.  
Pendant une mise à jour, Kubernetes peut rendre un pod indisponible tout en créant un pod supplémentaire temporairement.  
Cela permet d’avoir un rolling update assez rapide sans consommer trop de ressources.

Nous avons choisi cette option pour le staging car cet environnement sert surtout à tester les déploiements, valider les changements et démontrer le rolling update.  
Nous acceptons donc un peu plus de risque qu’en production, tant que le service reste globalement disponible.

### Production
Nous avons choisis :

- `maxUnavailable: 0`
- `maxSurge: 1`

Cette stratégie privilégie la disponibilité.  
Aucun pod existant n’est retiré avant qu’un nouveau pod soit prêt, ce qui limite fortement le risque d’interruption de service.  
Le déploiement est un peu plus lent, mais c’est un compromis acceptable pour la production.

# Partie 3 Kubernetes Choix 4

## Choix 4 — Nombre de replicas

Nous avons choisi :

- 2 replicas en staging
- 3 replicas en production

### Justification pour le staging
Le staging sert à tester les déploiements.
Avec 2 replicas, nous pouvons remplacer un pod sans rendre l’application totalement indisponible.  
Avec 1 replica seulement, un rolling update avec `maxUnavailable: 1` peut provoquer une interruption de service, car l’unique pod peut être retiré avant que le nouveau soit prêt.  

### Justification pour la production
Nous avons choisi 3 replicas en production, car le sujet demande au minimum 3 replicas sur cet environnement.  
Ce choix améliore la disponibilité de l’application, permet une meilleure tolérance aux pannes.
Si un pod tombe, l’application reste accessible grâce aux autres replicas.