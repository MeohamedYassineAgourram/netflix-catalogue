# TME 3: Utiliser une API r&eacute;elle + collecte de donn&eacute;es (scraping / export)

## 1. Prise en main de l'API TMDB
### 1.1 Mécanisme d'authentification
Pour utiliser l'API de The Movie Database (TMDB), l'authentification est obligatoire. On a créé un compte développeur et généré mes identifiants.

L'authentification peut se faire de deux manières :       
    --> Par Clé API (API Key v3) : On ajoute un paramètre api_key=MA_CLE directement dans l'URL de la requête (Query Parameter). C'est la méthode qu'on a utilisée pour mes tests initiaux car elle est simple à mettre en place avec curl ou un navigateur.   
    --> Par Token (Bearer Token v4) : C'est une méthode plus sécurisée où la clé est envoyée dans les en-têtes (Headers) de la requête HTTP (Authorization: Bearer MON_TOKEN). C'est la méthode recommandée pour la production.

### 1.2 Les Endpoints identifiés
En lisant la documentation "Getting Started", on a identifié les endpoints (points d'entrée) principaux suivants :      
    --> Base URL : https://api.themoviedb.org/3        
    --> Films populaires : GET /movie/popular (Permet de récupérer les films du moment).       
    --> Recherche : GET /search/movie (Permet de trouver un film par mots-clés).        
    --> Détails : GET /movie/{movie_id} (Donne les infos complètes d'un film précis via son ID).       
    --> Tendances : GET /trending/movie/day (Les films qui buzzent aujourd'hui).        

### 1.3 Structure de la réponse (Format JSON)
L'API renvoie les données au format JSON.
La structure générale d'une réponse de type "liste" (comme Popular ou Search) est composée de :        
    --> page : Le numéro de la page consultée.         
    --> results : Un tableau (liste) contenant les objets films.          
    --> total_pages et total_results : Des métadonnées utiles pour gérer la pagination.         
Chaque film dans results contient des champs clés comme id, title, overview (synopsis) et poster_path (le lien partiel vers l'image).

### 1.4 Gestion des erreurs et Quotas
Il est important de gérer les codes de retour HTTP pour rendre l'application robuste :          
    --> 200 OK : La requête a réussi.            
    --> 401 Unauthorized : La clé API est manquante ou invalide (échec d'authentification).              
    --> 404 Not Found : L'endpoint demandé n'existe pas (ex: faute de frappe dans l'URL) ou l'ID du film est introuvable.         
    --> 429 Too Many Requests : Le quota d'appels a été dépassé. TMDB impose une limite (Rate Limit) pour éviter les abus. Si cette erreur survient, il faut attendre avant de refaire une requête.

### 1.5 Tests de connexion (Preuves)
Test 1 : Récupération des films populaires
J'ai effectué la commande suivante dans le terminal de ma VM :
```bash
    curl "https://api.themoviedb.org/3/movie/popular?api_key=XXXXXXXXXXXXXXXX&language=fr-FR"
```

<div align="center">
  <img src="./curl_TMDB_011.PNG" width=60%>
</div>

Test 2 : Recherche d'un film spécifique
Commande utilisée :
```bash
    curl "https://api.themoviedb.org/3/search/movie?api_key=XXXXXXXXXXXXXXXX&query=Batman&language=fr-FR"
```

<div align="center">
  <img src="./curl_TMDB_022.PNG" width=60%>
</div>


## 2. Intégration et adaptation du Back-end
### 2.1 Objectif et Architecture (Le concept de "Proxy")
Dans cette partie, l'objectif était d'adapter le back-end existant (réalisé sous FastAPI lors du TME 2) pour qu'il consomme l'API publique TMDB et expose sa propre API au front-end.

On a mis en place une architecture de type API Proxy / Agrégateur. Dans le monde réel, le front-end n'appelle presque jamais une API externe directement pour plusieurs raisons :
    --> Sécurité : La clé API TMDB (TMDB_API_KEY) est stockée de manière sécurisée côté serveur via un fichier .env et la bibliothèque python-dotenv. Elle n'est jamais exposée au client.
    --> Contrôle et nettoyage : Le back-end filtre les données brutes avant de les envoyer.
    --> Performance : Cela permet d'ajouter une couche de cache serveur.

### 2.2 Création de la route /movies et Standardisation
On a modifié la route /movies existante. Au lieu de lire un fichier JSON local statique, elle interroge désormais l'endpoint /movie/popular de TMDB.

Pour que le front-end n'ait pas à subir de changements majeurs, on a créé une fonction de standardisation des données. TMDB renvoie beaucoup d'informations inutiles pour l'interface. La fonction extrait uniquement les champs nécessaires et les renomme pour garder un format cohérent :
    --> title : récupéré depuis title (avec une valeur par défaut "Titre Inconnu" si manquant).
    --> year : extrait des 4 premiers caractères de release_date.
    --> description : récupéré depuis overview.
    -->image_url : reconstruit en combinant l'URL de base des images TMDB (https://image.tmdb.org/t/p/w500) et le poster_path. Si l'image est manquante, un placeholder (image de remplacement) est injecté côté serveur pour ne pas casser le design du front-end.
    --> tmdb_id : conservé pour une éventuelle utilisation future.

### 2.3 Gestion des erreurs (Robustesse)
Pour rendre l'API robuste, les erreurs potentielles lors de l'appel à TMDB sont interceptées via des blocs try...except et renvoient des messages HTTP propres côté client :
    --> Erreur de configuration (Code 500) : Si la clé API n'est pas trouvée dans le fichier .env au lancement.
    --> Erreur d'authentification TMDB (Code 401) : Si la clé fournie à TMDB est invalide.
    --> Erreur de Rate-Limit (Code 429) : Gérée si TMDB signale que trop de requêtes ont été effectuées.
    --> Erreurs réseau (Code 503) : Interceptées (requests.exceptions.ConnectionError) si le serveur perd sa connexion internet.

### 2.4 Implémentation d'une couche de Cache (Option recommandée réalisée)
Afin d'éviter d'épuiser les quotas de l'API TMDB (et d'éviter l'erreur 429) à chaque rafraîchissement du front-end, on a mis en place un système de cache simple en mémoire.
    --> Fonctionnement : Lors du premier appel, les données nettoyées sont stockées dans un dictionnaire global (variable cache_films) avec l'horodatage exact (time.time()).
    --> Avantage : Pendant une durée définie (ex: 60 secondes), tous les appels ultérieurs à /movies liront directement ces données en mémoire vive, sans faire de nouvelle requête HTTP vers TMDB. Cela améliore considérablement les performances et protège la clé API.


## 4. Collecte de données et Export
### 4.1 Stratégie de collecte et implémentation technique
Pour cette partie, on a choisi d'utiliser l'API officielle de TMDB (endpoint /movie/popular) plutôt que de faire du web scraping.          
    --> Quels films et combien ? Notre choix s'est porté sur l'extraction des films les plus populaires. Afin d'avoir un échantillon représentatif sans surcharger l'API, on a configuré notre script pour boucler sur les 5 premières pages de résultats, récupérant ainsi 100 films.                     
    -->Format et structure : Les données ont été exportées au format JSON, car c'est un format structuré idéal pour réimporter ces données dans une base de données NoSQL ultérieurement. Les images ne sont pas téléchargées localement pour économiser de l'espace disque; seule l'URL absolue (image_url) est sauvegardée.                
    --> Mise en œuvre : on a créé une route spécifique /export/movies dans mon back-end FastAPI. Lors de son appel, le serveur effectue les 5 requêtes HTTP, nettoie les données via la fonction de standardisation, et écrit un fichier horodaté dans un sous-dossier /exports.

### 4.2 Réflexion éthique et légale : API vs Web Scraping
Le choix de l'API plutôt que du web scraping (extraction directe du code HTML des pages web) a été fait pour des raisons techniques et légales :                       
    --> Légalité et CGU : Le web scraping est souvent limité ou interdit par les Conditions Générales d'Utilisation des plateformes. Consommer une API officielle garantit que l'on respecte les règles édictées par les créateurs de la donnée.     
    --> Respect de l'infrastructure (robots.txt) : Un script de scraping mal conçu peut surcharger les serveurs d'un site web. Bien que le scraping soit parfois toléré si l'on respecte les directives du fichier robots.txt et que l'on limite la fréquence des requêtes (User-Agent propre, délais entre les requêtes), l'API reste le moyen le plus sûr et éthique.        
    --> Fiabilité technique : Le scraping est fragile (si TMDB change la couleur d'un bouton ou la classe d'une balise div en HTML, le script casse). L'API garantit une structure de données (JSON) stable dans le temps.


## 5. Bonnes pratiques avancées (API réelle)
Dans le cadre de l'intégration d'une API externe comme TMDB, il est crucial de prendre en compte les contraintes du monde réel. Voici une synthèse des concepts clés, accompagnée d'une analyse de mon implémentation et des évolutions possibles pour un environnement de production.

### 5.1 Synthèse (avec sources)
--> Rate limiting et stratégies : Les API publiques limitent le nombre de requêtes pour protéger leurs serveurs (renvoyant souvent le code HTTP 429 Too Many Requests). Pour gérer cela sans faire planter l'application cliente, on utilise des stratégies de Retry (réessayer la requête) couplées à un algorithme d'Exponential Backoff (augmenter progressivement le temps d'attente entre chaque essai pour ne pas saturer le réseau) (Source : Documentation Google Cloud API Design).            

--> Cache (local / serveur) : Le cache permet de stocker temporairement des données fréquemment demandées. Un cache serveur (ex: Redis) évite de refaire des appels à l'API tierce, tandis qu'un cache local (ex: LocalStorage du navigateur web) évite au client de refaire des requêtes au serveur. (Source : MDN Web Docs - HTTP Caching).                   

--> Pagination : Transférer de grandes quantités de données dégrade les performances. La pagination (offset/limit ou par numéro de page) divise les réponses en sous-ensembles gérables. (Source : TMDB API Documentation).                      

--> Sécurité (Gestion des secrets) : Les clés d'API ne doivent jamais être exposées dans le code front-end (HTML/JS), car elles sont visibles par tous les utilisateurs. Le back-end agit comme un Proxy sécurisé, utilisant des variables d'environnement (.env) non incluses dans les systèmes de versioning comme Git. (Source : OWASP Top 10 - Cryptographic Failures).                 

--> Logs et Observabilité : L'observabilité permet de surveiller la santé de l'application. Les logs (journaux d'événements) enregistrent les erreurs, les requêtes entrantes et les temps de réponse, ce qui est vital pour le débogage asynchrone. (Source : The Twelve-Factor App - Logs).

### 5.2 Ce que on a mis en place dans ce TME
Dans le cadre de notre application, on a implémenté plusieurs de ces bonnes pratiques :                   
    --> Sécurité : on a complètement masqué ma clé API TMDB du front-end en créant un proxy via FastAPI. La clé est sécurisée dans un fichier .env lu via python-dotenv.              
    --> Cache Serveur simple : on a mis en place un cache en mémoire vive (un dictionnaire Python stockant les films et un horodatage time.time()). Pendant une durée de 60 secondes, les appels du front-end ne déclenchent aucune requête vers TMDB, protégeant ainsi notre quota.                
    --> Gestion de la Pagination / Limite : Le front-end peut envoyer un paramètre ?limit=N au back-end pour limiter la taille de la réponse (ex: charger uniquement 12 films), réduisant la charge réseau.                  
    --> Gestion des erreurs (Rate Limit) : on a implémenté des blocs try...except dans notre back-end pour intercepter les erreurs HTTP de TMDB (y compris l'erreur 429) afin de renvoyer un message d'erreur contrôlé au lieu de faire planter le serveur.

### 5.3 Ce qu'on ferais pour une mise en production réelle
Si cette application devait être déployée pour des milliers d'utilisateurs (Production), voici les évolutions qu'on apporterais : 
    --> Rate Limiting côté Front et Back : on ajouterais une bibliothèque de Rate Limiting (comme slowapi) sur notre propre serveur FastAPI pour empêcher qu'un attaquant ne spamme notre serveur. on implémenterais également l'algorithme d'Exponential Backoff sur nos requêtes requests.get vers TMDB.            
    --> Cache Robuste : on remplacerais le dictionnaire Python en mémoire par un véritable système de cache distribué comme Redis ou Memcached.      
    --> Pagination Front-End complète : on ajouterais un bouton "Charger plus" (ou un "Infinite Scroll") sur l'interface utilisateur, qui irait interroger les pages 2, 3, etc., de notre API de manière asynchrone.           
    --> Système de Logs : on intégrerais la bibliothèque standard logging de Python pour écrire les erreurs dans un fichier dédié, et on utiliserais un outil de monitoring (comme Sentry ou Datadog) pour être alerté en temps réel en cas d'augmentation anormale du taux d'erreurs (erreurs 500).