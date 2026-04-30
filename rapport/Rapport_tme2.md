# TME 2: Conception et mise en place d'une application Web (Front-end & Back-end)

## Partie 1: Architecture & conception

### --> &Eacute;tude des architectures existantes

#### A. L'Architecture Monolithique

Dans un monolithe, l'ensemble de l'application (UI, logique métier et accès aux données) est développé, compilé et déployé comme une seule unité logique. Tout le code partage la même mémoire et les mêmes ressources.

- Avantages : Simplicité de développement et de déploiement initial. - Performances élevées (pas d'appels réseau entre les composants). - Facilité de test de bout en bout.
- Limites : "Spaghetti code" : avec le temps, les composants deviennent trop entrelacés. - Scalabilité difficile : on est obligé de dupliquer toute l'application même si seul un petit module est surchargé. - Risque systémique : une erreur critique dans un module peut faire tomber toute l'application.
- Cas d'usage : Petites applications, preuves de concept (PoC) ou équipes très réduites.

#### B. L'Architecture Client-Serveur

Ce modèle sépare l'application en deux entités : le Client (l'interface utilisateur) qui initie des requêtes, et le Serveur qui traite ces requêtes et renvoie les ressources. Ils communiquent généralement via le protocole HTTP.

- Avantages : \* Séparation des responsabilités (UI vs Logique). - Indépendance technologique : on peut changer le langage du client sans toucher au serveur. - Centralisation des données et de la sécurité sur le serveur.
- Limites : Dépendance totale à la connectivité réseau. - Le serveur peut devenir un "goulot d'étranglement" s'il reçoit trop de requêtes.
- Cas d'usage : La majorité des sites web modernes et des applications mobiles.

#### C. L'Architecture en Couches (Layered Architecture)

Également appelée "n-tier", cette architecture organise le code en strates horizontales (généralement : Présentation, Application, Domaine/Métier, Infrastructure/Données). Chaque couche a un rôle précis et ne peut communiquer qu'avec la couche immédiatement inférieure.

- Avantages : Très forte maintenabilité : le code est rangé logiquement. - Facilité de remplacement : on peut changer la base de données sans modifier l'interface. - Testabilité accrue grâce au découplage.
- Limites : "Effet de cascade" : un changement mineur en bas peut parfois remonter toutes les couches. - Peut introduire une complexité inutile pour des fonctions très simples.
- Cas d'usage : Logiciels d'entreprise complexes et robustes.

#### D. L'Architecture Microservices

L'application est décomposée en une suite de services autonomes, spécialisés dans une fonction métier unique (ex: service de recherche, service de notation, service utilisateur). Chaque service possède sa propre base de données.

- Avantages : Scalabilité granulaire : on multiplie uniquement les services les plus utilisés. - Résilience : la panne d'un microservice ne paralyse pas l'ensemble du système. - Agilité : chaque équipe peut travailler sur son propre service avec ses propres technos.

- Limites : Complexité opérationnelle extrême (réseau, déploiement, surveillance). - Difficulté à maintenir la cohérence des données entre les services.
- Cas d'usage : Systèmes à très grande échelle (Netflix, Amazon, Uber).

### --> &Eacute;tude des design patterns

#### A. MVC (Model-View-Controller)

- Problème résolu : Le mélange du code d'interface utilisateur avec la logique métier et la gestion des données.
- Présentation : Ce pattern divise l'application en trois composants : le Modèle (données), la Vue (interface) et le Contrôleur (logique de lien).
- Avantages : Permet de modifier l'interface sans toucher à la logique de données. Facilite le travail en équipe.
- Limites : Peut devenir complexe à gérer si les interactions entre la vue et le modèle sont trop nombreuses.

#### B. Repository

- Problème résolu : Le couplage direct entre la logique de l'application et le système de stockage des données (Base de données, fichier JSON).
- Présentation : Il agit comme une couche d'abstraction (une interface) entre la logique métier et la source de données.
- Avantages : On peut changer de base de données (ex: passer de JSON à PostgreSQL) sans modifier une seule ligne de la logique métier.
- Limites : Ajoute une couche de code supplémentaire qui peut sembler inutile pour des applications très simples.

