# Rapport de Projet — CinéCatalogue (Netplixe)
## Conception et déploiement d'une application Web orientée données et API

---

**Module :** LU3IN403 — Opérations et Systèmes Cloud  
**Cursus :** Licence 3 Informatique — Sorbonne Université  
**Année universitaire :** 2024–2025  
**Date de rendu :** 30 Avril 2026

---

## Table des matières

1. [Introduction et contexte](#1-introduction-et-contexte)
2. [Architecture et choix techniques](#2-architecture-et-choix-techniques)
3. [Source des données](#3-source-des-données)
4. [Back-end : FastAPI et MongoDB](#4-back-end--fastapi-et-mongodb)
5. [Front-end : HTML / CSS / JavaScript](#5-front-end--html--css--javascript)
6. [Communication Front-end / Back-end](#6-communication-front-end--back-end)
7. [Conteneurisation avec Docker](#7-conteneurisation-avec-docker)
8. [Orchestration avec Kubernetes](#8-orchestration-avec-kubernetes)
9. [Pipeline CI/CD avec GitHub Actions](#9-pipeline-cicd-avec-github-actions)
10. [Fonctionnalités avancées](#10-fonctionnalités-avancées)
11. [Difficultés rencontrées et solutions](#11-difficultés-rencontrées-et-solutions)
12. [Conclusion](#12-conclusion)

---

## 1. Introduction et contexte

### 1.1 Présentation générale

Ce projet s'inscrit dans le cadre du module **LU3IN403 — Opérations et Systèmes Cloud** de la Licence 3 Informatique à Sorbonne Université. L'objectif est de concevoir, implémenter et déployer une application web complète, proche des pratiques professionnelles actuelles.

L'application réalisée est **Netplixe**, un catalogue de films inspiré de l'interface de Netflix. Elle permet aux utilisateurs de parcourir un catalogue de plus de 25 000 films, de les rechercher, de les filtrer par genre ou par note, de consulter une fiche détaillée pour chaque film, de sauvegarder leurs favoris et d'obtenir des recommandations de films similaires.

### 1.2 Objectifs pédagogiques

Le projet vise à démontrer la maîtrise des compétences suivantes :

- Conception d'une **architecture web moderne** découplée en couches (front-end / back-end / base de données)
- Développement d'une **API RESTful** avec FastAPI (Python)
- Utilisation d'une **base de données NoSQL** (MongoDB)
- Implémentation de fonctionnalités avancées : cache Redis, moteur de recherche Elasticsearch, pagination
- **Conteneurisation** de l'ensemble de l'application avec Docker
- **Orchestration** des conteneurs avec Kubernetes (Minikube)
- Mise en place d'un **pipeline CI/CD** automatisé avec GitHub Actions
- Justification des choix techniques dans un rapport structuré

### 1.3 Périmètre fonctionnel

L'application couvre les fonctionnalités suivantes :

**Fonctionnalités de base :**
- Affichage d'un catalogue dynamique de films organisé en rangées horizontales (Netflix-style)
- Recherche par titre et description (insensible à la casse)
- Filtrage par type (Film), par genre (Action, Drame, Comédie, Thriller)
- Fiche détaillée de chaque film avec affiche, description, note et métadonnées

**Fonctionnalités avancées :**
- **Favoris** : sauvegarde des films préférés avec persistance localStorage
- **Films similaires** : recommandations basées sur les genres partagés
- **Cache Redis** : mise en cache de toutes les réponses API (TTL configurable)
- **Recherche avancée Elasticsearch** : recherche multi-critères avec tolérance aux fautes
- **Pagination** : navigation par pages avec compteurs et boutons dynamiques

---

## 2. Architecture et choix techniques

### 2.1 Étude des architectures

Avant d'écrire la moindre ligne de code, nous avons analysé les grandes familles d'architectures logicielles pour choisir celle qui convenait le mieux à notre projet.

#### Architecture Monolithique

Dans un monolithe, l'ensemble de l'application (interface, logique métier, accès aux données) est développé et déployé comme une seule unité. Si cette approche est simple au départ, elle devient rapidement un obstacle : scalabilité difficile, déploiements risqués, et code difficilement maintenable à long terme.

#### Architecture Client-Serveur

Ce modèle sépare le client (navigateur) du serveur (API). Le client initie des requêtes HTTP, le serveur répond avec des données JSON. Cette séparation rend chaque composant indépendant : on peut modifier le front-end sans toucher au back-end, et vice-versa.

#### Architecture Microservices

L'application est découpée en services autonomes (un service de recherche, un service de recommandation, etc.), chacun avec sa propre base de données. Adoptée par des géants comme Netflix et Amazon, elle offre une scalabilité granulaire. Cependant, sa complexité opérationnelle est disproportionnée pour un projet de cette taille.

#### Notre choix : Architecture Client-Serveur en couches

Nous avons retenu l'architecture **client-serveur en couches** pour les raisons suivantes :

- Elle est **naturellement adaptée à Docker** : chaque composant (front-end, back-end, base de données) devient un conteneur isolé
- Elle facilite le **déploiement Kubernetes** via des services distincts
- Elle respecte le principe de **séparation des responsabilités** : le front-end gère l'affichage, le back-end gère la logique, la base de données gère la persistance
- Elle reste **maîtrisable** pour une équipe de deux développeurs

```
┌─────────────────────────────────────────────────────────┐
│                      Navigateur                          │
│              HTML / CSS / JavaScript                     │
│         (fetch API → requêtes HTTP JSON)                 │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTP REST
┌──────────────────────────▼──────────────────────────────┐
│                  Back-end FastAPI                        │
│        Logique métier · Validation · Sérialisation       │
└────────┬─────────────────┬─────────────────┬────────────┘
         │                 │                 │
   ┌─────▼─────┐    ┌──────▼──────┐  ┌──────▼──────┐
   │  MongoDB  │    │    Redis    │  │Elasticsearch│
   │  (données)│    │   (cache)   │  │ (recherche) │
   └───────────┘    └─────────────┘  └─────────────┘
```

### 2.2 Design patterns appliqués

#### Pattern MVC (Model-View-Controller)

Nous appliquons ce pattern de manière naturelle :
- **Model** : la collection MongoDB `shows` et les fonctions de mapping (`clean_show`, `import_csv`)
- **View** : les fichiers `index.html`, `style.css` et les fonctions de rendu JS (`buildCard`, `openModal`)
- **Controller** : les routes FastAPI (`get_shows`, `search_shows`, `get_similar`, etc.)

#### Pattern Repository

Le fichier `database.py` joue le rôle de couche d'abstraction entre la logique métier et le stockage. La variable `shows_collection` encapsule l'accès à MongoDB. Si nous décidions de migrer vers PostgreSQL, seul `database.py` serait à modifier.

#### Pattern Singleton

La connexion MongoDB (`MongoClient`) est instanciée une seule fois dans `database.py` et importée dans `main.py`. Python garantit qu'un module importé n'est évalué qu'une seule fois, implémentant ainsi naturellement le pattern Singleton.

### 2.3 Stack technologique choisie

| Composant | Technologie | Justification |
|-----------|-------------|---------------|
| Front-end | HTML / CSS / JS Vanilla | Pas de dépendance de build, performances maximales, apprentissage des fondamentaux |
| Back-end | Python 3.11 + FastAPI | Productivité élevée, documentation Swagger automatique, typage natif |
| Base de données principale | MongoDB 7 | Flexibilité du schéma JSON, adapté aux données de films hétérogènes |
| Cache | Redis 7 | Performances extrêmes (µs), TTL natif, très utilisé en production |
| Recherche avancée | Elasticsearch 8.12 | Moteur de recherche full-text avec scores de pertinence et tolérance aux fautes |
| Conteneurisation | Docker + Docker Compose | Standard de l'industrie, isolement parfait des environnements |
| Orchestration | Kubernetes (Minikube) | Gestion du cycle de vie des conteneurs, scaling, résilience |
| CI/CD | GitHub Actions | Intégration native GitHub, gratuit pour les projets publics |

---

## 3. Source des données

### 3.1 Choix du dataset

Nous avons choisi d'utiliser un **dataset Kaggle** combinant les données TMDB (The Movie Database) et IMDb plutôt qu'une API externe. Ce dataset contient **25 000 films** avec les champs suivants : `title`, `overview` (description), `genres`, `vote_average` (note sur 10), `release_date`, `poster_path` (chemin vers l'affiche), `vote_count`.

**Justification de ce choix par rapport à l'API TMDB directe :**

| Critère | Dataset Kaggle | API TMDB en direct |
|---------|---------------|-------------------|
| Volume de données | 25 000 films disponibles immédiatement | Limité par les quotas (40 req/10s) |
| Indépendance | Aucune clé API requise | Clé exposable, quota épuisable |
| Performance | Chargement unique en base | Latence réseau à chaque requête |
| Disponibilité | 100% offline | Dépend de l'infrastructure TMDB |

Lors du TME 3, nous avons étudié en détail l'API TMDB — son système d'authentification par Bearer Token, ses endpoints (`/movie/popular`, `/search/movie`, `/trending/movie/day`), la gestion des erreurs HTTP (401, 429, 503) et les bonnes pratiques de rate limiting avec Exponential Backoff. Pour le projet final, le dataset nous offre la même richesse de données sans les contraintes d'une API externe.

### 3.2 Import et normalisation des données

Le script `backend/import_data.py` réalise le pipeline d'import :

```python
def import_csv(path: str = "data/tmdb_imdb.csv"):
    shows_collection.drop()           # Vide la collection avant réimport
    df = pd.read_csv(path)            # Lecture avec pandas
    records = df.to_dict(orient="records")  # Conversion en liste de dicts

    for r in records:
        # 1. Nettoyage des NaN (pandas → None pour MongoDB)
        for k, v in r.items():
            if isinstance(v, float) and math.isnan(v):
                r[k] = None

        # 2. Normalisation vers notre format standard
        r["type"]        = "Movie"
        r["description"] = r.get("overview")
        r["listed_in"]   = r.get("genres")
        r["rating"]      = r.get("vote_average")

        # 3. Construction de l'URL d'affiche TMDB
        poster = r.get("poster_path")
        if poster and str(poster).startswith("/"):
            r["poster_url"] = f"https://image.tmdb.org/t/p/w500{poster}"

    shows_collection.insert_many(records)

    # 4. Création d'index pour accélérer les requêtes
    shows_collection.create_index("title")
    shows_collection.create_index("type")
    shows_collection.create_index("listed_in")
    shows_collection.create_index("rating")
```

**Points clés de cette implémentation :**

- **Gestion des NaN** : pandas représente les cellules vides par `float('nan')`. MongoDB n'accepte pas NaN mais accepte `None`. La conversion est faite explicitement pour chaque champ.
- **Normalisation** : TMDB nomme le synopsis `overview` ; notre API l'expose sous `description` pour cohérence. Idem pour `genres` → `listed_in` et `vote_average` → `rating`.
- **Index MongoDB** : Créés sur les champs les plus filtrés (`title`, `type`, `listed_in`, `rating`). Un index fonctionne comme l'index d'un livre — sans lui, MongoDB scanne tous les documents séquentiellement ; avec lui, il va directement aux documents correspondants.
- **`insert_many`** : Une seule opération d'insertion en masse est bien plus rapide que 25 000 appels `insert_one`.

### 3.3 Import automatique au démarrage

L'import est déclenché automatiquement quand la base est vide, via l'event `startup` de FastAPI :

```python
@app.on_event("startup")
def startup_event():
    if shows_collection.count_documents({}) == 0:
        import_csv()
    _es_ensure_index()
    _es_index_all()
```

---

## 4. Back-end : FastAPI et MongoDB

### 4.1 Connexion aux services — `database.py`

Ce fichier est le point d'entrée de toutes les connexions aux services externes. Il implémente une **stratégie de dégradation gracieuse** : si Redis ou Elasticsearch ne sont pas disponibles, l'application continue de fonctionner normalement avec MongoDB seul.

```python
from pymongo import MongoClient
import os

# ─── MongoDB (obligatoire) ─────────────────────────────
MONGODB_URI = os.getenv("MONGODB_URI", "mongodb://localhost:27017")
client           = MongoClient(MONGODB_URI)
db               = client["netflix"]
shows_collection = db["shows"]

# ─── Redis (optionnel) ────────────────────────────────
redis_client = None
try:
    import redis as redis_lib
    REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
    _r = redis_lib.from_url(REDIS_URL, decode_responses=True,
                             socket_connect_timeout=2)
    _r.ping()
    redis_client = _r
    print("[Redis] Connecté ✓")
except Exception as e:
    print(f"[Redis] Non disponible : {e}")

# ─── Elasticsearch (optionnel) ────────────────────────
es_client = None
try:
    from elasticsearch import Elasticsearch
    ES_URL = os.getenv("ES_URL", "http://localhost:9200")
    _es = Elasticsearch(ES_URL, request_timeout=5)
    if _es.ping():
        es_client = _es
        print("[Elasticsearch] Connecté ✓")
except Exception as e:
    print(f"[Elasticsearch] Non disponible : {e}")
```

**Points techniques importants :**

- `os.getenv("MONGODB_URI", "mongodb://localhost:27017")` : lit la variable d'environnement injectée par Docker ou Kubernetes. La valeur par défaut `localhost:27017` permet le développement local sans Docker.
- `decode_responses=True` : Redis stocke nativement des bytes. Cette option fait la conversion automatique en strings Python.
- `socket_connect_timeout=2` : on n'attend pas plus de 2 secondes pour la connexion Redis. Si le service ne répond pas, on passe à autre chose immédiatement.

### 4.2 API principale — `main.py`

#### Configuration de l'application FastAPI

```python
app = FastAPI(
    title="Netflix Catalogue API",
    description="API REST pour le catalogue Netflix — Projet LU3IN403",
    version="2.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**CORS (Cross-Origin Resource Sharing)** : une sécurité du navigateur bloque les requêtes JavaScript vers un domaine ou un port différent. Sans cette configuration, le front-end sur `localhost:8080` ne pourrait pas appeler le back-end sur `localhost:8000`. `allow_origins=["*"]` autorise toutes les origines — suffisant pour un projet académique ; en production, on restreindrait au domaine de l'application.

#### Le système de cache Redis

```python
def make_key(*args) -> str:
    raw = ":".join(str(a) for a in args)
    return "netplixe:" + hashlib.md5(raw.encode()).hexdigest()

def cache_get(key: str):
    if not redis_client:
        return None
    val = redis_client.get(key)
    return json.loads(val) if val else None

def cache_set(key: str, data, ttl: int = 300):
    if not redis_client:
        return
    redis_client.setex(key, ttl, json.dumps(data, default=str))
```

`make_key` génère une clé unique pour chaque combinaison de paramètres en calculant un hash MD5. Par exemple, la requête `GET /shows?limit=20&sort_by=rating` génère toujours la même clé hexadécimale. `setex` (SET with EXpiration) stocke la donnée avec une durée de vie automatique : après `ttl` secondes, Redis supprime la clé et la prochaine requête régénèrera la réponse.

#### Endpoint `/health` — Sonde de disponibilité

```python
@app.get("/health")
def health():
    return {
        "status": "ok",
        "total_shows": shows_collection.count_documents({}),
        "redis":         {"available": redis_ok},
        "elasticsearch": {"available": es_ok},
    }
```

Cet endpoint est utilisé par la **readinessProbe Kubernetes** — K8s l'appelle toutes les 10 secondes pour savoir si le pod est prêt à recevoir du trafic. Si la réponse n'est pas HTTP 200, K8s redirige le trafic vers d'autres pods.

#### Endpoint `/shows` — Catalogue paginé avec filtres

```python
@app.get("/shows")
def get_shows(
    limit:   int           = Query(default=20, ge=1, le=200),
    skip:    int           = Query(default=0,  ge=0),
    page:    Optional[int] = Query(default=None, ge=1),
    type:    Optional[str] = Query(default=None),
    genre:   Optional[str] = Query(default=None),
    sort_by: Optional[str] = Query(default=None),
):
```

`Query(ge=1, le=200)` : FastAPI valide automatiquement les paramètres. Si `limit=-5` ou `limit=999` est reçu, FastAPI répond HTTP 422 (Unprocessable Entity) sans que nous ayons à écrire de code de validation manuellement.

```python
if page is not None:
    skip = (page - 1) * limit   # page=3, limit=20 → skip=40

query = {}
if type:
    query["type"] = type
if genre:
    query["listed_in"] = {"$regex": re.escape(genre), "$options": "i"}
```

`re.escape(genre)` protège contre les injections : si un utilisateur envoie `Action.*` comme genre, `re.escape` le transforme en `Action\.\*` — un littéral, pas une regex malveillante.

La réponse inclut les **métadonnées de pagination** :

```python
result = {
    "items":  items,
    "total":  total,
    "page":   (skip // limit) + 1,
    "pages":  math.ceil(total / limit),
    "from_cache": False,
}
cache_set(cache_key, result)   # Mise en cache 5 minutes
```

Le front-end utilise `total`, `page` et `pages` pour construire les boutons de navigation.

#### Endpoint `/shows/search` — Recherche simple

```python
@app.get("/shows/search")
def search_shows(q: str = Query(..., min_length=1), ...):
    regex = {"$regex": re.escape(q), "$options": "i"}
    query = {"$or": [{"title": regex}, {"description": regex}, {"listed_in": regex}]}
    cursor = shows_collection.find(query).sort("rating", -1).skip(skip).limit(limit)
```

`$or` combine plusieurs conditions : le film correspond si le terme de recherche apparaît dans le titre, la description **ou** les genres. `.sort("rating", -1)` trie les résultats par note décroissante — les films les mieux notés apparaissent en premier.

#### Endpoint `/shows/favorites` — Favoris

```python
class FavoritesRequest(BaseModel):
    ids: List[str]

@app.post("/shows/favorites")
def get_favorites(body: FavoritesRequest):
    valid_ids = [ObjectId(id_) for id_ in body.ids if ObjectId.is_valid(id_)]
    docs = shows_collection.find({"_id": {"$in": valid_ids}})
    return [clean_show(doc) for doc in docs]
```

Les favoris sont stockés côté client dans le `localStorage` du navigateur (uniquement leurs IDs). Quand l'utilisateur clique sur "Mes Favoris", le front-end envoie la liste des IDs au back-end via une requête POST, et le back-end retourne les documents complets. Ce design présente plusieurs avantages : les données des films restent toujours à jour (on lit en base à chaque fois), et le back-end n'a pas besoin de gérer des sessions utilisateurs.

`ObjectId.is_valid(id_)` protège contre les IDs malformés qui provoqueraient une erreur MongoDB.

#### Endpoint `/shows/{id}/similar` — Films similaires

```python
@app.get("/shows/{show_id}/similar")
def get_similar(show_id: str, limit: int = Query(default=8, ge=1, le=20)):
    doc = shows_collection.find_one({"_id": ObjectId(show_id)})
    genres = [g.strip() for g in (doc.get("listed_in") or "").split(",") if g.strip()]
    
    genre_regex = "|".join(re.escape(g) for g in genres[:3])
    fetch_limit = min(limit * 50, 500)
    
    cursor = shows_collection.find({
        "_id":       {"$ne": ObjectId(show_id)},
        "listed_in": {"$regex": genre_regex, "$options": "i"},
        "rating":    {"$ne": None, "$gte": 6.0},
    }).sort("rating", -1).limit(fetch_limit)

    seen:  set  = set()
    items: list = []
    for d in cursor:
        key = (str(d.get("title","")) + str(d.get("release_year",""))).lower().strip()
        if key and key not in seen:
            seen.add(key)
            items.append(clean_show(d))
            if len(items) >= limit:
                break
```

**Algorithme de recommandation :**
1. Extraire les genres du film demandé (ex : `["Action", "Crime", "Drama"]`)
2. Construire une regex MongoDB : `Action|Crime|Drama`
3. Chercher les films ayant au moins un genre en commun, avec une note ≥ 6.0
4. Récupérer jusqu'à 500 candidats (50× la limite souhaitée) pour compenser les doublons du dataset
5. Dédupliquer par clé `titre+année` en Python
6. Retourner les `limit` premiers films uniques

Le multiplicateur ×50 est nécessaire car le dataset contient des doublons. Sans lui, nous risquerions de renvoyer plusieurs fois le même film.

### 4.3 Elasticsearch — Recherche avancée

#### Création et remplissage de l'index

```python
def _es_ensure_index():
    if not es_client.indices.exists(index="shows"):
        es_client.indices.create(
            index="shows",
            mappings={
                "properties": {
                    "title":       {"type": "text", "analyzer": "standard"},
                    "description": {"type": "text", "analyzer": "standard"},
                    "listed_in":   {"type": "text"},
                    "rating":      {"type": "float"},
                    "release_year": {"type": "keyword"},
                }
            }
        )

def _es_index_all():
    from elasticsearch.helpers import bulk
    def _generate():
        for doc in shows_collection.find({}):
            mid = str(doc["_id"])
            yield {
                "_index": "shows",
                "_id": mid,
                "_source": { "mongo_id": mid, "title": doc.get("title"), ... }
            }
    bulk(es_client, _generate(), chunk_size=500)
```

`bulk` indexe les documents par lots de 500, ce qui est bien plus efficace que 25 000 insertions individuelles.

#### Requête de recherche avancée

```python
def _search_es(q, genre, min_year, max_year, min_rating, type_, limit, skip):
    must = []
    if q:
        must.append({
            "multi_match": {
                "query":     q,
                "fields":    ["title^3", "description", "listed_in^2"],
                "fuzziness": "AUTO",
                "type":      "best_fields",
            }
        })
    
    filters = []
    if genre:
        filters.append({"match": {"listed_in": genre}})
    if min_rating is not None:
        filters.append({"range": {"rating": {"gte": min_rating}}})
    
    es_query = {"bool": {"must": must or [{"match_all": {}}], "filter": filters}}
    resp = es_client.search(index="shows", query=es_query, from_=skip, size=limit)
```

**Points clés de la requête Elasticsearch :**
- `title^3` : le titre a 3× plus de poids que la description. Un film dont le titre correspond exactement sera classé plus haut qu'un film qui ne mentionne le terme que dans sa description.
- `fuzziness: "AUTO"` : tolérance automatique aux fautes de frappe. "Avenjers" trouve "Avengers". Elasticsearch calcule la distance d'édition de Levenshtein pour décider si deux mots sont "proches".
- `bool/must` vs `bool/filter` : `must` affecte le score de pertinence (ordre des résultats), `filter` s'applique comme un filtre binaire sans modifier le score.

Après la recherche Elasticsearch, nous récupérons les données complètes depuis MongoDB :

```python
mongo_ids = [hit["_id"] for hit in hits]
docs_map  = {str(d["_id"]): d for d in shows_collection.find(
    {"_id": {"$in": [ObjectId(mid) for mid in mongo_ids]}}
)}
items = [clean_show(docs_map[mid]) for mid in mongo_ids if mid in docs_map]
```

ES stocke les champs indexés pour la recherche (titre, genres). MongoDB stocke les données complètes (affiche, description longue). ES donne les IDs dans le bon ordre de pertinence ; MongoDB donne les documents complets. Les deux s'utilisent de concert.

---

## 5. Front-end : HTML / CSS / JavaScript

### 5.1 Structure HTML — `index.html`

#### La barre de navigation

```html
<header class="navbar" id="navbar">
  <div class="navbar-left">
    <div class="logo">NETPLIXE</div>
    <nav class="nav-links">
      <a href="#" class="nav-link" data-type="Movie">Films</a>
      <a href="#" class="nav-link" data-genre="Action">Action</a>
      <a href="#" class="nav-link" id="navFavorites" data-section="favorites">
        Mes Favoris
        <span id="favBadge" class="fav-nav-badge hidden">0</span>
      </a>
    </nav>
  </div>
  <div class="navbar-right">
    <button id="advToggleBtn"><!-- Icône filtres SVG --></button>
    <input id="searchInput" class="search-input" placeholder="Titres, genres..." />
  </div>
</header>
```

Les **data attributes HTML5** (`data-type`, `data-genre`, `data-section`) permettent de transmettre des informations du HTML vers le JavaScript sans JavaScript dans le HTML. Le JS les lit via `link.dataset.type`, `link.dataset.genre`, etc. Le `favBadge` affiche un compteur rouge des favoris — il est masqué par défaut (`hidden`) et n'apparaît que lorsqu'au moins un favori existe.

#### Le panneau de recherche avancée

```html
<div id="advSearchPanel" class="adv-search-panel hidden">
  <div class="adv-search-inner">
    <div class="adv-field">
      <label for="advQ">Recherche</label>
      <input id="advQ" type="text" placeholder="Titre, description..." />
    </div>
    <div class="adv-field">
      <label for="advGenre">Genre</label>
      <select id="advGenre">
        <option value="Action">Action</option>
        ...
      </select>
    </div>
    <div class="adv-field">
      <label for="advMinYear">Année min</label>
      <input id="advMinYear" type="number" min="1900" max="2024" />
    </div>
    <div class="adv-field">
      <label for="advMinRating">Note min (0–10)</label>
      <input id="advMinRating" type="number" min="0" max="10" step="0.5" />
    </div>
  </div>
</div>
```

Ce panneau permet de combiner jusqu'à 5 critères simultanément : terme de recherche, genre, année minimale, année maximale, note minimale. Il est caché par défaut et s'anime vers le bas quand on clique sur l'icône de filtres.

#### La section Hero

```html
<section class="hero" id="hero">
  <div class="hero-backdrop" id="heroBackdrop"></div>
  <div class="hero-overlay"></div>
  <div class="hero-content">
    <h1 class="hero-title" id="heroTitle">Chargement...</h1>
    <div class="hero-meta" id="heroMeta"></div>
    <p class="hero-desc" id="heroDesc"></p>
    <div class="hero-buttons">
      <button class="btn-play" id="heroPlayBtn">▶ Lecture</button>
      <button class="btn-info" id="heroInfoBtn">ℹ Plus d'infos</button>
    </div>
  </div>
</section>
```

Le `hero-backdrop` est une `div` dont l'arrière-plan CSS est l'affiche du film en haute résolution. Le `hero-overlay` ajoute un dégradé CSS superposé — de gauche (opaque) à droite (transparent) et de bas (opaque) à haut (transparent) — reproduisant exactement l'effet visuel de Netflix.

#### Les rangées de films

```html
<section class="row-section">
  <h2 class="row-title">Populaire sur Netplixe</h2>
  <div class="row-wrapper">
    <button class="row-arrow arrow-left" data-row="rowPopular">‹</button>
    <div id="rowPopular" class="cards-row"></div>
    <button class="row-arrow arrow-right" data-row="rowPopular">›</button>
  </div>
</section>
```

Le `cards-row` est un conteneur `flex` avec `overflow-x: auto` : les cartes débordent horizontalement et l'utilisateur peut les faire défiler. Les flèches de navigation utilisent `data-row="rowPopular"` pour identifier quel conteneur faire défiler.

#### Le Modal

```html
<div id="modal" class="modal hidden">
  <div class="modal-overlay" id="modalBg"></div>
  <div class="modal-container">
    <button id="modalClose">✕</button>
    <div id="modalBody"></div>
    <div id="modalSimilar" class="modal-similar hidden">
      <h3>Films similaires</h3>
      <div id="modalSimilarRow" class="modal-similar-row"></div>
    </div>
  </div>
</div>
```

Le modal est constitué de deux couches : `modal-overlay` (fond sombre semi-transparent, cliquable pour fermer) et `modal-container` (la boîte de contenu animée). `modalBody` et `modalSimilarRow` sont vides au départ et remplis dynamiquement à chaque ouverture.

### 5.2 Design CSS — `style.css`

#### Variables CSS pour la cohérence du thème

```css
:root {
  --red:     #E50914;  /* Rouge Netflix */
  --dark:    #141414;  /* Fond principal */
  --card-bg: #181818;  /* Fond des cartes */
  --green:   #46d369;  /* Notes et nouveautés */
  --muted:   #999;     /* Texte secondaire */
}
```

Ces variables CSS permettent de modifier l'ensemble du thème en un seul endroit. `var(--red)` est utilisé sur tous les éléments actifs : boutons, badges, bordures au survol.

#### Navbar avec transition au scroll

```css
.navbar {
  position: fixed;
  background: linear-gradient(to bottom, rgba(0,0,0,0.85) 0%, transparent 100%);
  transition: background 0.4s ease;
}
.navbar.scrolled {
  background: rgba(20, 20, 20, 0.97);
  box-shadow: 0 2px 10px rgba(0,0,0,0.5);
}
```

La navbar est transparente sur le Hero (pour voir l'image en dessous). Lorsque l'utilisateur fait défiler la page, JavaScript ajoute la classe `.scrolled` et le fond passe à opaque avec une transition fluide de 0.4s — comportement identique à Netflix.

#### Cartes avec effet hover Netflix

```css
.card {
  transition: transform 0.3s ease, z-index 0s 0.3s;
}
.card:hover {
  transform: scale(1.1);
  z-index: 20;
  transition: transform 0.3s ease, z-index 0s;
}
.card:hover .card-hover-info { opacity: 1; }
```

Au survol, la carte grossit de 10% et passe au-dessus des autres (`z-index: 20`). La transition `z-index 0s 0.3s` signifie : au survol, le `z-index` change immédiatement (sans délai) pour que la carte soit visible. Quand on retire le survol, le `z-index` revient après 0.3s (le temps que l'animation de rétrécissement finisse avant de passer derrière).

#### Système de pagination

```css
.pagination-btn.active {
  background: var(--red);
  border-color: var(--red);
  color: #fff;
  font-weight: 700;
}
.pagination-btn:disabled {
  opacity: 0.35;
  cursor: default;
}
```

Le bouton de la page courante est mis en rouge. Les boutons désactivés (flèche "précédent" sur la page 1) sont visuellement atténués.

#### Responsive Design

```css
@media (max-width: 900px) {
  .nav-links { display: none; }
  .hero-title { font-size: 40px; }
  .row-section { padding: 0 28px; }
}
@media (max-width: 600px) {
  .hero { height: 70vh; }
  .hero-title { font-size: 28px; }
  .card { width: 130px; }
}
```

L'interface s'adapte aux tablettes (nav cachée, padding réduit) et aux mobiles (hero plus petit, cartes plus étroites).

### 5.3 Logique JavaScript — `script.js`

#### Appels API typés

```javascript
const API_BASE = "http://netflix.local/api";

async function fetchShows(params = {}) {
  const qs = new URLSearchParams(
    Object.fromEntries(
      Object.entries(params)
        .filter(([, v]) => v !== undefined && v !== null && v !== "")
    )
  );
  const res = await fetch(`${API_BASE}/shows?${qs}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

`URLSearchParams` construit automatiquement la query string. Le `.filter(...)` enlève les paramètres undefined ou vides — on n'envoie pas `?genre=undefined` dans l'URL.

#### Système de favoris (localStorage)

```javascript
const FAV_KEY = "netplixe_favorites";

function getFavoriteIds() {
  return JSON.parse(localStorage.getItem(FAV_KEY) || "[]");
}

function toggleFavorite(show) {
  const ids = getFavoriteIds();
  const idx = ids.indexOf(show.id);
  if (idx === -1) { ids.push(show.id); }
  else             { ids.splice(idx, 1); }
  localStorage.setItem(FAV_KEY, JSON.stringify([...new Set(ids)]));
  updateFavBadge();
  return idx === -1;   // true = ajouté, false = retiré
}
```

`localStorage` est un stockage persistant dans le navigateur — les données survivent à la fermeture de l'onglet. `new Set(ids)` élimine automatiquement les doublons éventuels. `splice(idx, 1)` retire 1 élément à l'index `idx`.

#### Construction des cartes (`buildCard`)

```javascript
function buildCard(show) {
  const fav = isFavorite(show.id);
  card.innerHTML = `
    <div class="card-inner">
      <button class="card-fav-btn${fav ? " active" : ""}">
        <svg fill="${fav ? "#E50914" : "none"}" ...>❤</svg>
      </button>
      ${show.poster_url
        ? `<img src="${esc(show.poster_url)}" loading="lazy">`
        : `<div class="card-placeholder">🎬</div>`}
    </div>`;

  const favBtn = card.querySelector(".card-fav-btn");
  favBtn.addEventListener("click", e => {
    e.stopPropagation();   // Empêche l'ouverture du modal
    const added = toggleFavorite(show);
    favBtn.querySelector("svg").setAttribute("fill", added ? "#E50914" : "none");
  });
}
```

`loading="lazy"` : le navigateur ne charge l'image que lorsqu'elle entre dans le viewport. Avec plusieurs centaines de cartes, cela évite de télécharger des dizaines de mégaoctets d'images invisibles.

`e.stopPropagation()` est essentiel : sans lui, cliquer sur le bouton favori déclencherait aussi l'événement `click` de la carte parente (qui ouvre le modal).

La fonction `esc()` protège contre les attaques XSS en encodant les caractères spéciaux HTML :

```javascript
function esc(str) {
  return String(str)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");
}
```

Si un titre de film contient `<script>alert("hack")</script>`, `esc()` le transforme en `&lt;script&gt;...&lt;/script&gt;` — affiché comme texte, jamais exécuté.

#### Déduplication côté client

```javascript
function dedup(shows) {
  const seen = new Set();
  return shows.filter(s => {
    const key = (String(s.title || "") + String(s.release_year || ""))
                .toLowerCase().trim();
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
}
```

Le dataset contenant des doublons, cette fonction filtre les films identiques (même titre + même année) avant affichage. `Set` ne peut contenir qu'une seule occurrence de chaque valeur — idéal pour cette opération de déduplication.

#### Chargement parallèle des rangées

```javascript
async function initRows() {
  await Promise.all([
    loadRow("rowPopular",   { limit: 100, skip: 0,   sort_by: "rating" }),
    loadRow("rowTrending",  { limit: 100, skip: 100, sort_by: "rating" }),
    loadRow("rowMustWatch", { limit: 100, skip: 200, sort_by: "rating" }),
    loadRow("rowAction",    { limit: 100, skip: 0,   genre: "Action" }),
    loadRow("rowDrama",     { limit: 100, skip: 0,   genre: "Drama" }),
    loadRow("rowComedy",    { limit: 100, skip: 0,   genre: "Comedy" }),
    loadRow("rowTop10",     { limit: 100, skip: 0,   sort_by: "rating" }, true, 10),
  ]);
}
```

`Promise.all` lance toutes les requêtes **en parallèle**. Au lieu de charger chaque rangée séquentiellement (7 allers-retours réseau en série), toutes les requêtes partent simultanément et la page se remplit d'un coup. Les offsets (`skip: 0`, `skip: 100`, `skip: 200`) garantissent que les trois premières rangées montrent des films différents.

#### Pagination récursive

```javascript
function renderPagination(container, currentPage, totalPages, onPageChange) {
  const addBtn = (label, page, disabled = false, active = false) => {
    const btn = document.createElement("button");
    btn.className = "pagination-btn" + (active ? " active" : "");
    btn.disabled = disabled;
    if (!disabled) btn.addEventListener("click", () => onPageChange(page));
    container.appendChild(btn);
  };

  addBtn("←", currentPage - 1, currentPage <= 1);
  buildPageRange(currentPage, totalPages).forEach(p =>
    addBtn(String(p), p, false, p === currentPage)
  );
  addBtn("→", currentPage + 1, currentPage >= totalPages);
}

async function loadNavFilter(baseParams, title, page) {
  const skip = (page - 1) * (baseParams.limit || 36);
  const data = await fetchShows({ ...baseParams, skip });
  
  renderPagination(filterPagination, data.page, data.pages, p => {
    loadNavFilter(baseParams, title, p);  // Récursif : chaque bouton appelle loadNavFilter
  });
}
```

La clé de cette implémentation est la **récursivité via callback** : `renderPagination` reçoit une fonction `onPageChange`. Quand on clique sur le bouton "page 3", le callback appelle `loadNavFilter(baseParams, title, 3)`. Cette nouvelle invocation charge les données de la page 3, puis appelle à nouveau `renderPagination` avec un nouveau callback. Le système est ainsi entièrement auto-suffisant.

---

## 6. Communication Front-end / Back-end

### 6.1 Le protocole REST

Notre API respecte les conventions RESTful :

| Méthode | Route | Action |
|---------|-------|--------|
| GET | `/shows` | Récupérer le catalogue paginé |
| GET | `/shows/search?q=...` | Recherche simple |
| GET | `/shows/search/advanced` | Recherche multi-critères |
| GET | `/shows/{id}` | Détail d'un film |
| GET | `/shows/{id}/similar` | Films similaires |
| POST | `/shows/favorites` | Charger les films favoris |
| GET | `/shows/stats` | Statistiques du catalogue |
| GET | `/health` | Sonde de disponibilité |

### 6.2 Format de réponse uniforme

Toutes les réponses de liste partagent le même format JSON :

```json
{
  "items":      [...],
  "total":      25000,
  "page":       2,
  "pages":      1250,
  "limit":      20,
  "skip":       20,
  "from_cache": true,
  "engine":     "elasticsearch"
}
```

Le champ `from_cache` indique si la réponse provient de Redis — utile pour le monitoring et affiché dans la barre de statistiques de recherche. Le champ `engine` indique si Elasticsearch ou MongoDB a servi la requête.

### 6.3 Gestion des erreurs HTTP

```python
@app.get("/shows/{show_id}")
def get_show(show_id: str):
    try:
        doc = shows_collection.find_one({"_id": ObjectId(show_id)})
    except Exception:
        raise HTTPException(status_code=400, detail="ID invalide")
    if not doc:
        raise HTTPException(status_code=404, detail="Show introuvable")
    return clean_show(doc)
```

- **400 Bad Request** : l'ID fourni n'est pas un ObjectId MongoDB valide
- **404 Not Found** : l'ID est valide mais aucun document ne correspond
- **422 Unprocessable Entity** : paramètres invalides (automatiquement géré par FastAPI via `Query`)

---

## 7. Conteneurisation avec Docker

### 7.1 Concepts fondamentaux Docker

Lors du TME 5, nous avons étudié en profondeur l'écosystème Docker :

- **Image** : Modèle immuable en couches, comparable à un plan de construction. Les couches sont partagées entre images — `python:3.11-slim` est téléchargée une seule fois et utilisée par toutes les images qui en dérivent.
- **Conteneur** : Instance vivante d'une image, processus isolé via les **namespaces** et **cgroups** Linux. Il partage le noyau de l'hôte mais dispose de son propre système de fichiers, réseau, et espace de processus.
- **VM vs Conteneur** : Une VM virtualise le matériel et embarque un OS complet (900MB+). Un conteneur partage le noyau de l'hôte et ne virtualise que l'espace utilisateur (quelques dizaines de MB).

### 7.2 Dockerfile du back-end

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 1. Copier les dépendances EN PREMIER (optimisation du cache Docker)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 2. Copier le code ensuite
COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Explication ligne par ligne :**

- `FROM python:3.11-slim` : image de base Python allégée (~150MB contre ~900MB pour l'image standard). La version `slim` supprime les compilateurs et outils de développement non essentiels à l'exécution.
- `WORKDIR /app` : crée et définit le répertoire de travail. Toutes les commandes suivantes s'exécutent depuis `/app`.
- **Optimisation du cache Docker** : les dépendances (`requirements.txt`) sont copiées et installées *avant* le code source. Docker met en cache chaque couche. Si on modifie uniquement `main.py`, Docker réutilise la couche `pip install` du cache et ne réinstalle pas toutes les dépendances — économisant plusieurs minutes à chaque build.
- `EXPOSE 8000` : déclaration documentaire (n'ouvre pas réellement le port).
- `CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]` : commande par défaut au démarrage. `0.0.0.0` : accepte les connexions de l'extérieur du conteneur.

### 7.3 Dockerfile du front-end

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
COPY style.css  /usr/share/nginx/html/style.css
COPY script.js  /usr/share/nginx/html/script.js

EXPOSE 80
```

Nginx sert les fichiers statiques. L'image `nginx:alpine` pèse seulement ~40MB. C'est un serveur web haute performance — bien plus adapté à servir des fichiers statiques que FastAPI.

### 7.4 Docker Compose — Orchestration locale

```yaml
version: "3.9"

services:
  mongo:
    image: mongo:7
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.2
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - es_data:/usr/share/elasticsearch/data

  backend:
    build: ./backend
    environment:
      MONGODB_URI: mongodb://mongo:27017
      REDIS_URL: redis://redis:6379
      ES_URL: http://elasticsearch:9200
    ports:
      - "8000:8000"
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy

  frontend:
    build: ./frontend
    ports:
      - "8080:80"
    depends_on:
      - backend

volumes:
  mongo_data:
  es_data:

networks:
  netflix-net:
    driver: bridge
```

**Points importants :**

- **healthcheck** : Docker vérifie régulièrement que MongoDB répond. Grâce à `depends_on: condition: service_healthy`, le back-end ne démarrera pas tant que MongoDB n'est pas opérationnel. Sans ça, le back-end démarrerait, essaierait de se connecter à une base encore en cours d'initialisation, et planterait.
- **Résolution DNS interne** : dans Docker Compose, les services se trouvent par leur nom (`mongo`, `redis`, `elasticsearch`). Ainsi `MONGODB_URI: mongodb://mongo:27017` résout vers l'IP du conteneur MongoDB sur le réseau interne.
- **Volumes nommés** (`mongo_data`, `es_data`) : les données persistent entre les redémarrages. Sans volume, les données seraient perdues à chaque `docker compose down`.
- **`ES_JAVA_OPTS=-Xms512m -Xmx512m`** : Elasticsearch est une application Java qui consomme par défaut 1 à 2 Go de RAM. Cette option limite la JVM à 512 Mo minimum et maximum — adapté aux environnements de développement.

### 7.5 Réseau Docker

Tous les services partagent le réseau `netflix-net` de type `bridge`. Le réseau bridge est le mode par défaut — il crée un réseau privé virtuel isolé de l'hôte (via NAT). Seuls les ports explicitement publiés (`8000:8000`, `8080:80`) sont accessibles depuis la machine hôte.

---

## 8. Orchestration avec Kubernetes

### 8.1 Concepts Kubernetes

Kubernetes (K8s) est un orchestrateur de conteneurs. Il gère le cycle de vie des applications : déploiement, scaling, mise à jour et résilience automatique.

| Concept | Rôle |
|---------|------|
| **Namespace** | Espace de noms isolé pour regrouper les ressources |
| **Pod** | Unité d'exécution (un ou plusieurs conteneurs) |
| **Deployment** | Décrit l'état désiré (image, replicas, variables d'env) |
| **Service** | Adresse réseau stable pour accéder aux pods |
| **ConfigMap** | Variables de configuration non-secrètes |
| **Secret** | Variables sensibles (mots de passe, clés API) |
| **Ingress** | Routeur HTTP exposant l'application vers l'extérieur |

### 8.2 Organisation en Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netflix-app
```

Toutes nos ressources sont regroupées dans le namespace `netflix-app`. Cela isole le projet des ressources système de Kubernetes et facilite la gestion (`kubectl get all -n netflix-app`).

### 8.3 ConfigMap — Configuration centralisée

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: netflix-config
  namespace: netflix-app
data:
  MONGODB_URI: "mongodb://mongo-service:27017"
  MONGODB_DB:  "netflix"
  REDIS_URL:   "redis://redis-service:6379"
  ES_URL:      "http://elasticsearch-service:9200"
```

Le ConfigMap centralise la configuration. Les deployments référencent ce ConfigMap via `envFrom: configMapRef`. Si l'URL de MongoDB change, on modifie uniquement le ConfigMap. En Kubernetes, les services se trouvent via le **DNS interne de K8s** — `mongo-service` est automatiquement résolu vers l'IP du pod MongoDB.

### 8.4 Deployment du back-end

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: netflix-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: netflix-backend:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: netflix-config
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

**Points importants :**

- **`replicas: 1`** : un seul pod back-end. Nous avons délibérément choisi une seule réplique pour éviter une **race condition** au démarrage : si deux pods démarrent simultanément et que la base est vide, les deux exécutent `import_csv()` en même temps, doublant les données. Un seul pod résout ce problème.
- **`imagePullPolicy: Never`** : indique à Kubernetes d'utiliser l'image locale (chargée dans Minikube via `minikube image load`) et de ne jamais essayer de la télécharger depuis Docker Hub.
- **`readinessProbe`** : K8s appelle `GET /health` toutes les 10 secondes. Tant que la probe ne répond pas HTTP 200, K8s ne route pas de trafic vers ce pod. `initialDelaySeconds: 15` laisse le temps au serveur FastAPI de démarrer et d'importer les données.
- **`resources`** : `requests` = ressources que Kubernetes garantit (le pod est schedulé sur un nœud avec au moins ces ressources disponibles) ; `limits` = maximum absolu (si dépassé, K8s termine le pod avec OOMKill).

### 8.5 Deployment du front-end

```yaml
spec:
  replicas: 2
  containers:
    - name: frontend
      image: netflix-frontend:latest
      imagePullPolicy: Never
      resources:
        requests: { memory: "32Mi", cpu: "10m" }
        limits:   { memory: "64Mi", cpu: "100m" }
```

Le front-end peut avoir **2 répliques** car Nginx servant des fichiers statiques est entièrement stateless. Si un pod Nginx tombe, l'autre continue de servir l'application. Les ressources sont très faibles (32Mi RAM) car Nginx est extrêmement léger comparé au back-end.

### 8.6 L'Ingress — Routeur HTTP

```yaml
# Route /api/* → back-end
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netflix-ingress-api
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: netflix.local
      http:
        paths:
          - path: /api(/|$)(.*)
            backend:
              service:
                name: backend-service
                port:
                  number: 8000

# Route /* → front-end
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netflix-ingress-frontend
spec:
  rules:
    - host: netflix.local
      http:
        paths:
          - path: /
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

L'Ingress est le point d'entrée unique de l'application. Toutes les requêtes arrivent sur `netflix.local` et sont routées selon le chemin :
- `/api/shows` → back-end FastAPI (port 8000), le préfixe `/api` est supprimé par `rewrite-target: /$2` avant transmission
- `/` → front-end Nginx (port 80)

### 8.7 Services Redis et Elasticsearch déployés

```yaml
# redis-deployment.yaml
containers:
  - name: redis
    image: redis:7-alpine
    ports:
      - containerPort: 6379
---
# redis-service.yaml
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
  type: ClusterIP
```

Les services Redis et Elasticsearch utilisent le type `ClusterIP` — accessibles uniquement depuis l'intérieur du cluster. Ils ne sont jamais exposés à l'extérieur, ce qui est une bonne pratique de sécurité.

---

## 9. Pipeline CI/CD avec GitHub Actions

### 9.1 Philosophie de l'intégration continue

Le pipeline CI/CD (Continuous Integration / Continuous Delivery) automatise les vérifications à chaque modification du code. Il s'exécute à chaque `git push` sur `main` et à chaque Pull Request.

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

### 9.2 Job 1 : Tests du back-end

```yaml
jobs:
  test-backend:
    runs-on: ubuntu-latest

    services:
      mongo:
        image: mongo:7
        ports:
          - 27017:27017

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        working-directory: backend
        run: pip install -r requirements.txt

      - name: Test import du module
        working-directory: backend
        env:
          MONGODB_URI: mongodb://localhost:27017
          MONGODB_DB: netflix_ci
        run: python -c "import main; print('OK')"

      - name: Test endpoint /health
        working-directory: backend
        env:
          MONGODB_URI: mongodb://localhost:27017
          MONGODB_DB: netflix_ci
        run: |
          uvicorn main:app --port 8000 &
          sleep 15
          curl -f http://localhost:8000/health
          kill %1
```

**Fonctionnement détaillé :**

- GitHub Actions lance un conteneur MongoDB (`services: mongo`) accessible sur `localhost:27017` pendant l'exécution des tests — exactement comme un vrai service de test d'intégration
- `pip install -r requirements.txt` : installe les mêmes dépendances que le Dockerfile
- `python -c "import main; print('OK')"` : vérifie que le module Python s'importe sans erreur (syntaxe correcte, dépendances trouvables)
- `curl -f http://localhost:8000/health` : `-f` fait échouer la commande si le code HTTP n'est pas 200, ce qui fait échouer le step CI

### 9.3 Job 2 : Build des images Docker

```yaml
  build-docker:
    needs: test-backend
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: docker build -t netflix-backend:latest backend/
      - run: docker build -t netflix-frontend:latest frontend/
      - run: docker compose build
```

Ce job ne s'exécute **que si le job `test-backend` a réussi** (`needs: test-backend`). Il vérifie que les Dockerfiles sont syntaxiquement corrects et que les images se construisent sans erreur.

### 9.4 Valeur ajoutée du pipeline

Sans CI/CD, les erreurs se découvrent en production. Avec ce pipeline :
- Une faute de syntaxe Python est détectée en quelques secondes
- Un Dockerfile cassé est détecté avant de bloquer le déploiement
- Chaque modification est validée automatiquement

---

## 10. Fonctionnalités avancées

### 10.1 Cache Redis

**Problème résolu** : Sans cache, chaque requête `/shows` refait un aller-retour vers MongoDB, même si les données n'ont pas changé. Avec 100 utilisateurs simultanés qui regardent tous la page d'accueil, on fait 700 requêtes MongoDB inutiles (7 rangées × 100 utilisateurs).

**Implémentation :**

Toutes les réponses des endpoints GET sont mises en cache dans Redis. La clé de cache est un hash MD5 de la combinaison de tous les paramètres :

```python
cache_key = make_key("shows", limit, skip, type, genre, sort_by)
cached = cache_get(cache_key)
if cached:
    cached["from_cache"] = True
    return cached    # Réponse en quelques microsecondes

# ... calcul depuis MongoDB ...

cache_set(cache_key, result, ttl=300)   # Cache 5 minutes
```

**TTL (Time To Live) différencié selon les données :**
- `/shows`, `/shows/search` : 5 minutes (données susceptibles de changer)
- `/shows/{id}`, `/shows/stats` : 1 heure (données stables)
- `/shows/{id}/similar` : 10 minutes (recommandations)

**Dégradation gracieuse** : si Redis est indisponible, `redis_client` est `None` et `cache_get` retourne `None` immédiatement. L'application fonctionne normalement, juste sans cache.

### 10.2 Films similaires

**Algorithme :**
1. Récupérer les genres du film sélectionné depuis MongoDB
2. Construire une regex MongoDB : `Action|Crime|Drama` (les 3 premiers genres)
3. Chercher les films partageant au moins un genre, avec note ≥ 6.0
4. Récupérer 50× plus de documents que nécessaire pour compenser les doublons
5. Dédupliquer en Python par clé `(titre+année).lower()`
6. Retourner les `limit` premiers films uniques, mis en cache 10 minutes

**Le multiplicateur ×50** est une réponse directe au problème de données dupliquées. Si on veut 8 films uniques et que le dataset a 75% de doublons, il faut théoriquement en récupérer 32 pour en trouver 8 distincts. En pratique, nous récupérons jusqu'à 500 documents pour avoir une large marge.

```python
fetch_limit = min(limit * 50, 500)
seen: set = set()
for d in cursor:
    key = (str(d.get("title","")) + str(d.get("release_year",""))).lower().strip()
    if key and key not in seen:
        seen.add(key)
        items.append(clean_show(d))
        if len(items) >= limit:
            break
```

### 10.3 Favoris

**Architecture client-centrique :**
- Côté front-end : les IDs des films favoris sont stockés dans le `localStorage` du navigateur sous la clé `netplixe_favorites`
- Côté back-end : l'endpoint `POST /shows/favorites` accepte une liste d'IDs et retourne les documents complets depuis MongoDB

**Avantages de cette architecture :**
- **Sans état côté serveur** : le back-end n'a pas besoin de gérer des sessions ou des comptes utilisateurs
- **Données toujours à jour** : à chaque ouverture de "Mes Favoris", on relit MongoDB — si les métadonnées d'un film changent, les favoris affichent les informations actuelles
- **Persistance locale** : les favoris survivent aux redémarrages du navigateur

**Icône cœur dynamique :**

L'état favori/non-favori est reflété visuellement en temps réel. Le SVG du cœur passe de `fill="none"` (contour vide) à `fill="#E50914"` (rouge plein) au clic, sans rechargement de la page.

### 10.4 Recherche avancée Elasticsearch

**Apports d'Elasticsearch par rapport à MongoDB :**

| Fonctionnalité | MongoDB `$regex` | Elasticsearch |
|----------------|-----------------|---------------|
| Tolérance aux fautes | Non | Oui (`fuzziness: AUTO`) |
| Pertinence (scoring) | Non (résultat binaire) | Oui (score TF-IDF) |
| Poids par champ | Non | Oui (`title^3`) |
| Performance sur texte | Lente sans index full-text | Optimisée (index inversé) |

**Exemple de requête ES :**

Recherche "action aventure" → `multi_match` cherche dans 3 champs avec pondération :
- `title^3` : si le titre contient "action aventure" → score très élevé
- `listed_in^2` : si le genre contient "Action Adventure" → score élevé
- `description` : si la description mentionne le terme → score normal

Le champ `engine` dans la réponse indique au front-end si la recherche a utilisé Elasticsearch (`"engine": "elasticsearch"`) ou le fallback MongoDB (`"engine": "mongodb"`), affiché dans la barre de stats.

### 10.5 Pagination

La pagination est implémentée de manière **cohérente** sur tous les points d'entrée de l'application :

**Côté back-end** : toutes les réponses de liste incluent `total`, `page`, `pages`, `limit`, `skip`. Le calcul :
```python
"page":  (skip // limit) + 1,
"pages": math.ceil(total / limit),
```

**Côté front-end** : la fonction `buildPageRange` génère un intervalle intelligent :
```javascript
function buildPageRange(current, total) {
  const range = new Set([1, total]);          // Toujours page 1 et dernière
  for (let i = Math.max(2, current - 2); i <= Math.min(total - 1, current + 2); i++) {
    range.add(i);   // 2 pages avant et après la courante
  }
  return [...range].sort((a, b) => a - b);
}
```

Si on est à la page 7 sur 50, les boutons affichés sont : `1 … 5 6 [7] 8 9 … 50`. Les `…` (points de suspension) indiquent les sauts dans la séquence.

---

## 11. Difficultés rencontrées et solutions

### 11.1 Race condition à l'import des données

**Problème** : Avec `replicas: 2` dans le deployment back-end, les deux pods démarraient simultanément. Ils vérifiaient tous les deux `count_documents({}) == 0`, trouvaient la base vide, et exécutaient `import_csv()` en parallèle. Résultat : 50 000 documents au lieu de 25 000.

**Symptôme** : L'API retournait `"total": 50000` mais seulement 25 000 titres uniques. Les rangées affichaient des films répétés.

**Solution** : Réduction à `replicas: 1` pour le back-end. L'import est une opération à exécuter une seule fois — un seul pod garantit qu'elle n'est jamais doublée.

### 11.2 Pagination des catégories nav non fonctionnelle

**Problème** : Cliquer sur "Page 3" après avoir affiché "Page 2" n'actualisait pas les données. Le callback de `renderPagination` ne rechargeait pas les films — il ne faisait que scroller la page.

**Diagnostic** : Le callback de pagination ne comprenait pas les paramètres de base (quel filtre, quel genre). Il ne savait donc pas quoi recharger.

**Solution** : Création de la fonction `loadNavFilter(baseParams, title, page)` qui encapsule les paramètres de base et se rappelle elle-même dans le callback :

```javascript
async function loadNavFilter(baseParams, title, page) {
  const data = await fetchShows({ ...baseParams, skip: (page - 1) * limit });
  renderPagination(filterPagination, data.page, data.pages, p => {
    loadNavFilter(baseParams, title, p);  // Récursif : même logique pour chaque page
  });
}
```

### 11.3 Films similaires identiques

**Problème** : La section "Films similaires" affichait 8 fois le même film (ex: "One Piece Fan Letter" ×8).

**Diagnostic** : Le dataset contient le même film inséré 4 à 8 fois. En ne récupérant que 8 documents triés par note, on obtenait souvent 8 copies du même film le mieux noté.

**Solution** : Récupérer jusqu'à 500 documents et dédupliquer en Python avant de retourner les 8 premiers uniques.

### 11.4 Rangées avec peu de films différents

**Problème** : Avec `limit: 18`, les rangées n'affichaient que 3 à 5 films uniques après déduplication.

**Diagnostic** : Le dataset ayant environ 75% de doublons, 18 documents MongoDB donnaient seulement 4 à 5 films distincts.

**Solution** : Augmenter `limit: 100` dans `initRows`. Après déduplication, chaque rangée affiche environ 25 films uniques, suffisant pour le défilement horizontal.

### 11.5 Pods non mis à jour après rebuild

**Problème** : Après modification du code et rebuild de l'image Docker, les pods Kubernetes continuaient d'utiliser l'ancienne image.

**Cause** : `imagePullPolicy: IfNotPresent` avec le tag `latest` signifie "utilise l'image locale si elle existe". Kubernetes ne détectait pas que l'image avait changé.

**Solution** : 
1. Passage à `imagePullPolicy: Never` (utilisation exclusive de l'image locale Minikube)
2. Ajout de `kubectl rollout restart deployment/backend` dans `start.sh` pour forcer le rechargement des pods après chaque rebuild

---

## 12. Conclusion

### 12.1 Bilan technique

Ce projet nous a permis de concevoir et déployer une application web complète en suivant une démarche professionnelle, de l'architecture initiale jusqu'au pipeline CI/CD.

**Fonctionnalités livrées :**

| Fonctionnalité | Statut | Technologie |
|----------------|--------|-------------|
| Catalogue dynamique de films | ✅ | FastAPI + MongoDB |
| Recherche par titre/description | ✅ | MongoDB `$regex` |
| Filtres par genre et type | ✅ | MongoDB query |
| Affiche TMDB haute résolution | ✅ | CDN TMDB |
| Modal détaillé | ✅ | JavaScript DOM |
| Films favoris | ✅ | localStorage + MongoDB |
| Films similaires | ✅ | MongoDB + déduplication Python |
| Cache Redis | ✅ | Redis 7 + TTL adaptatif |
| Recherche avancée multi-critères | ✅ | Elasticsearch 8.12 |
| Pagination complète | ✅ | Back-end métadonnées + JS dynamique |
| Interface responsive | ✅ | CSS media queries |
| Conteneurisation | ✅ | Docker + Docker Compose |
| Orchestration | ✅ | Kubernetes (Minikube) |
| Pipeline CI/CD | ✅ | GitHub Actions |

### 12.2 Architecture finale déployée

```
Internet
    │  http://netflix.local
    ▼
┌─────────────────────────────────────────────────────────┐
│               Kubernetes Ingress (nginx)                 │
│     /api/* → backend-service  │  /* → frontend-service  │
└──────────────────────────────────────────────────────────┘
         │                                │
┌────────▼────────┐              ┌────────▼────────┐
│  Backend FastAPI│              │ Frontend Nginx  │
│  (1 réplica)    │              │  (2 réplicas)   │
│  Pod K8s        │              │  Pod K8s        │
└────────┬────────┘              └─────────────────┘
         │
         ├── MongoDB     (stockage principal, 25k films)
         ├── Redis        (cache API, TTL 5-60 min)
         └── Elasticsearch (recherche full-text + fuzzy)
```

### 12.3 Compétences acquises

Ce projet nous a permis de maîtriser des compétences directement applicables en entreprise :

- **Conception** : choix d'architecture justifiés, identification des design patterns, anticipation des problèmes de scalabilité
- **Back-end** : API RESTful avec FastAPI, validation des données, gestion d'erreurs HTTP, connexion multi-bases
- **Front-end** : manipulation avancée du DOM, fetch API asynchrone, CSS animations, responsive design
- **Bases de données** : MongoDB (requêtes, index, agrégation), Redis (cache avec TTL), Elasticsearch (index inversé, scoring)
- **Docker** : Dockerfiles optimisés (layering, cache), Docker Compose (healthchecks, dépendances)
- **Kubernetes** : Deployments, Services, ConfigMaps, Secrets, Ingress, readinessProbe, gestion des ressources
- **CI/CD** : GitHub Actions, tests d'intégration, build automatisé

### 12.4 Perspectives d'amélioration

Pour une mise en production réelle, les évolutions prioritaires seraient :

1. **Authentification** : système de comptes utilisateurs (JWT) pour des favoris persistants côté serveur
2. **Déploiement cloud** : migration Minikube → GKE / EKS / AKS avec autoscaling HPA
3. **Monitoring** : Prometheus + Grafana pour visualiser les métriques (latence API, taux de hit Redis, etc.)
4. **Tests unitaires** : couverture des endpoints FastAPI avec pytest + httpx
5. **CDN** : mise en cache des assets statiques via un CDN (Cloudflare) pour réduire la charge Nginx

---

*Rapport rédigé dans le cadre du module LU3IN403 — Opérations et Systèmes Cloud*  
*Sorbonne Université — Licence 3 Informatique*
