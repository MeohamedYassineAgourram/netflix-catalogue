# TME 5: DOCKER

Objectif : &Agrave; la fin du TME vous devez **comprendre** Docker.
Interdiction de copier-coller sans comprendre.

---

# PLAN DU TME

1. Installation & compr&eacute;hension architecture Docker
2. Manipulation bas niveau des conteneurs
3. R&eacute;seaux Docker approfondis
4. Dockerfile d&eacute;taill&eacute; + optimisation
5. Dockerisation application multi-conteneurs
6. Debug & inspection
7. Layers & cache Docker

---

# PARTIE 1: INSTALLATION + ARCHITECTURE 

## Objectif
Comprendre :
- Docker Engine
- Images
- Conteneurs
- Registry
- Layers

## Travail demand&eacute;

1. Installer Docker sur la VM Ubuntu
2. Expliquer la diff&eacute;rence entre :
   - Image
   - Conteneur
   - VM

## Documentation officielle
https://docs.docker.com/engine/install/

## Questions &agrave; r&eacute;pondre:

- O&ugrave; sont stock&eacute;es les images ?
- Docker utilise-t-il un hyperviseur ?
- Quelle est la diff&eacute;rence avec VirtualBox ?

---

# PARTIE 2: 

## Exercice 1: Exploration

Lancez un conteneur Ubuntu.

Objectifs :
- Installer curl
- Installer net-tools
- Trouver l'IP du conteneur
- Tester connectivit&eacute; r&eacute;seau

Vous devez chercher les commandes.

---

## Exercice 2: Isolation

1. Lancez 2 conteneurs Ubuntu.
2. V&eacute;rifiez s'ils peuvent communiquer.
3. Cr&eacute;ez un r&eacute;seau Docker personnalis&eacute;.
4. Refaites le test.

Questions :
- Quelle est la diff&eacute;rence entre bridge et host ?

---

# PARTIE 3: 

## Analyse du Dockerfile suivant

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Questions :
- Explique le Dockerfile ligne par ligne.

---

# PARTIE 4: APPLICATION FILMS 

Vous avez :
- backend FastAPI
- frontend HTML

## &Eacute;tape 1
Dockeriser uniquement le backend.

Test attendu :
http://localhost:8000/movies

---

## &Eacute;tape 2
Cr&eacute;er 2 conteneurs (frontend + backend)

Contraintes :
- R&eacute;seau personnalis&eacute;
- Pas de localhost dans le frontend
- Utiliser nom du service

---

## &Eacute;tape 3
Cr&eacute;er une version mono-conteneur.

Questions :
- Avantages ?
- Inconv&eacute;nients ?

---

# PARTIE 5: DEBUGGING & INSPECTION

Commandes &agrave; d&eacute;couvrir :

- docker inspect
- docker stats
- docker top
- docker logs
- docker exec

Questions :
- O&ugrave; voir les variables d'environnement ?
- Comment limiter CPU d'un conteneur ?
- Comment limiter m&eacute;moire ?

---

# PARTIE 6 - LAYERS & CACHE (Avanc&eacute;)

Commandes :

docker history <image>
docker image inspect <image>

Questions :
- Combien de layers votre image poss&egrave;de-t-elle ?
- Comment r&eacute;duire la taille ?
- Pourquoi python:slim est plus l&eacute;ger ?

---


# ANNEXE: COMMANDES &Agrave; MA&Icirc;TRISER

- docker pull
- docker build
- docker run
- docker exec
- docker ps
- docker images
- docker logs
- docker inspect
- docker network create
- docker network ls
- docker volume create
- docker history
- docker stats
- docker run -d --memory=256m ubuntu sleep infinity

---

&Agrave; la fin vous devez &ecirc;tre capables de dockeriser n'importe quelle application simple.


---

# CORRECTIONS / TUTORIEL (&agrave; lire apr&egrave;s avoir cherch&eacute;)