#### C. Singleton

- Problème résolu : La création multiple et inutile d'objets gourmands en ressources (comme une connexion à une base de données).
- Présentation : Garantit qu'une classe ne possède qu'une seule et unique instance accessible globalement.
- Avantages : Économise de la mémoire et centralise l'accès à une ressource partagée.
- Limites : Introduit un état global, ce qui peut rendre les tests unitaires plus difficiles.

#### D. Factory (Fabrique)

- Problème résolu : La complexité liée à l'instanciation d'objets sans connaître exactement leur type final au moment de la rédaction du code.
- Présentation : Fournit une interface pour créer des objets dans une classe mère, tout en laissant les sous-classes décider du type d'objet à créer.
- Avantages : Découple le code qui utilise l'objet du code qui le crée.
- Limites : Rend le code plus abstrait et parfois plus difficile à lire pour les débutants.

#### E. Observer

- Problème résolu : La nécessité de notifier plusieurs composants lorsqu'un changement survient dans un objet, sans que cet objet ne connaisse ses observateurs.
- Présentation : Un objet "Sujet" maintient une liste de ses dépendants "Observateurs" et les informe automatiquement de tout changement d'état.
- Avantages : Favorise un couplage faible entre les objets.
- Limites : Si mal géré, peut entraîner des fuites de mémoire ou des notifications en cascade imprévues.

### --> Choix et Justification

Pour ce projet, nous avons retenu les choix suivants :

#### Architecture : Client-Serveur

- Raison du choix : C'est l'architecture la plus adaptée à l'utilisation de Docker. Elle nous permet de créer deux conteneurs isolés : un pour le Front-end et un pour le Back-end. Cela facilite également le déploiement sur Kubernetes via des services distincts.
- Pourquoi pas les autres ? L'architecture monolithique rendrait la conteneurisation moins pertinente et limiterait la scalabilité. Les microservices, bien que puissants, introduiraient une complexité de gestion réseau disproportionnée par rapport à la taille actuelle de notre équipe et du projet.

#### Design Patterns : MVC et Repository

- Raison du choix :
  - MVC : Pour assurer une séparation stricte entre nos composants React/JS (Vue) et notre logique FastAPI (Contrôleur).
  - Repository : Ce choix est stratégique. Notre application utilise initialement un fichier movies.json. Le pattern Repository nous permettra de migrer plus tard vers PostgreSQL ou MongoDB (exigences du projet final) sans avoir à réécrire la logique de recherche ou de recommandation.
- Pourquoi pas les autres ? Nous avons limité nos choix à deux patterns majeurs pour éviter une "sur-conception" (over-engineering) qui alourdirait inutilement le code initial.

#### Conception de l'Application

<div align="center">
  <img src="./architecture.png" width="500">
</div>

#### Organisation Globale et Rôles

- Front-end (La Vue) : Développé en HTML/CSS/JS (ou React). Son rôle est d'envoyer les recherches de l'utilisateur au Back-end et d'afficher les cartes de films de manière esthétique et ergonomique.
- Back-end (Le Contrôleur/Logique) : Développé avec FastAPI. C'est le cerveau de l'application. Il reçoit les requêtes, filtre les films, et prépare les données au format JSON pour le client.
- Données (Le Modèle) : Actuellement stockées dans movies.json. Ce composant est la source d'information brute. Il est isolé par le pattern Repository pour garantir une évolution facile vers une base de données SQL ou NoSQL.

L'ensemble de cette structure est conçu pour être cloud-native, facilitant l'automatisation via des pipelines CI/CD.

## Partie 2: Back-end & API

### 2.2 Étude de la Stack Back-end

#### --> Étude comparative des langages

Pour construire notre API, nous avons comparé trois écosystèmes majeurs :

