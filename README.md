# Analyse de notre consommation Youtube 
### Mattéo CORREGER et Elia HUREL 

## Tout d'abord, il faut demander la base de données à Youtube, voici les étapes à suivre. 
1) Aller sur le site **Google Takeout** ;
2) Cliquer sur **"Tout désélectionner"** ;
3) Cocher **YouTube et YouTube Music** ;
4) Aller dans **"Formats multiples"** et sélectionner **"JSON"** pour le format de l'historique ;
5) Cliquer sur **"Étape suivante"** puis **"Créer une exportation"**
6) Vous recevrez un mail et vous téléchargerez un dossier nommé **"Takeout"**.

### Outil pour l'analyse de données, as pd pour écrire le raccourci 
```
import pandas as pd
```
### Outil pour transformer les chiffres en graphique
```
import matplotlib.pyplot as plt
```
### Outil pour chercher les dossiers sur notre ordinateur
```
import os
```
### Traducteur pour lire le format json
```
import json
```
### Outil qui cherche des fichiers dans le dossier de notre ordinateur
```
import glob 
```
# 1. Chargement de la base de données de manière universelle
```
dossier_racine = 'VOTRE CHEMIN VERS VOTRE DOSSIER TAKEOUT' 
```
### On cherche le fichier watch-history.json n'importe où dans le dossier
```
def trouver_et_charger_historique(racine):
    fichiers = glob.glob(os.path.join(racine, '**', 'watch-history.json'), recursive=True)
    
    if not fichiers:
        print("Erreur : Fichier watch-history.json introuvable dans le dossier.")
        return None
    
    chemin_final = fichiers[0]
    print(f"Fichier trouvé : {chemin_final}")
    
    with open(chemin_final, 'r', encoding='utf-8') as f:
        return json.load(f)
```
# 2. Le TOP 10 des chaînes que l'on regarde le plus. 
### On utilise le dossier avec l'historique de nos visionnages. 
```
data = trouver_et_charger_historique(dossier_racine)

if data:
    df = pd.DataFrame(data)
    
    # Extraction du nom de la chaîne
    # Dans le JSON YouTube, le nom de la chaîne est dans 'subtitles' -> 'name'
    def extraire_chaine(x):
        if isinstance(x, list) and len(x) > 0:
            return x[0].get('name')
        return None

    df['Chaine'] = df['subtitles'].apply(extraire_chaine)
```    
### Nettoyage : on enlève les lignes sans chaîne (ex: pubs ou vidéos supprimées)
```  
    df = df.dropna(subset=['Chaine'])

    #On définit ce que c'est le "top 10".
    top10 = df['Chaine'].value_counts().head(10)
```
### Graphique
```
    plt.figure(figsize=(12, 8))
    top10.sort_values().plot(kind='barh', color='#FF69B4', edgecolor='black')
    plt.title("Mon Top 10 des chaînes YouTube les plus regardées", fontsize=14)
    plt.xlabel("Nombre de vidéos visionnées")
    plt.ylabel("Nom de la chaîne")
    plt.tight_layout()
    plt.show()

    print("\n------- TOP 10 -------")
    print(top10)
```
# 3. Analyse des 5 vidéos les plus regardées. 

### Première tentative de filtrage
```
df_v1 = pd.DataFrame(data)

#On filtre par mots-clés simples

df_v1 = df_v1[df_v1['title'].str.contains('Regardé|Watched|Vous avez regardé', na=False, case=False)]

# On essaie de retirer "Ads" 
if 'details' in df_v1.columns:
    def est_une_pub(x):
        if isinstance(x, list) and len(x) > 0:
            return x[0].get('name') == 'Ads'
        return False
    df_v1 = df_v1[~df_v1['details'].apply(est_une_pub)]

top_replays_incorrect = df_v1['title'].value_counts().head(5)

print("Les 5 vidéos les plus regardées (Polluées par les PUBS)")
print(top_replays_incorrect)
```

