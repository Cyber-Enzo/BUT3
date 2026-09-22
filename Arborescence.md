# 📂 Structure du Dépôt BUT 3 R&T Cyber

Afin de garder notre documentation claire et facile à utiliser, voici l'arborescence officielle du dépôt. 
Tous nos documents doivent être rédigés au format **Jupyter Notebook (`.ipynb`)** et placés dans le bon dossier.

```text
BUT3-RT-Cyber-Doc/
├── README.md                        # Fichier d'accueil avec les règles de contribution
├── .gitignore                       # Règles d'exclusion (cache Python, checkpoints Jupyter, fichiers OS)
├── 📁 Templates/                    # Modèles de base pour harmoniser nos documents
│   ├── modele_cours.ipynb           # Structure type pour un cours magistral
│   └── modele_tp.ipynb              # Structure type pour un TP (avec cellules de code)
│
├── 📁 R500_Reseaux_Informatiques/
│   ├── 📁 CM_TD/                    # Notes de cours théoriques (Cisco, routage, etc.)
│   ├── 📁 TP_Cisco/                 # Notebooks pour documenter les configs routeurs/switchs
│   └── 📁 TP_Python/                # Notebooks exécutables pour les scripts Python
│
├── 📁 R503_Ingenierie_et_Maths/
│   ├── 📁 Systemes_Telecoms/        # Schémas et protocoles télécoms
│   └── 📁 Maths_CDP/                # Formules mathématiques (utilisez LaTeX dans les cellules Markdown)
│
├── 📁 R505_Anglais/
│   └── 📁 Vocabulaire_Cyber/        # Listes de termes techniques, préparations orales
│
├── 📁 R506_Communication/
│
├── 📁 R507_Administration_Systeme/
│   ├── 📁 CM_TD/                    # Théorie admin sys
│   └── 📁 TP/                       # Commandes Linux/Windows, scripts Bash/PowerShell
│
├── 📁 R507_PPP/                     # Projet Personnel et Professionnel (recherches stage/alternance)
│
├── 📁 R508_Gestion_Projets/
│   └── 📁 Methodologie/             # Outils, diagrammes de Gantt, méthodes agiles
│
├── 📁 R509_Architectures_Securisees/
│   └── 📁 TP_Pentest_Secu/          # Traces de commandes d'audit, analyses de trames, rapports de vulnérabilités
│
├── 📁 SAE501_Solution_Technique/
│   ├── 📁 Livrables/                # Rapports d'avancement et rapport final
│   └── 📁 Reunions/                 # Comptes-rendus de réunions de groupe
│
└── 📁 SAE503_Securisation_Supervision/
    ├── 📁 Architecture/             # Choix techniques, topologie et schémas réseau
    ├── 📁 Deploiement/              # Logs d'installation et configurations des services
    └── 📁 Reunions/                 # Comptes-rendus de suivi de projet
```

### 💡 Rappel sur le nommage des fichiers

Pour que les dossiers restent triés chronologiquement, merci d'utiliser cette convention pour vos fichiers `.ipynb` :
`YYYY-MM-DD_Titre_du_document.ipynb` (exemple : `2026-09-22_TP_Routage_Python.ipynb`).