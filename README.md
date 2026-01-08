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

```
import pandas as pd
import json
from googleapiclient.discovery import build
from tabulate import tabulate
import matplotlib.pyplot as plt
import seaborn as sns

# Liste de correspondance des ID et noms de catégories (les plus courants)
CATEGORY_MAP = {
    "1": "Film & Animation",
    "2": "Autos & Vehicles",
    "10": "Music",
    "15": "Pets & Animals",
    "17": "Sports",
    "19": "Travel & Events",
    "20": "Gaming",
    "22": "People & Blogs",
    "23": "Comedy",
    "24": "Entertainment",
    "25": "News & Politics",
    "26": "Howto & Style",
    "27": "Education",
    "28": "Science & Technology",
    "29": "Nonprofits & Activism",
    "30": "Movies",
    "43": "Shows"
}

# ==========================================
# NOUVELLE FONCTION : AFFICHAGE DES CATÉGORIES
# ==========================================

def display_category_table():
    """Affiche le tableau des ID et des noms pour la référence."""
    print("\n--- TABLEAU DE RÉFÉRENCE DES CATÉGORIES YOUTUBE ---")
    table_data = [[id, name] for id, name in CATEGORY_MAP.items()]
    # Utilise tabulate pour un affichage propre
    print(tabulate(table_data, headers=["ID", "Catégorie"], tablefmt="fancy_grid"))

def get_category_name(category_id):
    """Convertit un ID en nom de catégorie."""
    return CATEGORY_MAP.get(category_id, f"ID Inconnu ({category_id})")

# ==========================================
# FONCTIONS D'ANALYSE (Reprise de la version précédente)
# ==========================================

def get_youtube_client():
    return build('youtube', 'v3', developerKey=API_KEY)

def get_current_trending_videos(youtube):
    """Récupère les tendances actuelles."""
    request = youtube.videos().list(
        part="snippet,statistics",
        chart="mostPopular",
        regionCode="FR",
        maxResults=50
    )
    response = request.execute()

    trending_list = []
    for item in response['items']:
        trending_list.append({
            'video_id': item['id'],
            'category_id': item['snippet']['categoryId']
        })
    return pd.DataFrame(trending_list)

def load_my_history(filepath):
    """Charge et nettoie l'historique."""
    try:
        with open(filepath, 'r', encoding='utf-8') as f:
            data = json.load(f)
    except FileNotFoundError:
        print(f"Erreur : Fichier {filepath} introuvable.")
        return pd.DataFrame()

    my_history = []
    for entry in data:
        if 'titleUrl' in entry and 'watch?v=' in entry['titleUrl']:
            video_id = entry['titleUrl'].split('watch?v=')[-1][:11]
            my_history.append({'video_id': video_id, 'date': entry['time']})
    return pd.DataFrame(my_history).drop_duplicates(subset=['video_id'])

def analyze_influence():
    youtube = get_youtube_client()

    df_trending = get_current_trending_videos(youtube)
    df_my_history = load_my_history(TAKEOUT_FILE)

    # Étape 1 : Calculer l'intersection
    videos_communes = df_my_history[df_my_history['video_id'].isin(df_trending['video_id'])]
    nb_communs = len(videos_communes)

    # Étape 2 : Récupérer les catégories de votre historique récent
    recent_ids = df_my_history['video_id'].head(50).tolist()
    res = youtube.videos().list(part="snippet", id=",".join(recent_ids)).execute()
    my_categories_list = [item['snippet']['categoryId'] for item in res['items']]

    df_my_cats = pd.DataFrame(my_categories_list, columns=['category_id'])

    # --- Préparation pour le graphique ---
    df_trending['category_name'] = df_trending['category_id'].apply(get_category_name)
    df_my_cats['category_name'] = df_my_cats['category_id'].apply(get_category_name)

    # VISUALISATION
    sns.set_theme(style="whitegrid")

    plt.figure(figsize=(14, 7))

    # Graphique 1 : Catégories en Tendance (Noms affichés)
    plt.subplot(1, 2, 1)
    # Utilisez 'category_name' pour les étiquettes
    df_trending['category_name'].value_counts().plot(kind='pie', autopct='%1.1f%%', title="Catégories en Tendance (France)")
    plt.ylabel('Proportion')

    # Graphique 2 : Mes Catégories (Noms affichés)
    plt.subplot(1, 2, 2)
    df_my_cats['category_name'].value_counts().plot(kind='pie', autopct='%1.1f%%', title="Mes Catégories (Audit Personnel)")
    plt.ylabel('Proportion')

    plt.tight_layout()
    plt.show()

    # CONCLUSION
    print(f"\n--- RÉSULTAT DE L'AUDIT ---")
    print(f"Vous avez regardé {nb_communs} vidéos qui sont actuellement dans le Top 50 Tendances.")
    influence_score = (nb_communs / 50) * 100
    print(f"Votre indice d'influence directe est de : {influence_score:.1f}%")

# ==========================================
# MAIN EXECUTION
# ==========================================

if __name__ == "__main__":
    display_category_table() # Affiche le tableau de référence au début
    analyze_influence()
```


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