### Nouveau filtrage plus robuste
```
df_v2 = pd.DataFrame(data)

# On ne garde que ce qui appartient officiellement à YouTube
df_v2 = df_v2[df_v2['header'] == 'YouTube']

# Une vraie vidéo a toujours une chaîne.
# Les annonces Google n'ont pas ce champ. On supprime donc les lignes vides.
df_v2 = df_v2.dropna(subset=['subtitles'])

# On retire explicitement "Des annonces Google"
if 'details' in df_v2.columns:
    def est_une_annonce(x):
        if isinstance(x, list) and len(x) > 0:
            return x[0].get('name') == 'Des annonces Google'
        return False
    df_v2 = df_v2[~df_v2['details'].apply(est_une_annonce)]

# Nettoyage universel du titre 
df_v2['Titre_Propre'] = df_v2['title'].str.replace(
    r'^(Vous avez regardé\s|Regardé\s|Watched\s)', '', regex=True, case=False
)

# Résultat
top_replays_correct = df_v2['Titre_Propre'].value_counts().head(5)

print("Les 5 vidéos les plus regardées")
print(top_replays_correct)
```
# 4. Part des vidéos regardées quand on est abonné et non abonné

### Chargement du fichier abonnements 
```
fichiers_abos = glob.glob(os.path.join(dossier_racine, '**', '*abonnements*.csv'), recursive=True)

if fichiers_abos:
    df_abos = pd.read_csv(fichiers_abos[0])
    liste_abos = df_abos['Titres des chaînes'].str.lower().unique().tolist()
```
### SÉCURITÉ : Recréation de la colonne 'Chaine' si elle manque.
``` 
    if 'Chaine' not in df.columns:
        df['Chaine'] = df['subtitles'].apply(lambda x: x[0].get('name') if isinstance(x, list) and len(x) > 0 else "Inconnu")
```
### Fonction de vérification : Abonné ou non ?
```
    def verifier_si_abonne(nom):
        if str(nom).lower() in liste_abos:
            return 'Vidéos de mes Abonnements'
        return 'Vidéos visionnées sans être abonné'
    
    df['Source_Visionnage'] = df['Chaine'].apply(verifier_si_abonne)
    stats_fidelite = df['Source_Visionnage'].value_counts()

### Graphique
     plt.figure(figsize=(8, 8))
    plt.pie(stats_fidelite, labels=stats_fidelite.index, autopct='%1.1f%%', 
            startangle=140, colors=['#FF69B4', '#FFFC00'], pctdistance=0.85,
            wedgeprops={'edgecolor': 'white', 'linewidth': 3})
    
    # Dessin du trou central
    plt.gcf().gca().add_artist(plt.Circle((0,0), 0.70, fc='white'))
    
    plt.title("Part des abonnements dans mon historique", fontsize=15)
    plt.show()
```
# 5. Nombres d'heures converti en nombre de jours de visionnage sur Youtube depuis le début
```
fichiers = glob.glob(os.path.join(dossier_racine, '**', 'watch-history.json'), recursive=True)
if fichiers:
    with open(fichiers[0], 'r', encoding='utf-8') as f:
        data = json.load(f)
    
    df = pd.DataFrame(data)
```    
### On filtre pour ne garder que les vidéos (on exclut les pubs ou recherches)
```
    df = df[df['title'].str.contains('regardé|Watched', na=False)]
    
    nb_videos = len(df)
```
### Calcul du temps (estimations)
### On estime qu'une vidéo dure en moyenne 15 minutes car on a pas la durée de chaque vidéo
```
    duree_moy_min = 15 
    
    total_minutes = nb_videos * duree_moy_min
    total_heures = total_minutes / 60
    total_jours = total_heures / 24

    print(f"--- NOMBRES D'HEURES DE VISIONNAGE ---")
    print(f"Nombre de vidéos vues : {nb_videos}")
    print(f"Temps total estimé : {total_heures:.1f} heures")
    print(f"Soit environ : {total_jours:.1f} jours complets (24h/24) !")
```
### Graphique pour voir l'évolution du temps de visionnage depuis le début. 
```
    df['Date'] = pd.to_datetime(df['time'], format='ISO8601')
# On groupe par mois
    conso_mensuelle = df.resample('M', on='Date').size()
# Conversion des mois en heures (nb vidéos * 10 / 60)
    heures_mensuelles = (conso_mensuelle * duree_moy_min) / 60

    plt.figure(figsize=(12, 6))
    heures_mensuelles.plot(kind='area', color='#FF69B4', alpha=0.3) 
    heures_mensuelles.plot(kind='line', color='#FF69B4', marker='o', linewidth=2)

    plt.title("Évolution de mon temps de visionnage (Heures / Mois)", fontsize=14)
    plt.xlabel("Mois de l'année")
    plt.ylabel("Nombre d'heures (estimé)")
    plt.grid(True, linestyle='--', alpha=0.5)
    plt.tight_layout()
    plt.show()
else:
    print("Fichier watch-history.json introuvable.")
```
    