- Python (FastAPI): - API rapides, Data Science, IA, Prototypage. - Très productif, typage clair, doc automatique. - Un peu moins rapide que Go ou Java en calcul pur.
- Node.js (Express): - Applications temps réel, I/O intensifs, Web. - Même langage que le Front-end (JS), immense communauté. - Gestion de la concurrence parfois complexe (callback hell).
- Java (Spring Boot): - Grandes entreprises, banques, systèmes critiques. - Très robuste, extrêmement performant, sécurisé. - Courbe d'apprentissage raide, verbeux.

#### --> Analyse par KPI (Key Performance Indicators)

Nous avons défini 4 indicateurs pour valider notre choix en contexte "entreprise" :

- Productivité (Vitesse de développement) : Capacité à livrer une fonctionnalité rapidement.
- Courbe d'apprentissage : Facilité pour un nouvel ingénieur de rejoindre le projet.
- Performance (Latence) : Rapidité de réponse aux requêtes des utilisateurs.
- Écosystème : Disponibilité de bibliothèques tierces (recommandation, accès DB).

#### Choix de la Stack et Justification

Le choix retenu est Python avec le framework FastAPI.

- Justification : Python offre la meilleure Productivité et la Courbe d'apprentissage la plus douce pour notre équipe. Dans le cadre d'un système de recommandation de films, Python est imbattable grâce à son écosystème de traitement de données.
- FastAPI a été choisi car il est nativement "Cloud-Native" (très léger pour Docker) et génère automatiquement une documentation interactive (Swagger), ce qui est un KPI crucial pour la collaboration en entreprise.

### 2.3 Étape 1 -- Implémentation : API minimale (Hello World)

#### --> L'Environnement Virtuel (venv)

Un environnement virtuel est un espace isolé sur l'ordinateur dédié à un projet spécifique.

- Pourquoi l'utiliser ? Il permet d'installer des bibliothèques (comme FastAPI) sans polluer le système global. Chaque projet peut avoir ses propres versions d'outils.
- Problème résolu en entreprise : Il élimine le problème du "Ça marche sur mon ordinateur, mais pas sur le serveur". En figeant les versions dans un environnement virtuel, on s'assure que tous les développeurs travaillent exactement avec les mêmes outils.

### 2.4 Étape 2: API avec données (JSON)

#### --> Étude du format de données JSON

#### Qu'est-ce que le format JSON ?

Le JSON (JavaScript Object Notation) est un format de données textuel, léger et structuré. Bien qu'il soit dérivé de la syntaxe des objets JavaScript, il est totalement indépendant du langage de programmation. Il repose sur deux structures principales :

- Des paires Clé : Valeur (ex: "title": "Inception")
- Des Listes ordonnées de valeurs (Tableaux)

#### Pourquoi JSON est-il le standard des API Web ?

Le JSON s'est imposé comme le standard industriel pour plusieurs raisons :

- Interopérabilité : Il est lisible par quasiment tous les langages de programmation (Python, JS, Java, etc.).
- Légèreté : Sa structure est très concise par rapport à d'autres formats, ce qui réduit la bande passante nécessaire et accélère les transferts réseau.
- Facilité d'utilisation : Il est "human-readable" (lisible par l'homme) et très facile à transformer en objets manipulables dans le code (parsing).
- Support natif du navigateur : Comme le Web repose sur JavaScript, le format JSON est traité de manière optimale côté Front-end.

#### Comparaison avec d'autres formats

Il existe d'autres formats de données utilisés selon les besoins spécifiques du projet :

- XML (eXtensible Markup Language) : - Description : Format utilisant des balises personnalisées (similaire au HTML). - Cas d'usage : Très utilisé dans les anciens systèmes d'échange de données (SOAP) ou pour les fichiers de configuration complexes (ex: fichiers de projet Java Maven).

