# Mini-projet CyberRange Diateam

## Présentation

Le CyberRange Diateam permet de construire et d’exécuter des topologies réseau et système virtualisées. Le mini-projet commence par la prise en main de la plateforme, puis met en pratique ces notions autour d’attaques de niveau 2 sur un réseau local.

## Accéder au CyberRange

### Prérequis réseau

1. Brancher le câble Ethernet **avant de démarrer la machine**. Sans cette connexion au démarrage, la machine ne disposera pas de la route vers le serveur.
2. Sur la station, vérifier que la route vers `172.20.9.0/24` passe par le routeur `148.60.211.64` ou `148.60.210.239`, selon l’adresse IP locale.

### Connexion avec Hyneview

Le client lourd **Hyneview 4.8.14** se trouve sur le bureau. Lancer l’application et utiliser les paramètres suivants :

| Champ | Valeur |
| --- | --- |
| Serveur / IP | `172.20.9.1` |
| Login | `but3fi` |
| Mot de passe | `bonjour1234` |

Le compte `but3fi` est partagé entre les étudiants : **ne pas modifier son mot de passe**. Le client permet également d’ouvrir la documentation de l’outil au format PDF.

## Phase 1 — Prise en main

L’objectif est de savoir manipuler les entités et construire des topologies réutilisables. La plateforme fournit notamment des routeurs, commutateurs et systèmes.

À maîtriser :

- créer, démarrer, arrêter, cloner et supprimer des entités ;
- créer une topologie et y ajouter des entités ;
- interconnecter les entités pour former un scénario réseau ;
- installer Debian 12 sur une entité et vérifier sa connexion à Internet ;
- organiser les entités dans le catalogue et adopter une stratégie de nommage des topologies ;
- cloner une entité dans une nouvelle topologie.

### Paramètres réseau pour Debian 12

Le sujet indique les paramètres suivants pour l’accès à Internet depuis l’entité Debian :

| Paramètre | Valeur |
| --- | --- |
| Plage réseau | `172.22.255.0/24` |
| Passerelle par défaut | `172.22.255.254` |
| DNS indiqué dans le sujet | `129.20.211.22/23` |

## Phase 2 — Étude d’attaques de niveau 2

Construire une topologie simple de réseau local et étudier un ou plusieurs des scénarios proposés dans le sujet :

- empoisonnement de cache ARP ;
- DHCP starvation ou rogue DHCP.

Pour chaque scénario étudié, comprendre le mécanisme, mettre en place les entités nécessaires, réaliser l’attaque dans la topologie et en documenter le déroulement. Les procédures d’installation des services et outils seront documentées séparément.

### Éléments à documenter

Illustrer la mise en œuvre à l’aide d’éléments observables, par exemple :

- captures de trafic réalisées avec Wireshark ;
- journaux des systèmes concernés ;
- explication des étapes de l’attaque et de leurs effets sur le réseau ;
- indicateurs de compromission observés.

Exemples d’indicateurs mentionnés dans le sujet : plusieurs offres DHCP provenant d’adresses MAC inhabituelles, des baux DHCP anormalement courts ou des changements répétés (« flapping ») des entrées ARP.
