# 🎓 Documentation BUT 3 R&T - Parcours Cybersécurité

Bienvenue sur le dépôt central de documentation pour notre année de **BUT 3 Réseaux & Télécommunications (Parcours Cybersécurité)**.

Ce dépôt a pour but de centraliser, structurer et partager nos prises de notes, comptes-rendus de TP, fiches de révisions et documentations de SAÉ tout au long de l'année. En collaborant ici, nous créons une base de connaissances utile  ! 🚀

## 📂 Arborescence du dépôt

Le dépôt est organisé par matière (Ressources et SAÉ). Dans chaque dossier de matière, vous retrouverez généralement une sous-arborescence classique :

* `CM/` : Notes de Cours Magistraux

* `TD/` : Exercices et corrections de Travaux Dirigés

* `TP/` : Comptes-rendus, scripts et configurations de Travaux Pratiques

### 📚 Liste des Modules

* **R500** : Réseaux informatiques (Cisco, Python, etc.)

* **R503** : Ingénierie de systèmes télécoms & Maths CDP

* **R505** : Anglais 3A

* **R506** : Communication 3A

* **R507_Admin** : Administration système

* **R507_PPP** : Projet Personnel et Professionnel

* **R508** : Gestion de projets

* **R509** : Architectures sécurisées

* **SAE 501** : Concevoir, réaliser et présenter une solution technique

* **SAE 503** : Assurer la sécurisation et la supervision

## 🛠️ Comment contribuer ? (Workflow)

Puisque nous travaillons tous directement sur la branche **`main`**, il est crucial de bien suivre ces étapes pour éviter d'écraser le travail des autres.

### 1. Toujours se mettre à jour avant de travailler

Avant de commencer à rédiger quoi que ce soit, récupérez les dernières modifications des autres collaborateurs :

```
git pull origin main

```

### 2. Rédiger et organiser la documentation

* Toute la documentation est rédigée sous forme de **Notebooks Jupyter** (`.ipynb`).

* Respectez la convention de nommage des fichiers pour s'y retrouver facilement : `YYYY-MM-DD_Titre_du_document.ipynb` (ex: `2026-09-22_TP_Routage_Python.ipynb`).

* Placez votre fichier `.ipynb` dans le bon sous-dossier de la bonne matière (ex: `R500_Reseaux_Informatiques/TP/`).

### 3. Sauvegarder et envoyer (Commit & Push)

Une fois votre travail terminé ou bien avancé, envoyez-le sur le dépôt :

```
git add .
git commit -m "Ajout du CR de TP sur les sockets en Python (R500)"
git push origin main

```

*Note : Si la commande `push` est refusée, c'est que quelqu'un d'autre a ajouté des fichiers entre temps. Refaites un `git pull origin main`, puis relancez votre `git push`.*

## ✍️ Règles de rédaction (Notebooks Jupyter)

* **Cellules Markdown :** Utilisez-les pour structurer votre document (titres `#`, `##`), rédiger vos explications, vos observations de TP et inclure des formules ou des tableaux.

* **Cellules de Code :** Utilisez-les pour intégrer vos scripts Python, commandes d'administration ou requêtes.

* **Outputs :** Pensez à exécuter vos cellules de code avant de faire votre *commit*. GitHub affiche les résultats d'exécution des cellules, ce qui est très pratique pour relire un TP sans avoir à relancer le code !

* **Images :** Si vous devez intégrer des captures d'écran (ex: topologies Cisco, requêtes Wireshark), placez les images dans un sous-dossier `img/` à l'intérieur du dossier de votre module, et appelez-les dans une cellule Markdown avec la syntaxe standard `![Description](img/nom_image.png)`.