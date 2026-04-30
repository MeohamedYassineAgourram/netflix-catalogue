# TME 5: DOCKER

## PARTIE 1 : INSTALLATION & ARCHITECTURE
### 1. Compréhension des concepts clés
Avant de manipuler Docker, il est essentiel de définir les éléments qui le composent :           
    --> **`Docker Engine`** : C'est le cœur du système, le logiciel qui permet de créer et faire tourner les conteneurs.        
    --> **`Image`** : C'est un modèle immuable (qui ne change pas) composé de couches (layers) utilisé pour créer des conteneurs. On peut le voir comme un plan de construction ou un "moule".       
    --> **`Conteneur`** : C'est un processus isolé (utilisant les technologies namespaces et cgroups de Linux) qui tourne à partir d'une image. C'est l'application finale en cours d'exécution.       
    --> **`Registry`** : C'est le catalogue en ligne (comme le Docker Hub) où l'on peut stocker, partager et télécharger des images.    
    --> **`Layers (Couches)`** : Les images sont construites en empilant des couches successives, ce qui permet d'optimiser le stockage et le téléchargement.

### 2. Installation de Docker sur la VM Ubuntu
Nous avons procédé à l'installation de Docker sur notre machine virtuelle Ubuntu en utilisant le gestionnaire de paquets avec les commandes suivantes :
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
docker --version
```

Pour valider l'installation, nous avons exécuté notre tout premier conteneur de test avec la commande **`sudo docker run hello-world`**.

<div align="center">
  <img src="Docker_Installation.png" width=60%>
  <p><em>Légende</em> : Résultat de l'exécution du conteneur hello-world prouvant le bon fonctionnement de Docker.</p>
</div>

### 3. Différence entre Image, Conteneur et VM
Il est important de distinguer ces trois concepts :             
    --> **`L'Image`** : C'est un simple fichier de configuration statique et immuable.       
    --> **`Le Conteneur`** : C'est l'instance vivante de cette image, exécutée sous forme de processus isolé.        
    --> **`La Machine Virtuelle (VM)`** : Contrairement au conteneur, une VM embarque un système d'exploitation (OS) complet et nécessite un hyperviseur pour fonctionner, ce qui la rend beaucoup plus lourde et gourmande en ressources.

### 4. Réponses aux questions
#### Où sont stockées les images ?
Sur notre système Linux, les images téléchargées sont stockées localement dans l'espace géré par Docker, généralement dans le répertoire **`/var/lib/docker`**.

#### Docker utilise-t-il un hyperviseur ?
Non, sur les systèmes Linux, Docker n'utilise pas d'hyperviseur. Il crée des conteneurs natifs qui partagent directement le noyau (kernel) du système d'exploitation hôte.

#### Quelle est la différence avec VirtualBox ?
VirtualBox est un outil qui crée des Machines Virtuelles (VM) classiques utilisant un hyperviseur et incluant chacune un OS complet. Docker est beaucoup plus léger car il ne virtualise pas le matériel, il isole simplement des processus au sein du même OS.


## PARTIE 2 : MANIPULATION BAS NIVEAU DES CONTENEURS
### Exercice 1 : Exploration
L'objectif de cet exercice était de lancer un conteneur Ubuntu de base, d'y installer des outils réseau et de vérifier sa capacité à communiquer avec l'extérieur.

#### 1. Lancement du conteneur et installation des dépendances
Nous avons démarré un conteneur nommé **`u1`** en mode interactif afin d'accéder à son terminal :
```bash
docker run -it --name u1 ubuntu:22.04 bash
```

Une fois à l'intérieur, l'image Ubuntu étant très légère (version minimaliste), nous avons dû mettre à jour la liste des paquets et installer les outils réseau nécessaires (**`curl`**, **`net-tools`**, **`iproute2`** et **`iputils-ping`**) :
```bash
apt update
apt install -y curl net-tools iproute2 iputils-ping
```