- YAML (YAML Ain't Markup Language) : - Description : Format basé sur l'indentation (les espaces), privilégiant la lisibilité humaine. - Cas d'usage : C'est le standard pour la configuration d'outils DevOps tels que Docker et Kubernetes, que nous utiliserons plus tard dans ce projet.

- CSV (Comma-Separated Values) : - Description : Format très simple où chaque ligne représente un enregistrement et chaque donnée est séparée par une virgule. - Cas d'usage : Idéal pour l'exportation de bases de données vers des tableurs comme Microsoft Excel.

#### Qu'est-ce qu'une API (Application Programming Interface) ?

Une API est une interface qui permet à deux logiciels de communiquer entre eux. Dans notre cas, il s'agit d'une API Web (HTTP).

- Rôle : Elle sert de "contrat" entre le Client (Front-end) et le Serveur (Back-end). Le serveur expose des points d'accès précis et le client sait exactement comment les appeler.
- Endpoints (Routes) : Ce sont les URL spécifiques disponibles (ex: /movies). Chaque endpoint correspond à une action précise (récupérer des films, en ajouter un, etc.).
- Format de réponse : L'API renvoie des données structurées (généralement du JSON) et non une page web visuelle (HTML).

## Partie 3: Front-end

### 3.2 Choix de la Stack Front-end

Nous avons comparé trois approches techniques pour le développement de l'interface client.

#### Définition des KPI (Key Performance Indicators)

Pour guider notre choix, nous avons utilisé les critères suivants :

- Complexité (Setup) : Temps et effort nécessaires pour afficher "Hello World".
- Maintenabilité : Facilité à gérer le code quand l'application grossit.
- Marché de l'emploi : Demande des entreprises pour cette technologie.
- Performance : Rapidité de chargement pour l'utilisateur.

- Tableau Comparatif

| KPI / Technologie      | Vanilla JS (HTML/CSS/JS)                                | React (via Vite)                                   | Vue.js                                  |
| :--------------------- | :------------------------------------------------------ | :------------------------------------------------- | :-------------------------------------- |
| **Complexité (Setup)** | **Très Faible** (Aucune installation requise)           | **Moyenne** (Nécessite Node.js, NPM, Build tools)  | **Faible/Moyenne**                      |
| **Maintenabilité**     | Faible sur les gros projets (Code "spaghetti" possible) | **Excellente** (Architecture par composants)       | Très bonne                              |
| **Marché de l'emploi** | Base indispensable, mais rarement suffisant seul        | **Dominant** (Standard de l'industrie)             | Fort (Alternatives populaires)          |
| **Performance**        | **Maximale** (Pas de surcouche)                         | Très bonne (Virtual DOM)                           | Très bonne                              |
| **Cas d'usage**        | Prototypes, petites pages, apprentissage des bases      | Applications complexes (SPA), Gestion d'état lourd | Compromis entre simplicité et puissance |

#### Choix et Justification

Stack retenue pour ce TME : HTML / CSS / JavaScript (Sans Framework).

Justification :

- Approche Pédagogique : L'objectif immédiat est de comprendre les mécanismes fondamentaux de la communication HTTP (fetch) et la manipulation du DOM. Utiliser un framework comme React masquerait cette complexité utile à l'apprentissage.
- Légèreté : Pour afficher une simple liste de 5 films, l'utilisation d'un framework complet (React) serait disproportionnée ("over-engineering") à ce stade du projet.
- Rapidité de mise en œuvre : Pas besoin de configurer d'outils de build (Webpack/Vite) ou d'installer des dépendances node_modules. Un simple navigateur suffit.

#### Résultat final:

<div align="center">
  <img src="./interface.PNG" width=90%>
</div>


## Partie 4: Communication front-end / back-end

### 4.3 Mise à jour du back-end (côté serveur)

#### Qu'est-ce que le CORS
Le CORS est une sécurité intégrée aux navigateurs web (Chrome, Firefox, etc.). Par défaut, pour des raisons de sécurité, un navigateur interdit à un site web (le "Front") de faire des requêtes vers un serveur (le "Back") si ces deux-là n'ont pas la même origine.
Qu'est-ce qu'une "Origine" ? C'est la combinaison de trois éléments :
- Le Protocole (http vs https)
- Le Domaine (localhost vs mon-api.com)
- Le Port (5173 vs 8000)
Si l'un de ces trois éléments change, c'est une "Cross-Origin" (origine différente).