Cette partie sert &agrave; **valider** votre travail et &agrave; comprendre les commandes correctes.
Ne la lisez pas avant d'avoir tent&eacute; les exercices.

---

## PARTIE 1: Installation & v&eacute;rification

### Installation Docker (Ubuntu): m&eacute;thode simple (paquets Ubuntu)
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
docker --version
```
Test :
```bash
sudo docker run hello-world
```

### (Option) Utiliser Docker sans sudo
```bash
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```
> Sur certaines VM, il faut se d&eacute;connecter / reconnecter pour que le groupe soit pris en compte.

### Lien officiel (autres OS + m&eacute;thode officielle)
- Docs Docker Engine install : https://docs.docker.com/engine/install/

### R&eacute;ponses rapides aux questions d'architecture
- **Images** : mod&egrave;les immuables (couches/layers) pour cr&eacute;er des conteneurs
- **Conteneurs** : processus isol&eacute;s (namespaces/cgroups) qui utilisent une image + couche &eacute;criture
- **VM** : OS complet + hyperviseur (plus lourd)
- O&ugrave; sont stock&eacute;es les images ? --> dans l'espace Docker (ex : `/var/lib/docker` sur Linux, d&eacute;pend de l'installation)
- Docker utilise un hyperviseur ? --> Sur Linux : non (conteneurs natifs). Sur Mac/Windows : souvent via VM l&eacute;g&egrave;re (Docker Desktop).

---

## PARTIE 2: 

### Exercice 1: Lancer Ubuntu + installer outils + IP

1) Lancer un conteneur interactif :
```bash
docker run -it --name u1 ubuntu:22.04 bash
```

2) Dans le conteneur, installer curl + net-tools + iproute2 :
```bash
apt update
apt install -y curl net-tools iproute2
```

3) Trouver l'IP du conteneur (2 fa&ccedil;ons) :

- **Depuis le conteneur** :
```bash
ip a
```
- **Depuis l'h&ocirc;te** :
```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' u1
```

4) Tester la connectivit&eacute; :
```bash
ping -c 2 1.1.1.1
curl -I https://example.com
```

> Si `ping` ne marche pas : `apt install -y iputils-ping`

### Exercice 2: 2 conteneurs + r&eacute;seau personnalis&eacute;

1) Lancer 2 conteneurs (en arri&egrave;re-plan) :
```bash
docker run -dit --name u1 ubuntu:22.04 bash
docker run -dit --name u2 ubuntu:22.04 bash
```

2) Installer ping/curl dans u1/u2 (via exec) :
```bash
docker exec -it u1 bash -lc "apt update && apt install -y iproute2 iputils-ping curl"
docker exec -it u2 bash -lc "apt update && apt install -y iproute2 iputils-ping curl"
```

3) Test ping par IP :
```bash
U2IP=$(sudo docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' u2)
docker exec -it u1 ping -c 2 $U2IP
```

4) Cr&eacute;er un r&eacute;seau user-defined et connecter les conteneurs :
```bash
docker network create films-net
docker network connect films-net u1
docker network connect films-net u2
```

5) Test ping par **nom** (DNS Docker) :
```bash
docker exec -it u1 ping -c 2 u2
```

Pourquoi &ccedil;a marche “par nom” maintenant ?
- Sur un r&eacute;seau user-defined, Docker fournit un **DNS interne** (r&eacute;solution des noms des conteneurs).

### Diff&eacute;rence bridge vs host (tr&egrave;s r&eacute;sum&eacute;)
- **bridge** : r&eacute;seau virtuel NAT, ports &agrave; publier pour acc&eacute;der depuis l'h&ocirc;te
- **host** : partage r&eacute;seau de l'h&ocirc;te (moins d'isolation)

---

## PARTIE 3: Dockerfile en profondeur

### Analyse du Dockerfile (r&eacute;ponses)
- `COPY requirements.txt` avant le code --> **cache** : on ne r&eacute;installe pas les d&eacute;pendances &agrave; chaque modif de code.
- `EXPOSE` --> documentation interne (ne publie pas le port). Publication : `-p`.

---

## PARTIE 4: Application Films (solutions)

### &Eacute;tape 1: Backend seul

Cr&eacute;e le dockerFile dans le dossier application/backend

```bash
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

```


```bash
cd application/backend
docker build -t films-backend .
docker run -d --name backend -p 8000:8000 films-backend
curl http://localhost:8000/movies
docker logs -f backend
docker exec -it backend bash # Entrez dans le conteneur puis utilisez `ls` pour voir son contenu
```

### &Eacute;tape 2: Deux conteneurs + r&eacute;seau

1) Frontend : remplacer l'URL API par `http://backend:8000/movies` (pas localhost)

2) Dockerfile frontend :
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