#### 2. Identification de l'adresse IP et test de connectivité
Pour trouver l'adresse IP attribuée par Docker à notre conteneur **`u1`**, nous avons utilisé la commande interne **`ip a`**.
(Note : Il est également possible d'obtenir cette information depuis la machine hôte avec la commande **`docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' u1`**).

Enfin, nous avons validé l'accès à Internet du conteneur en effectuant un test de ping vers un serveur DNS public (Cloudflare) et une requête HTTP via curl :
```bash
ping -c 2 1.1.1.1
curl -I https://netflix.com
```

<div align="center">
  <div style="display: flex; justify-content: center; align-items: flex-start; gap: 10px;">
    <img  src="Ping_Part2_Ex1.png" style="width: 48%;" alt="Capture d'écran du test ping">
    <img src="Curl_Part2_Ex1.png" style="width: 48%;" alt="Capture d'écran du test curl">
  </div>
  <p><em>Légende</em> : Tests de connectivité (ping et curl) réussis depuis l'intérieur du conteneur u1.</p>
</div>

### Exercice 2 : Isolation et Réseaux
Ce second exercice met en évidence l'isolation par défaut des conteneurs et la manière de les faire communiquer via un réseau personnalisé.

#### 1. Lancement des conteneurs en arrière-plan
Nous avons lancé deux nouveaux conteneurs (**`u1`** et **`u2`**) en mode détaché (**`-d`**) pour qu'ils tournent en tâche de fond :
```bash
docker run -dit --name u1 ubuntu:22.04 bash
docker run -dit --name u2 ubuntu:22.04 bash
```

Nous avons ensuite installé les utilitaires réseau sur ces deux conteneurs directement depuis l'hôte grâce à la commande **`docker exec`** :
```bash
docker exec -it u1 bash -lc "apt update && apt install -y iproute2 iputils-ping curl"
docker exec -it u2 bash -lc "apt update && apt install -y iproute2 iputils-ping curl"
```

#### 2. Communication par défaut (via adresse IP)
Par défaut, les conteneurs placés sur le réseau standard de Docker peuvent communiquer entre eux si l'on connaît leur adresse IP exacte. En récupérant l'IP de **`u2`**, nous avons pu effectuer un ping depuis **`u1`**. Cependant, cette méthode n'est pas pratique car les adresses IP peuvent changer.

#### 3. Création d'un réseau personnalisé (DNS interne)
Pour simplifier la communication, nous avons créé un réseau Docker défini par l'utilisateur (user-defined network) nommé films-net, et nous y avons connecté nos deux conteneurs :
```bash
docker network create films-net
docker network connect films-net u1
docker network connect films-net u2
```

#### 4. Test de résolution de nom (DNS)
Une fois sur ce réseau personnalisé, la magie du DNS interne de Docker opère. Nous avons pu vérifier que **`u1`** parvient à joindre **`u2`** en utilisant simplement son nom de conteneur :
```bash
docker exec -it u1 ping -c 2 u2
```

<div align="center">
  <img src="Ping_Part2_Ex2.png" width=60%>
  <p><em>Légende</em> : Succès du ping depuis u1 vers u2 en utilisant le nom d'hôte grâce au réseau personnalisé films-net.</p>
</div>

#### Réponses aux questions
Quelle est la différence entre les réseaux **`bridge`** et **`host`** ?      
    --> **`Le réseau bridge (pont)`** : C'est le mode par défaut. Il crée un réseau virtuel privé isolé de l'hôte (fonctionnant avec un système de NAT). Pour qu'un service situé dans un conteneur bridge soit accessible depuis l'extérieur, il est obligatoire de publier explicitement ses ports de communication.     
    --> **`Le réseau host (hôte)`** : Dans ce mode, le conteneur perd cette isolation réseau et partage directement l'interface réseau de la machine hôte principale. Il est donc plus performant mais offre beaucoup moins de sécurité et d'isolation.


## PARTIE 3 : ANALYSE DU DOCKERFILE
### Analyse et explication du Dockerfile
Le fichier **`Dockerfile`** fourni est un script de construction pour créer une image Docker contenant une application Python. Voici l'explication détaillée de chaque instruction, ligne par ligne :

```bash
FROM python:3.11-slim
```
Explication : C'est l'image de base (les fondations). On dit à Docker de partir d'un système qui contient déjà Python 3.11. Le mot **`slim`** signifie que c'est une version "allégée", débarrassée des outils inutiles pour que l'image finale soit la plus petite et légère possible.

```bash
WORKDIR /app
```
Explication : On crée un dossier appelé **`/app`** à l'intérieur du conteneur et on s'y place. Toutes les commandes suivantes seront exécutées depuis ce dossier (c'est notre espace de travail).

```bash
COPY requirements.txt .
```
Explication : On copie le fichier qui contient la liste des dépendances (les bibliothèques Python dont l'application a besoin) depuis notre ordinateur vers le dossier **`/app`** du conteneur.
Note d'optimisation (très importante) : On fait cette copie avant le reste du code pour utiliser le mécanisme de cache de Docker. Ainsi, si on modifie seulement notre code source plus tard, Docker n'aura pas besoin de retélécharger et réinstaller toutes les dépendances.

```bash
RUN pip install --no-cache-dir -r requirements.txt
```
Explication : On installe les bibliothèques Python listées dans le fichier. L'option **`--no-cache-dir`** demande à l'installateur (**`pip`**) de ne pas garder les fichiers d'installation temporaires en mémoire, toujours dans le but de garder l'image très légère.

```bash
COPY . .
```
Explication : On copie tout le reste de notre code source (le premier **`.`**) vers le dossier de travail du conteneur (le deuxième **`.`**).

```bash
EXPOSE 8000
```
Explication : C'est une simple déclaration. Cela sert de documentation interne pour indiquer que l'application à l'intérieur du conteneur va écouter sur le port 8000. Attention : cette ligne ne publie pas réellement le port vers l'extérieur (pour cela, il faut utiliser l'option **`-p`** lors du lancement du conteneur).

```bash
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
Explication : C'est la commande par défaut qui sera exécutée tout à la fin, uniquement quand le conteneur démarrera. Elle lance le serveur web (**`uvicorn`**) pour faire tourner notre application Python. L'adresse **`0.0.0.0`** indique au serveur d'accepter les connexions venant de l'extérieur du conteneur.


## PARTIE 4 : APPLICATION FILMS
L'objectif de cette partie est de dockeriser une application complète composée d'un backend en Python (FastAPI) et d'un frontend en HTML.

### Étape 1 : Dockeriser uniquement le backend
Pour commencer, nous avons isolé la partie "serveur" (le backend) qui fournit les données des films.
Nous avons créé le **`Dockerfile`** suivant dans le répertoire **`application/backend`** :
```Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Nous avons ensuite construit l'image et lancé le conteneur en publiant le port 8000 :
```bash
docker build -t films-backend .
docker run -d --name backend -p 8000:8000 films-backend
```

Test validé : En accédant à **`http://localhost:8000/movies`**, nous obtenons bien la réponse JSON de notre API avec la liste des films.

<div align="center">
  <img src="./Curl_Part4_Ex1.png" width=60%>
  <p><em>Légende</em> : Le backend fonctionne et renvoie correctement les données JSON.</p>
</div>

### Étape 2 : Créer 2 conteneurs (frontend + backend)
Nous avons ensuite séparé l'architecture en deux conteneurs distincts, communiquant via un réseau privé.      
    1. Configuration : Nous avons créé un réseau personnalisé **`films-net`**. Dans le code du frontend (**`index.html`**), nous avons respecté la contrainte de remplacer **`localhost`** par le nom du service backend (**`http://backend:8000/movies`**).   
    2. Dockerfile du Frontend : Nous avons utilisé une image **`nginx:alpine`** pour servir la page HTML sur le port 80.   
    3. Lancement : Les deux conteneurs ont été lancés sur le réseau **`films-net`**.

Observation et Analyse : En nous connectant sur **`http://localhost:8080`** (le frontend), la structure de la page s'affiche mais les données et les images sont manquantes.
Pourquoi ? Le navigateur web (Firefox) s'exécute sur l'hôte, à l'extérieur du réseau Docker **`films-net`**. Il ne peut donc pas résoudre le nom DNS interne **`backend`** défini dans le fichier HTML. De plus, le backend n'est pas encore configuré pour servir les fichiers statiques (images). Cela justifie le passage à l'architecture de l'Étape 3.

### Étape 3 : Version mono-conteneur
Pour régler le problème de communication client/serveur et simplifier l'accès, nous avons regroupé le frontend et le backend au sein d'un même conteneur.                 
1. **Modifications du code :** Le fichier **`index.html`** a été déplacé dans le répertoire **`static/`** du backend.
    * L'appel API dans le HTML a été remplacé par un chemin relatif : **`fetch("/movies")`**.
    * Le fichier **`main.py`** a été mis à jour pour servir la page d'accueil (**`/`**) et monter le répertoire des fichiers statiques.               

2. **Déploiement :** Après avoir supprimé les anciens conteneurs pour libérer le port 8000, nous avons reconstruit et lancé cette version unifiée :
```bash
docker build -t films-mono .
docker run -d --name films-mono -p 8000:8000 films-mono
```

<div align="center">
  <img src="./Test_Part4_Ex3.png" width=60%>
  <p><em>Légende</em> : L'architecture mono-conteneur permet d'afficher correctement l'application complète.</p>
</div>

* **`Quels sont les avantages d'une version mono-conteneur ?`**                  
Le déploiement est beaucoup plus simple puisqu'il n'y a qu'une seule image à construire et un seul conteneur à gérer. C'est une solution très rapide et idéale pour de petits projets.

* **`Quels sont les inconvénients ?`**                  
L'architecture est moins modulaire. Le frontend et le backend sont fortement couplés, ce qui rend le système difficile à "scaler" de manière indépendante. C'est également une approche moins propre par rapport aux standards de l'architecture microservices privilégiée en production et dans les entreprises.


## PARTIE 5 : DEBUGGING & INSPECTION
### 1. Découverte des commandes d'inspection
* **`docker inspect <conteneur>`** : C'est l'outil d'analyse le plus complet. Il renvoie un objet JSON contenant la totalité des métadonnées et de la configuration bas niveau du conteneur (adresses IP, points de montage, configuration réseau, état actuel, etc.).           
* **`docker stats`** : Cet outil agit comme un gestionnaire de tâches en temps réel. Il affiche un flux continu détaillant la consommation des ressources de tous les conteneurs actifs (pourcentage d'utilisation du CPU, mémoire vive (RAM) allouée et limite, trafic réseau (I/O) et écritures disque).          
* **`docker top <conteneur>`** : Similaire à la commande top sous Linux, elle permet de lister les processus (PID) qui s'exécutent actuellement à l'intérieur du conteneur, depuis le point de vue de l'hôte.             
* **`docker logs <conteneur>`** : Permet de récupérer les journaux (logs) générés par l'application principale du conteneur. C'est indispensable pour lire les messages d'erreur ou les requêtes reçues sans avoir à entrer dans le conteneur.                  
* **`docker exec -it <conteneur> <commande>`** : Permet d'exécuter une nouvelle commande dans un conteneur qui tourne déjà. Nous l'avons beaucoup utilisée dans les parties précédentes pour ouvrir un terminal (bash) à l'intérieur de nos environnements isolés.

<div align="center">
  <img src="./Docker-Stats_Part5.png" width=60%>
  <p><em>Légende</em> : Surveillance en temps réel de la consommation des ressources du conteneur via docker stats.</p>
</div>

### 2. Réponses aux questions
* **Où voir les variables d'environnement d'un conteneur ?**                     
Les variables d'environnement se trouvent dans les métadonnées du conteneur. Il faut utiliser la commande **`docker inspect <nom_conteneur>`** et chercher dans le JSON généré. Plus précisément, elles sont listées dans la section **`"Config"`**, sous la clé **`"Env"`**.

* **Comment limiter le CPU d'un conteneur ?**            
Par défaut, un conteneur peut utiliser toutes les ressources du processeur de la machine hôte. Pour le limiter, il faut ajouter l'option **`--cpus`** lors de la création du conteneur avec **`docker run`**.                     
Exemple : **`docker run --cpus="0.5" ubuntu`** limitera le conteneur à l'utilisation maximale d'un demi-cœur de processeur.

* **Comment limiter la mémoire (RAM) d'un conteneur ?**                  
De la même manière, on utilise l'option **`-m`** ou **`--memory`** lors de l'exécution de la commande **`docker run`**. Si le conteneur dépasse cette limite stricte, le système risque de forcer l'arrêt du processus principal (erreur OOM Kill - Out Of Memory).                
Exemple : **`docker run -m 512m ubuntu`** limitera l'empreinte mémoire du conteneur à 512 mégaoctets maximum.


## PARTIE 6 : LAYERS & CACHE (Avancé)
### 1. Découverte des commandes liées aux images
Nous avons utilisé deux commandes pour analyser la structure de notre image finale films-mono :                         
* **`docker history <nom_image>`** : Cette commande affiche l'historique de construction de l'image. Elle liste chaque étape (chaque instruction du Dockerfile) comme une couche distincte, en précisant sa taille en mégaoctets. Cela permet de repérer facilement quelle ligne du Dockerfile alourdit l'image.                

* **`docker image inspect <nom_image>`** : Comme pour les conteneurs, cette commande renvoie le détail technique complet de l'image au format JSON. C'est ici que l'on peut voir l'architecture, le système d'exploitation de base, et surtout la liste exacte des empreintes cryptographiques (SHA256) de toutes les couches dans la section "RootFS.Layers".

### 2. Réponses aux questions
* **Combien de layers votre image possède-t-elle ?**              
Pour obtenir le nombre exact de couches physiques, nous pouvons compter les éléments dans la section RootFS. Notre image possède 8 layers. Cela correspond aux couches de l'image de base Python, auxquelles s'ajoutent les couches générées par nos instructions **`COPY`** et **`RUN`**.

* **Comment réduire la taille d'une image Docker ?**               
Il existe plusieurs bonnes pratiques pour alléger une image :                    
  1. **Regrouper les commandes **`RUN`**** : Chaque instruction **`RUN`** crée une nouvelle couche. Enchaîner les commandes avec **`&&`** (ex: **`RUN apt update && apt install ...`**) permet de ne créer qu'une seule couche.

  2. **Nettoyer les caches** : Toujours supprimer les fichiers temporaires après une installation (comme l'option **`--no-cache-dir`** de pip, ou **`rm -rf /var/lib/apt/lists/*`** pour apt).

  3. **Utiliser un fichier **`.dockerignore`**** : Lors de l'instruction **`COPY . .`**, Docker copie tout le dossier. Utiliser un **`.dockerignore`** évite d'importer des fichiers inutiles (comme le dossier **`__pycache__`** ou les fichiers cachés de l'OS) qui alourdiraient l'image pour rien.

  4. **Le Multi-stage build** : Une technique avancée consistant à utiliser une image pour compiler le code, puis à copier uniquement le résultat final dans une image vierge très légère.

* **Pourquoi l'image python:slim est-elle plus légère ?**               
L'image standard **`python:3.11`** contient non seulement Python, mais aussi tout un ensemble d'outils de compilation (compilateurs C/C++, bibliothèques de développement, documentation) permettant de compiler des modules complexes depuis leurs sources.
La version **`python:3.11-slim`** est amputée de tous ces outils de développement non essentiels. Elle ne conserve que le strict minimum vital pour exécuter un script Python pré-compilé, ce qui réduit considérablement son poids total (elle passe souvent de près de 1 Go à environ 150 Mo).