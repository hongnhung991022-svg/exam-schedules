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

```
# Cellule 1 : Installation des outils
!pip install pandas matplotlib seaborn google-api-python-client isodate
```

- Interprétation analytique

Cette étape constitue le socle de l’appropriation des données personnelles.
En configurant ces outils, l’utilisateur ne se contente plus de consommer une interface, mais se dote des moyens techniques pour analyser le fonctionnement algorithmique sous-jacent.
L’usage de l’API YouTube est central : il donne accès à des informations structurelles (classements, catégories, métadonnées) absentes de l’interface publique, rendant possible un audit indépendant du système de recommandation.

### 2.Détection des nouvelles vidéos

- Description technique

Ce code charge le fichier watch-history.json issu de Google Takeout et enrichit chaque vidéo à l’aide de l’API YouTube (catégorie, durée, vues, tags).
Un mécanisme de mise à jour incrémentale est implémenté afin de limiter la consommation du quota API : seules les vidéos non encore analysées sont traitées.
Les visualisations produites portent sur les chaînes dominantes, la répartition horaire du visionnage et la popularité des contenus consommés.

```
import pandas as pd
import json
import os
from googleapiclient.discovery import build
import matplotlib.pyplot as plt
import seaborn as sns
from datetime import datetime

# ==========================================
# CONFIGURATION
# ==========================================
API_KEY = "AIzaSyAWr5iPhQC3U5Af_Ts5bf8qPc6BP_rTJH0" # À ne jamais mettre sur Git ! Utilisez un fichier config ou env.
DB_FILE = "local_video_database.csv"
TAKEOUT_FILE = "watch-history.json" # Le fichier téléchargé depuis Google

# ==========================================
# MODULE 1 : GESTION DES DONNÉES ET MISE À JOUR
# ==========================================

def load_takeout_data(filepath):
    """Charge et nettoie les données brutes de Google Takeout."""
    print(f"Chargement de {filepath}...")
    try:
        with open(filepath, 'r', encoding='utf-8') as f:
            data = json.load(f)
    except FileNotFoundError:
        print("Erreur : Fichier Takeout introuvable.")
        return pd.DataFrame()

    clean_data = []
    for entry in data:
        # On ne garde que les entrées qui sont des vidéos (pas les pubs ou visites de chaine)
        if 'titleUrl' in entry and 'watch?v=' in entry['titleUrl']:
            video_id = entry['titleUrl'].split('watch?v=')[1]
            # Gestion de la date
            watch_time = entry['time'] # Format ISO
            clean_data.append({
                'video_id': video_id,
                'title': entry['title'].replace("Vous avez regardé ", ""),
                'watch_time': watch_time
            })

    return pd.DataFrame(clean_data)

def get_video_details(video_ids, api_key):
    """Interroge l'API YouTube pour récupérer les catégories (Enrichissement)."""
    youtube = build('youtube', 'v3', developerKey=api_key)
    video_details = []

    # L'API accepte max 50 IDs par requête
    chunk_size = 50
    for i in range(0, len(video_ids), chunk_size):
        chunk = video_ids[i:i+chunk_size]
        request = youtube.videos().list(
            part="snippet,contentDetails,statistics",
            id=','.join(chunk)
        )
        response = request.execute()

        for item in response['items']:
            video_details.append({
                'video_id': item['id'],
                'category_id': item['snippet']['categoryId'],
                'channel_title': item['snippet']['channelTitle'],
                'tags': item['snippet'].get('tags', []), # Les tags sont clés pour comprendre les tendances
                'duration': item['contentDetails']['duration'],
                'view_count': item['statistics'].get('viewCount', 0)
            })

    return pd.DataFrame(video_details)

def update_database(new_df):
    """Mécanisme de mise à jour incrémentale."""
    # 1. Charger la base existante si elle existe
    if os.path.exists(DB_FILE):
        existing_db = pd.read_csv(DB_FILE)
        print(f"Base existante chargée : {len(existing_db)} vidéos.")
    else:
        existing_db = pd.DataFrame(columns=['video_id'])
        print("Création d'une nouvelle base de données locale.")

    # 2. Identifier les vidéos inconnues (qu'on n'a pas encore analysées)
    # On compare les IDs du Takeout avec ceux de la DB locale
    merged = new_df.merge(existing_db, on='video_id', how='left', indicator=True)
    unknown_videos = merged[merged['_merge'] == 'left_only']['video_id'].unique()

    print(f"Nouvelles vidéos détectées à analyser via API : {len(unknown_videos)}")

    if len(unknown_videos) > 0:
        # Attention aux quotas API (limitez si nécessaire pour les tests)
        # Pour l'exemple, on limite à 200 nouvelles vidéos pour ne pas griller le quota gratuit
        details_df = get_video_details(list(unknown_videos)[:200], API_KEY)

        # Fusionner et sauvegarder
        full_db = pd.concat([existing_db, details_df], ignore_index=True).drop_duplicates(subset=['video_id'])
        full_db.to_csv(DB_FILE, index=False)
        print("Base de données mise à jour avec succès.")
        return full_db
    else:
        print("Aucune nouvelle donnée API nécessaire.")
        return existing_db

# ==========================================
# MODULE 2 : ANALYSE ET VISUALISATION
# ==========================================

def analyze_trends(history_df, metadata_df):
    """Crée les graphiques pour la présentation."""

    # Fusionner l'historique (QUAND j'ai regardé) avec les métadonnées (QUOI)
    df = history_df.merge(metadata_df, on='video_id', how='inner')

    # Convertir le temps
    df['watch_time'] = pd.to_datetime(df['watch_time'], format='ISO8601') # Added format='ISO8601'
    df['hour'] = df['watch_time'].dt.hour
    df['day_of_week'] = df['watch_time'].dt.day_name()

    # Configuration du style
    sns.set_theme(style="whitegrid")

    # --- Graphique 1 : Ma "Bulle" (Catégories) ---
    # Note: L'API renvoie des ID de catégories (ex: '10' = Music).
    # Il faudrait une map complète, ici simplifié.
    plt.figure(figsize=(10, 6))
    top_channels = df['channel_title'].value_counts().head(10)
    sns.barplot(x=top_channels.values, y=top_channels.index, palette="viridis")
    plt.title("Les 10 chaînes qui dominent mon flux (Ma Bulle)")
    plt.xlabel("Nombre de vues")
    plt.tight_layout()
    plt.show()

    # --- Graphique 2 : Chronobiologie de l'addiction ---
    plt.figure(figsize=(10, 6))
    sns.histplot(df['hour'], bins=24, kde=True, color="red")
    plt.title("À quelle heure l'algorithme me capte-t-il le mieux ?")
    plt.xlabel("Heure de la journée (0-24)")
    plt.ylabel("Fréquence")
    plt.tight_layout()
    plt.show()

    # --- Graphique 3 : Mainstream vs Niche ---
    # Analyse basée sur le nombre de vues global des vidéos regardées
    df['view_count'] = pd.to_numeric(df['view_count'])

    plt.figure(figsize=(10, 6))
    sns.boxplot(x=df['view_count'])
    plt.xscale('log') # Échelle logarithmique car les vues varient énormément
    plt.title("Distribution des vues : Suis-je un consommateur de contenu 'Viral' ?")
    plt.xlabel("Vues totales de la vidéo (Échelle Log)")
    plt.tight_layout()
    plt.show()

# ==========================================
# MAIN
# ==========================================

if __name__ == "__main__":
    # 1. Charger l'historique brut
    history = load_takeout_data(TAKEOUT_FILE)

    if not history.empty:
        # 2. Mettre à jour les métadonnées (API)
        metadata = update_database(history)

        # 3. Lancer l'analyse
        analyze_trends(history, metadata)
    else:
        print("Veuillez placer le fichier watch-history.json dans le dossier.")
```

![](Ma_Bulle.png)

![](Chronobiologie_de_l'addiction.png)

![](MainstreamvsNiche.png)

- Interprétation analytique

Cette analyse met en évidence les régularités comportementales de consommation.
La distribution horaire révèle des périodes de forte exposition algorithmique, tandis que la concentration sur un nombre restreint de chaînes matérialise l’existence d’une bulle de filtrage.
L’analyse du niveau de popularité (contenus mainstream vs niche) permet d’évaluer le degré d’exposition aux logiques virales et aux mécanismes de comparaison sociale associés.

### 3.Récupération des données via l’API YouTube

- Description technique

Ce bloc récupère le Top 50 des vidéos tendances en France via l’API YouTube et calcule leur intersection avec l’historique personnel de l’utilisateur.
Il génère des diagrammes circulaires comparant les catégories de contenus tendances et celles effectivement consommées.
La structure CATEGORY_MAP permet de traduire les identifiants numériques en catégories lisibles.

![](Catégories_en_Tendance.png)


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