# 6. Analyse de la saisonnalité 
```
# Préparation des données temporelles
if 'Date_Heure' in df.columns:
    df['Mois'] = df['Date_Heure'].dt.month_name()
    df['Jour_Eng'] = df['Date_Heure'].dt.day_name()
    
# Définition des ordres et traductions 
    ordre_jours = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
    ordre_mois = ['January', 'February', 'March', 'April', 'May', 'June', 
                  'July', 'August', 'September', 'October', 'November', 'December']
    traduction = {'Monday':'Lun', 'Tuesday':'Mar', 'Wednesday':'Mer', 'Thursday':'Jeu', 
                  'Friday':'Ven', 'Saturday':'Sam', 'Sunday':'Dim'}

# Création du tableau de données
    pivot_table = df.pivot_table(index='Jour_Eng', columns='Mois', aggfunc='size', fill_value=0)
    
# On réorganise le tableau selon nos listes d'ordre
    pivot_table = pivot_table.reindex(index=ordre_jours, columns=ordre_mois).fillna(0)

# Création de la Heatmap
    plt.figure(figsize=(14, 7))
    im = plt.imshow(pivot_table, cmap='RdPu', aspect='auto') 
    
    plt.colorbar(im, label="Nombre de vidéos")
    
# Configuration des axes
    plt.xticks(range(len(ordre_mois)), [m[:3] for m in ordre_mois]) 
    plt.yticks(range(len(ordre_jours)), [traduction[j] for j in ordre_jours])
    
# Ajout des chiffres à l'intérieur
    for i in range(len(ordre_jours)):
        for j in range(len(ordre_mois)):
            val = pivot_table.iloc[i, j]
            plt.text(j, i, int(val), ha="center", va="center", 
                     color="black" if val < pivot_table.values.max()/2 else "white")

    plt.title("Intensité de visionnage : Mois vs Jours", fontsize=14)
    plt.show()
```