```
import pandas as pd
import json
from googleapiclient.discovery import build
import matplotlib.pyplot as plt
import seaborn as sns

# ==========================================
# FONCTIONS TECHNIQUES
# ==========================================

def calculate_sensitivity_score(influence_percentage):
    """Calcule le score de 1 à 10."""
    score = influence_percentage / 5
    if score < 1: score = 1.0
    if score > 10: score = 10.0
    return round(score, 1)

def get_current_trending_videos(youtube):
    """Récupère le Top 50 YouTube France."""
    try:
        request = youtube.videos().list(
            part="snippet,statistics",
            chart="mostPopular",
            regionCode="FR",
            maxResults=50
        )
        response = request.execute()
        trending_list = [{'video_id': item['id'], 'category_id': item['snippet']['categoryId']} for item in response['items']]
        return pd.DataFrame(trending_list)
    except Exception as e:
        print(f"Erreur API YouTube : {e}")
        return pd.DataFrame()

def load_my_history(filepath):
    """Charge votre fichier Takeout."""
    try:
        with open(filepath, 'r', encoding='utf-8') as f:
            data = json.load(f)

        my_history = []
        for entry in data:
            if 'titleUrl' in entry:
                video_id = entry['titleUrl'].split('v=')[-1][:11]
                my_history.append({'video_id': video_id})
        return pd.DataFrame(my_history)
    except FileNotFoundError:
        print(f"❌ ERREUR : Le fichier '{filepath}' est introuvable. Glissez-le dans le dossier à gauche sur Colab.")
        return pd.DataFrame()

# ==========================================
# FONCTION PRINCIPALE (Celle qui affiche tout)
# ==========================================

def analyze_influence():
    print("🚀 Démarrage de l'analyse...\n")

    # 1. Connexion API
    youtube = build('youtube', 'v3', developerKey=API_KEY)

    # 2. Chargement des données
    df_trending = get_current_trending_videos(youtube)
    df_my_history = load_my_history(TAKEOUT_FILE)

    if df_trending.empty or df_my_history.empty:
        print("⚠️ Analyse impossible : données manquantes.")
        return

    # 3. Comparaison (Intersection)
    videos_communes = df_my_history[df_my_history['video_id'].isin(df_trending['video_id'])]
    nb_communs = len(videos_communes)

    # 4. Calcul des scores
    influence_perc = (nb_communs / 50) * 100
    sensi_score = calculate_sensitivity_score(influence_perc)

    # 5. AFFICHAGE DES RÉSULTATS DANS LA CONSOLE
    print("-" * 40)
    print("📊 RÉSULTATS DE L'AUDIT RGPD")
    print("-" * 40)
    print(f"✅ Vidéos analysées dans votre historique : {len(df_my_history)}")
    print(f"🔥 Vidéos en commun avec le Top 50 Tendances : {nb_communs}")
    print(f"🎯 Indice d'influence directe : {influence_perc}%")
    print(f"\n⚡ SCORE DE SENSIBILITÉ (1-10) : {sensi_score}/10")
    print("-" * 40)

    # 6. Petit message d'interprétation automatique
    if sensi_score <= 2:
        print("Interprétation : Vous êtes dans une 'Bulle de Niche'. L'algorithme vous connaît trop bien pour vous proposer du contenu généraliste.")
    elif sensi_score <= 5:
        print("Interprétation : Influence modérée. Vous suivez quelques tendances mais gardez un profil spécifique.")
    else:
        print("Interprétation : Profil 'Mainstream'. Vous êtes fortement synchronisé avec la culture populaire actuelle.")

# ==========================================
# LANCEMENT (Indispensable pour que ça s'affiche !)
# ==========================================
if __name__ == "__main__":
    analyze_influence()
```