3) Lancement :
```bash

# V&eacute;rifier si le r&eacute;seau films-net existe d&eacute;j&agrave;
docker network ls

# Si rien ne s'affiche --> cr&eacute;er le r&eacute;seau
docker network create films-net

# Sinon --> ne rien faire (ne pas recr&eacute;er le r&eacute;seau)


# Voir tous les conteneurs
docker ps -a

# Supprimer backend s'il existe
docker rm -f backend

cd application/backend
docker build -t films-backend .
docker run -d --name backend --network films-net -p 8000:8000 films-backend

cd ../frontend
docker build -t films-frontend .
docker run -d --name frontend --network films-net -p 8080:80 films-frontend
```

Test : `http://localhost:8080`

Debug r&eacute;seau :
```bash
docker network inspect films-net
```

### &Eacute;tape 3: Mono-conteneur (Front + Back ensemble)

#### Objectif

Cr&eacute;er un seul conteneur Docker qui :
- Sert l'API `/movies`
- Sert la page web `/`
- Sert les fichiers statiques (HTML, ...)

---

#### Structure du projet

Dans `application/backend/` :

```
application/backend/
  main.py
  requirements.txt
  Dockerfile
  static/
    index.html
```

---

#### Nouveau main

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], 
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# API
movies = [
    {"title": "Inception", "image": "/static/images/inception.png"},
    {"title": "Interstellar", "image": "/static/images/interstellar.png"},
    {"title": "The Matrix", "image": "/static/images/matrix.png"},
]

@app.get("/movies")
def get_movies():
    return movies

# Page principale
@app.get("/")
def root():
    return FileResponse("static/index.html")

# Servir les fichiers statiques (CSS, JS, images…)
app.mount("/static", StaticFiles(directory="static"), name="static")
```

---

#### Important : Modifier index.html

Dans `index.html`, utiliser :

```javascript
fetch("/movies")
```

Ne pas utiliser :
```
http://backend:8000/movies
```
Car ici tout tourne dans le m&ecirc;me conteneur.

---

### Nouveau Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

#### Build & Run

Depuis `application/backend` :

```bash
docker build -t films-mono .
docker rm -f films-mono 2>/dev/null || true
docker run -d --name films-mono -p 8000:8000 films-mono
```

---

### Tests

- Frontend : http://localhost:8000/
- API : http://localhost:8000/movies

---


## Avantages
- D&eacute;ploiement simple
- Un seul conteneur &agrave; g&eacute;rer
- Facile pour petit projet

## Inconv&eacute;nients
- Moins modulaire
- Difficile &agrave; scaler s&eacute;par&eacute;ment
- Moins propre en architecture microservices

---

En production, on s&eacute;pare g&eacute;n&eacute;ralement frontend et backend.


---

## PARTIE 5: Debug & inspection (solutions)

```bash
docker inspect backend
docker stats
docker top backend
docker logs backend
docker exec -it backend bash
```

Limiter CPU/m&eacute;moire :
```bash
docker run -d --name limited --cpus="0.5" -m 256m ubuntu:22.04 sleep infinity
```

---

## PARTIE 6: Layers & cache (solutions)

```bash
docker history films-backend
docker image inspect films-backend
```

---