#### Pourquoi ça bloque ?
Le navigateur bloque l'appel pour te protéger. Imagine que tu sois connecté à ton compte bancaire (banque.com) et que, sur un autre onglet, tu visites un site malveillant (pirate.com).
Sans le CORS, le site pirate.com pourrait envoyer une requête à banque.com en utilisant ta session (tes cookies) pour vider ton compte. Le navigateur dit donc : "Je bloque tout, sauf si le serveur m'autorise explicitement à laisser passer ce site spécifique."

#### Comment résoudre le problème (Exemple FastAPI)
Pour régler ça, on ne modifie pas le Front, on configure le Back. On dit au serveur : "Ok, j'accepte que le site situé à l'adresse http://localhost:5173 me parle."
Voici comment l'implémenter avec FastAPI :

```py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# 1. On définit la liste des origines autorisées (ton Front)
origins = [
    "http://localhost:5173",  # Port par défaut de Vite/React/Vue
    "http://127.0.0.1:5173",
]

# 2. On ajoute le middleware CORS au serveur
app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,           # Autorise mon Front
    allow_credentials=True,
    allow_methods=["*"],             # Autorise tous les verbes (GET, POST, etc.)
    allow_headers=["*"],             # Autorise tous les headers
)

@app.get("/")
def read_root():
    return {"message": "Le CORS est configuré !"}
```

### 4.5 Analyse : Où traiter les données ? (Client vs Serveur)
Dans une architecture Client-Serveur, le choix du lieu de traitement impacte directement la performance et la sécurité de l'application.

#### 1. Synthèse des types de traitement
| Fonctionnalité         | Lieu recommandé              | Justification                                                                              |
| :--------------------- | :--------------------------- | :----------------------------------------------------------------------------------------- |
| **Filtrage / Tri**     | **Mixte (Dépend du volume)** | **Client** pour la réactivité si < 100 items. Serveur si > 1000 items.                     |
| **Pagination**         | **Serveur**                  | Évite le transfert de données inutiles (économie de bande passante).                       |
| **Validation**         | **Double (Obligatoire)**     | **Client** pour l'UX (immédiat). Serveur pour la sécurité (crucial).                       |
| **Sécurité / Auth**    | **Serveur**                  | Le client est "public", toute logique sensible doit être cachée.                           |


#### 2. Analyse détaillée par cas d'usage
A. Le Filtrage (ex: par genre ou titre)
  --> Côté Client : Si on a 50 films, il est préférable de tous les envoyer au client. Le filtrage est alors instantané (pas de temps de chargement entre chaque clic).
  --> Côté Serveur : Si on a 10 000 films, envoyer tout le fichier JSON bloquerait le navigateur. Le serveur doit alors faire une requête SQL WHERE genre = 'Action' et n'envoyer que le résultat.

B. La Pagination
  --> Pourquoi le Serveur ? C'est la règle d'or. La pagination doit presque toujours être gérée par le serveur (via des paramètres comme limit et offset).
  --> Justification : Il est inutile de charger 500 images si l'utilisateur ne regarde que les 10 premières. Cela préserve la batterie des mobiles et la mémoire vive.

C. Validation et Sécurité
  --> Le principe "Never Trust the Client" : Un utilisateur malveillant peut facilement modifier le code JavaScript dans son navigateur pour contourner une validation (ex: changer un prix).
  --> Le rôle du Serveur : Le serveur est le seul garant de l'intégrité des données. Il doit re-vérifier chaque information reçue avant de l'enregistrer.
  --> Le rôle du Client : Ici, le traitement côté client sert uniquement au confort (UX). Il permet d'afficher un message d'erreur rouge avant même que l'utilisateur n'appuie sur "Envoyer".

