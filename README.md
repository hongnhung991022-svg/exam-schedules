# Analyse des Données YouTube – Technique de Programmation
## Contexte du projet
À l’ère des plateformes numériques, YouTube se présente comme un service capable d’offrir une expérience de consommation de contenu entièrement personnalisée, supposément alignée sur les préférences individuelles de chaque utilisateur. Cette promesse de personnalisation s’accompagne toutefois d’une transformation profonde de l’interface et des modes d’accès à l’information.

L’onglet « Tendances », autrefois commun à l’ensemble des utilisateurs et garant d’une certaine transparence collective, a progressivement été marginalisé, voire remplacé, par un flux algorithmique individualisé. Cette évolution rend de moins en moins perceptible l’existence de dynamiques globales de diffusion de contenus. L’utilisateur a alors le sentiment de naviguer librement dans un environnement riche et diversifié, alors même que ses choix sont en partie orientés par des mécanismes algorithmiques invisibles.

Ces mécanismes, conçus pour maximiser le temps d’engagement et la rétention de l’attention, soulèvent des enjeux majeurs en matière de santé mentale, de bien-être psychologique et de liberté de choix. Le paradoxe réside dans le fait que plus l’algorithme se prétend « personnel », plus son fonctionnement réel échappe à la compréhension de l’utilisateur.

## Objectifs
Ce projet a pour objectif de rendre visible l’influence des recommandations algorithmiques de YouTube à partir des données personnelles de l’utilisateur. En exploitant les données YouTube Takeout et la programmation Python, il vise à :
- Analyser l’historique de visionnage de manière objective.
- Identifier les thématiques et tendances dominantes dans les contenus recommandés.
- Évaluer le degré d’exposition à des dynamiques potentiellement problématiques.
- Favoriser une prise de recul critique face aux mécanismes de personnalisation.

Le projet s’inscrit dans une démarche d’audit algorithmique et d’hygiène numérique, permettant à l’utilisateur de mieux comprendre son environnement informationnel.

## Technologies utilisées
- Python
- Fichiers JSON
- Pandas : manipulation des données
- YouTube Data API v3
- Jupyter Notebook (pour la restitution et la visualisation)

##  Structure du projet
- Installation et configuration initiale
- Analyse comportementale profonde
- Audit d'influence directe (Score de Sensibilité)
- Calcul du Score de Sensibilité (1-10)
- Analyse sémantique des tags (Influence Thématique)
- Conclusion

## Description technique du script
### 1.Création et chargement de la base de données locale
- Description technique

Ce premier bloc met en place l’environnement logiciel du projet.
Il importe les bibliothèques essentielles : Pandas pour la manipulation des données, Matplotlib / Seaborn pour la visualisation, et google-api-python-client pour l’accès à l’API YouTube.
La clé API permet l’authentification des requêtes et l’accès aux métadonnées normalement non exposées à l’utilisateur.

- Interprétation analytique

Cette étape constitue le socle de l’appropriation des données personnelles.
En configurant ces outils, l’utilisateur ne se contente plus de consommer une interface, mais se dote des moyens techniques pour analyser le fonctionnement algorithmique sous-jacent.
L’usage de l’API YouTube est central : il donne accès à des informations structurelles (classements, catégories, métadonnées) absentes de l’interface publique, rendant possible un audit indépendant du système de recommandation.

### 2.Détection des nouvelles vidéos

- Description technique

Ce code charge le fichier watch-history.json issu de Google Takeout et enrichit chaque vidéo à l’aide de l’API YouTube (catégorie, durée, vues, tags).
Un mécanisme de mise à jour incrémentale est implémenté afin de limiter la consommation du quota API : seules les vidéos non encore analysées sont traitées.
Les visualisations produites portent sur les chaînes dominantes, la répartition horaire du visionnage et la popularité des contenus consommés.

![Ma bulle](Ma_bulle.png)

- Interprétation analytique

Cette analyse met en évidence les régularités comportementales de consommation.
La distribution horaire révèle des périodes de forte exposition algorithmique, tandis que la concentration sur un nombre restreint de chaînes matérialise l’existence d’une bulle de filtrage.
L’analyse du niveau de popularité (contenus mainstream vs niche) permet d’évaluer le degré d’exposition aux logiques virales et aux mécanismes de comparaison sociale associés.

### 3.Récupération des données via l’API YouTube

- Description technique

Ce bloc récupère le Top 50 des vidéos tendances en France via l’API YouTube et calcule leur intersection avec l’historique personnel de l’utilisateur.
Il génère des diagrammes circulaires comparant les catégories de contenus tendances et celles effectivement consommées.
La structure CATEGORY_MAP permet de traduire les identifiants numériques en catégories lisibles.

- Interprétation analytique

Cette étape répond à la question centrale de l’influence visible des tendances.
Un taux d’intersection nul indique l’absence de consommation directe des vidéos tendances, non pas comme signe d’indépendance, mais comme indicateur d’une hyper-personnalisation algorithmique.
L’utilisateur est isolé du flux grand public au profit d’un environnement parfaitement ajusté à son profil, réduisant la diversité cognitive et la sérendipité.

### 4.Fusion et mise à jour de la base
- Description technique

Ce code synthétise les résultats précédents en un indicateur unique.
Le pourcentage d’influence directe est converti en score sur 10, puis interprété automatiquement selon trois niveaux :
Bulle de Niche (≤2), Influence Modérée (3–5), Mainstream (≥6) .

- Interprétation analytique

Un score faible traduit une prédictibilité élevée du comportement utilisateur.
L’algorithme n’a plus besoin d’exposer ce profil aux tendances globales, car son engagement est déjà optimisé.
Cette situation correspond à une forme d’enfermement algorithmique : la personnalisation extrême maximise l’efficacité économique du système au détriment de la diversité informationnelle.

### 5.Analyse des tags

- Description technique

Ce dernier bloc extrait les tags des vidéos tendances et des vidéos récemment visionnées par l’utilisateur.
Il calcule l’intersection sémantique entre ces ensembles afin de mesurer une synchronisation thématique, indépendamment de l’identité exacte des vidéos.

- Interprétation analytique

Cette analyse révèle une influence plus diffuse mais plus profonde.
Même en l’absence de consommation directe des contenus viraux, une forte convergence des thématiques indique une exposition aux mêmes mécanismes émotionnels (urgence, conflit, controverse).
Elle met en évidence une pollution thématique : l’influence de masse persiste sous une forme personnalisée, moins visible mais tout aussi structurante pour l’attention et l’état cognitif de l’utilisateur.