- Interprétation analytique

Un score faible traduit une prédictibilité élevée du comportement utilisateur.
L’algorithme n’a plus besoin d’exposer ce profil aux tendances globales, car son engagement est déjà optimisé.
Cette situation correspond à une forme d’enfermement algorithmique : la personnalisation extrême maximise l’efficacité économique du système au détriment de la diversité informationnelle.

### 5.Analyse des tags

- Description technique

Ce dernier bloc extrait les tags des vidéos tendances et des vidéos récemment visionnées par l’utilisateur.
Il calcule l’intersection sémantique entre ces ensembles afin de mesurer une synchronisation thématique, indépendamment de l’identité exacte des vidéos.

```
import pandas as pd
from googleapiclient.discovery import build
import matplotlib.pyplot as plt


def get_tags_analysis():
    youtube = build('youtube', 'v3', developerKey=API_KEY)

    # 1. RÉCUPÉRER LES TAGS DES TENDANCES (L'air du temps)
    print("Analyse des thématiques tendances en France...")
    request = youtube.videos().list(part="snippet", chart="mostPopular", regionCode="FR", maxResults=50)
    res_trending = request.execute()

    trending_tags = []
    for item in res_trending['items']:
        tags = item['snippet'].get('tags', [])
        trending_tags.extend([t.lower() for t in tags])

    # 2. RÉCUPÉRER LES TAGS DE VOTRE HISTORIQUE (Votre bulle)
    print("Analyse de vos thématiques personnelles...")
    # On prend les 20 dernières vidéos pour voir l'influence récente
    df_history = load_my_history(TAKEOUT_FILE).head(20)
    my_ids = df_history['video_id'].tolist()

    res_my_videos = youtube.videos().list(part="snippet", id=",".join(my_ids)).execute()
    my_tags = []
    for item in res_my_videos['items']:
        tags = item['snippet'].get('tags', [])
        my_tags.extend([t.lower() for t in tags])

    # 3. CALCUL DE LA SYNCHRONISATION
    set_trending = set(trending_tags)
    set_my = set(my_tags)
    common_tags = set_trending.intersection(set_my)

    sync_score = (len(common_tags) / len(set_trending)) * 100 if set_trending else 0

    # 4. AFFICHAGE
    print(f"\n--- AUDIT DES TENDANCES CACHÉES ---")
    print(f"Mots-clés en commun avec les tendances : {list(common_tags)[:10]}...")
    print(f"Indice de Synchronisation Thématique : {sync_score:.2f}%")

    # Échelle de 1 à 10 pour la santé mentale
    mental_impact_score = round(sync_score / 2, 1) # Plus on est synchronisé, plus l'influence est forte
    if mental_impact_score > 10: mental_impact_score = 10.0

    print(f"Score d'exposition aux thématiques globales : {mental_impact_score}/10")

# Appel de la fonction
get_tags_analysis()
```
- Interprétation analytique

Cette analyse révèle une influence plus diffuse mais plus profonde.
Même en l’absence de consommation directe des contenus viraux, une forte convergence des thématiques indique une exposition aux mêmes mécanismes émotionnels (urgence, conflit, controverse).
Elle met en évidence une pollution thématique : l’influence de masse persiste sous une forme personnalisée, moins visible mais tout aussi structurante pour l’attention et l’état cognitif de l’utilisateur.

# Résultats et observations
L’analyse met en évidence :
- Une forte répétition de certains mots-clés
- Une homogénéisation progressive des contenus recommandés
- Une exposition différenciée selon les centres d’intérêt initiaux

Ces éléments confirment l’existence de bulles algorithmiques influençant la consommation de contenu.
# Conclusion : Vers une hygiène numérique éclairée
Ce projet montre comment un simple audit de données personnelles peut révéler des mécanismes algorithmiques invisibles.

Grâce à :
- l’accès aux données personnelles (RGPD),
- l’usage d’outils de programmation accessibles,
- l’utilisateur peut reprendre une partie du contrôle sur sa consommation numérique.

L’objectif n’est pas de rejeter YouTube, mais de développer une conscience critique face aux systèmes de recommandation.

# Perspectives d’amélioration
- Analyse temporelle de l’évolution des recommandations
- Comparaison entre plusieurs profils utilisateurs
- Visualisations avancées (graphes, réseaux de tags)
- Automatisation complète du pipeline

# Auteur
- Nenad Jovanovic
- Thi Hong Nhung Nguyen