# 7. Quels jours de la semaine on regarde le plus de vidéos Youtube ? 
```
# On s'assure que la colonne Date_Heure existe avec le bon format
df['Date_Heure'] = pd.to_datetime(df['time'], format='ISO8601')

# On extrait le jour en anglais
df['Jour_Eng'] = df['Date_Heure'].dt.day_name()

# On définit l'ordre exact pour que le graphique soit bien rangé
ordre_jours = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
# On traduit pour le graphique final
traduction = {'Monday':'Lun', 'Tuesday':'Mar', 'Wednesday':'Mer', 'Thursday':'Jeu', 
              'Friday':'Ven', 'Saturday':'Sam', 'Sunday':'Dim'}

stats_jours = df['Jour_Eng'].value_counts().reindex(ordre_jours)
stats_jours.index = stats_jours.index.map(traduction)
```
### Graphique
```
plt.figure(figsize=(10, 6))
stats_jours.plot(kind='bar', color='#FF7F50', edgecolor='black')
plt.title("Répartition de mon visionnage par jour", fontsize=14)
plt.ylabel("Nombre de vidéos")
plt.xticks(rotation=0)
plt.show()
```
# 8. À quelle heure on regarde le plus souvent Youtube ? 
```
# Il faut d'abord que python arrive à lire les horaires sur le fichier
if data is not None:
# L'astuce est ici : format='ISO8601' permet de lire les dates YouTube sans erreur
    df['Date_Heure'] = pd.to_datetime(df['time'], format='ISO8601')
    
# On extrait l'heure
    df['Heure'] = df['Date_Heure'].dt.hour

# On compte les vidéos par heure
    rythme_horaire = df['Heure'].value_counts().sort_index()
```
### Graphique
```
    plt.figure(figsize=(12, 6))
    rythme_horaire.plot(kind='bar', color='skyblue', edgecolor='black')
    plt.title("Mon pic d'activité YouTube sur 24h", fontsize=14)
    plt.xlabel("Heure de la journée (0h - 23h)")
    plt.ylabel("Nombre de vidéos lancées")
    plt.xticks(rotation=0)
    plt.grid(axis='y', linestyle='--', alpha=0.7)
    plt.tight_layout()
    plt.show()
```

# 9. Recap 2025. 

### SÉCURITÉ :  Recréation de la colonne 'Chaine' si elle manque.
```
df['Chaine'] = df['subtitles'].apply(lambda x: x[0].get('name') if isinstance(x, list) and len(x) > 0 else "Inconnu")
```

### On filtre uniquement pour l'année 2025
```
df_2025 = df[df['Date_Heure'].dt.year == 2025].copy()
```
### Nombre de vidéos regardées cette année
```
nb_videos_2025 = len(df_2025)
```
### Estimation du temps de visionnage(15 min par vidéo)
```
minutes_2025 = nb_videos_2025 * 15
heures_2025 = minutes_2025 / 60
jours_2025 = heures_2025 / 24
```
# 4. Affichage propre
```
print("="*40)
print(f" RECAP YOUTUBE DE 2025")
print("="*40)
print(f"Nombre de vidéos vues  : {nb_videos_2025}")
print(f"Temps en heures        : {heures_2025:.1f} h")
print(f"Temps en jours         : {jours_2025:.1f} jours")
print("="*40)
```
### On filtre pour enlever les 'Inconnu' de l'analyse 2025
```
df_2025_propre = df_2025[df_2025['Chaine'] != 'Inconnu']
```
## On calcule le Top 1
```
if not df_2025_propre.empty:
    top_1_2025 = df_2025_propre['Chaine'].value_counts().idxmax()
    nb_vues_top1 = df_2025_propre['Chaine'].value_counts().max()
    
    print("="*50)
    print(f"La chaîne la plus regardée en  2025 est : {top_1_2025}")
    print(f"Tu l'as regardée : {nb_vues_top1} fois")
    print("="*50)
else:
    print("Aucun nom de chaîne trouvé pour 2025.")
```    
## Moyenne par jour en 2025
```
jours_distincts_2025 = df_2025['Date_Heure'].dt.date.nunique()
moyenne_par_jour = nb_videos_2025 / jours_distincts_2025

print(f"Tu as regardé YouTube {jours_distincts_2025} jours sur 365.")
print(f"En moyenne, tu lances {moyenne_par_jour:.1f} vidéos par jour.")
```
## Le jour le plus chargé de 2025
```
record_jour = df_2025['Date_Heure'].dt.date.value_counts().head(1)
date_record = record_jour.index[0]
valeur_record = record_jour.values[0]

print(f"Ton record en 2025 : Tu as regardé {valeur_record} vidéos le {date_record} !")
```