#### 3. Conclusion pour notre projet
Pour notre catalogue de films :
  --> Le stockage et la limitation (limit) sont traités côté Serveur (dans main.py). C'est plus robuste.
  --> L'affichage et la mise en forme sont traités côté Client (script.js).
  --> Prochaine évolution : Si nous ajoutons une barre de recherche, nous devrons décider si le filtre se fait en JS (rapide) ou via une nouvelle route API (plus professionnel pour de gros catalogues).

#### Sources et bonnes pratiques
MDN Web Docs - Client-side vs Server-side
OWASP Top 10 - Injection and Insecure Design (Pourquoi la validation serveur est obligatoire)

### 4.6 Étude : Ce qui distingue une bonne d'une mauvaise API
La qualité d'une API (Application Programming Interface) se mesure à sa facilité d'utilisation, sa robustesse et sa prédictibilité.

#### 1. Synthèse : Bonne vs Mauvaise API
| Caractéristique        | Une Bonne API                          | Une Mauvaise API                                                                           |
| :--------------------- | :------------------------------------- | :----------------------------------------------------------------------------------------- |
| **Codes HTTP**         | Utilise des noms au pluriel (/movies). | Utilise des verbes (/get-all-movies).                                                      |
| **Codes HTTP**         | Codes précis (200, 400, 404, 500).     | Renvoie toujours 200, même en cas d'erreur.                                                |
| **Erreurs**            | Messages clairs : "Limit must be > 0". | "Internal Server Error" ou crash sans message.                                             |
| **Documentation**      | Auto-générée et interactive (Swagger). | Pas de doc ou doc PDF jamais à jour.                                                       |
| **Validation**         | Vérifie les données avant traitement.  | Plante dès qu'on envoie une mauvaise valeur.                                               |
| **Consistance**        | Format JSON constant et prédictible.   | Change de nom de champ selon les routes.                                                   | 

#### 2. Exemples concrets appliqués à notre projet
Nous avons appliqué plusieurs principes de "Clean API" dans notre implémentation Python :

A. Utilisation des conventions RESTful
Au lieu de créer une route /getMovies, nous utilisons @app.get("/movies"). C'est le standard REST : on utilise la méthode HTTP (GET) pour définir l'action et le nom de la ressource (/movies) pour définir l'objet.

B. Validation automatique avec FastAPI
Grâce à limit: Optional[int] = Query(None, gt=0), notre API refuse automatiquement les requêtes absurdes (ex: ?limit=-5). Le client reçoit une erreur 422 (Unprocessable Entity) propre, ce qui évite au serveur de traiter des données corrompues.

C. Auto-documentation (Swagger)
En utilisant FastAPI, nous avons exposé une documentation interactive sur /docs. Cela permet à n'importe quel développeur Front-end de tester les routes sans avoir accès au code source du Back-end.

#### 3. Difficultés rencontrées et résolutions
Le développement de cette interface a soulevé plusieurs défis techniques :
  --> Le blocage CORS : C'est la difficulté majeure. Comprendre que le navigateur bloque la communication entre le port 5173 et 8000 par sécurité a nécessité la mise en place d'un middleware spécifique.
  --> Format de réponse JSON : Au début, le Front-end cherchait un champ movie.image alors que le Back-end envoyait movie.image_url. La cohérence des noms entre le modèle (JSON) et la vue (JS) est primordiale.
  --> Gestion des fichiers statiques : Faire la différence entre une "route" API et un "serveur de fichiers" pour les images a été une étape clé de la compréhension de l'architecture.

#### 4. Sources consultées
  --> Documentation officielle de FastAPI (fastapi.tiangolo.com) : Pour les bonnes pratiques de validation et de documentation.
  --> Microsoft API Design Guidelines : Pour les conventions de nommage des ressources et la gestion des versions (v1/v2).
  --> Blog technique "Refactoring Guru" : Pour les concepts de Design Patterns appliqués aux échanges de données.