# Sujet du projet : Conception et d&eacute;ploiement d'une application Web orient&eacute;e donn&eacute;es et API

## 1. Contexte g&eacute;n&eacute;ral

Dans le monde professionnel, le d&eacute;veloppement d'une application web moderne ne se limite pas &agrave; &eacute;crire du code.
Il implique une **d&eacute;marche compl&egrave;te**, allant de la conception &agrave; la mise en production, en passant par :

- le choix d'une **architecture coh&eacute;rente** ;
- le d&eacute;veloppement **front-end et back-end** ;
- la gestion et l'exploitation de **donn&eacute;es r&eacute;elles** ;
- l'utilisation de **bases de donn&eacute;es** ;
- la **conteneurisation** et le **d&eacute;ploiement** ;
- la **documentation** et la **justification des choix techniques**.

Ce projet vise &agrave; vous placer dans une situation **r&eacute;aliste**, proche des pratiques industrielles.

---

## 2. Organisation du travail

- **Travail en bin&ocirc;me obligatoire**  
  (le travail en bin&ocirc;me est obgligatoire pour ce projet, il a pour objectif de refl&eacute;ter un contexte r&eacute;el de d&eacute;veloppement)
- Technologies **libres**, tant qu'elles sont coh&eacute;rentes et justifi&eacute;es
- Le projet est construit **progressivement** au fil des TME
- Le **rapport est r&eacute;dig&eacute; au fur et &agrave; mesure**, et non &agrave; la fin uniquement

---

## 3. Objectif global du projet

Le but du projet est de **concevoir, justifier, impl&eacute;menter et d&eacute;ployer** une application web permettant :

- d'afficher un **catalogue de films** ;
- de proposer une **barre de recherche** ;
- de reposer sur une **architecture claire front-end / back-end / base de donn&eacute;es** ;
- d'exposer et consommer une **API HTTP** ;
- d'&ecirc;tre **conteneuris&eacute;e et d&eacute;ployable**.

---

## 4. Source des donn&eacute;es (au choix)

Vous n'&ecirc;tes **pas oblig&eacute;s** d'utiliser une API open-source.

### Option A -- API externe
- Exemple : TMDB
- Lecture de documentation
- Gestion de cl&eacute; API
- Normalisation des donn&eacute;es

### Option B -- Dataset existant
- Exemple : Kaggle  
  https://www.kaggle.com/datasets
- Import dans une base :
  - SQL,
  - NoSQL,
  - ou combinaison des deux

Vous devez justifier le choix de la source de donn&eacute;es.

---

## 5. Architecture technique attendue

### 5.1 Front-end (au choix)

Exemples possibles :
- Astro
- React + Vite
- Angular
- Vue + Vite
- Svelte / SvelteKit

---

### 5.2 Back-end (au choix)

Exemples possibles :
- FastAPI (Python)
- Node.js (Express ou NestJS)
- PHP (Laravel ou Symfony)
- ASP.NET Core (C#)
- Go (Fiber ou Gin)

---

### 5.3 Base de donn&eacute;es (obligatoire)

Vous devez utiliser **au moins une base de donn&eacute;es** :

- SQL : PostgreSQL, MySQL, MariaDB
- NoSQL : MongoDB, Redis, Neo4j, Elasticsearch

Les combinaisons sont autoris&eacute;es.

---

## 6. Fonctionnalit&eacute;s minimales attendues

- Catalogue de films dynamique
- Barre de recherche
- Communication front-end / back-end fonctionnelle
- Gestion minimale des erreurs

---

## 7. Fonctionnalit&eacute;s avanc&eacute;es (optionnelles mais valoris&eacute;es)

- Films favoris
- Recommandations de films similaires
- Cache (Redis)
- Recherche avanc&eacute;e (Elasticsearch)
- Pagination

---

## 8. Conteneurisation et d&eacute;ploiement (obligatoire)

### 8.1 Docker
- Conteneuriser :
  - front-end
  - back-end
  - base de donn&eacute;es

### 8.2 Kubernetes
- D&eacute;ployer sur un cluster
- Utiliser :
  - Pods
  - Services
  - Ingress
  - ConfigMaps
  - Secrets
- R&eacute;fl&eacute;chir au scaling

---

## 9. CI/CD et d&eacute;p&ocirc;t Git

- D&eacute;p&ocirc;t Git (GitLab ou GitHub)
- Pipeline CI/CD automatis&eacute; :
  - build
  - tests (si pr&eacute;sents)
  - build des images Docker
  - d&eacute;ploiement

---

## 10. Modalit&eacute;s de rendu

Le projet doit &ecirc;tre rendu sous la forme d'une **archive `.zip`** contenant :

- l'ensemble du code source ;
- un fichier expliquant **comment lancer le projet** (`README.md` obligatoire) ;
- id&eacute;alement :
  - un script unique &agrave; ex&eacute;cuter (`run.sh`, `make up`, `docker-compose up`, etc.)
  - permettant de lancer automatiquement l'application.
- un document PDF rapport du projet, reprenant les &eacute;l&eacute;ments con&ccedil;us au fil des TME ;
- une vid&eacute;o de pr&eacute;sentation (5 minutes maximum, voir ci-dessous).

L'objectif est que l'application puisse &ecirc;tre **lanc&eacute;e facilement** sans configuration complexe.

---

## 11. Vid&eacute;o de pr&eacute;sentation avec d&eacute;monstration

**Vid&eacute;o de pr&eacute;sentation avec d&eacute;monstration** (5 minutes maximum)  
- Pr&eacute;sentation :
  - de l'architecture,
  - des fonctionnalit&eacute;s principales,
  - du d&eacute;ploiement.
- D&eacute;monstration de l'application.


---

## 12. Date limite

**&Agrave; rendre avant le 30 Avril 2026 &agrave; 23h59**

Tout rendu hors d&eacute;lai pourra &ecirc;tre p&eacute;nalis&eacute;.

---

## 13. Crit&egrave;res d'&eacute;valuation

L'&eacute;valuation portera notamment sur :
- la coh&eacute;rence de l'architecture ;
- la qualit&eacute; des choix techniques ;
- la ma&icirc;trise des outils (API, base, Docker, Kubernetes) ;
- la capacit&eacute; &agrave; justifier ;
- la clart&eacute; de la documentation / rapport ;
- la qualit&eacute; de la pr&eacute;sentation / d&eacute;monstration.

---

## 14. Esprit du projet

L'objectif n'est pas de produire l'application la plus complexe possible,
mais de montrer que vous &ecirc;tes capables de :

- travailler en &eacute;quipe ;
- r&eacute;fl&eacute;chir avant de coder ;
- faire des choix r&eacute;alistes ;
- documenter et expliquer votre travail ;
- vous rapprocher des pratiques professionnelles.

### Exemple d'architecture possible (non impos&eacute;e)

&Agrave; titre **illustratif**, une architecture possible pour ce projet est pr&eacute;sent&eacute;e ci-dessous.

Cet exemple montre une application d&eacute;coup&eacute;e en plusieurs **conteneurs**, avec :
- un **front-end** jouant le r&ocirc;le de client ;
- un **back-end API** structur&eacute; (API Gateway + Data Access Layer) ;
- une ou plusieurs **bases de donn&eacute;es** (SQL, NoSQL, recherche, cache) ;
- un **moteur d'algorithmes** s&eacute;par&eacute; (recommandation, similarit&eacute;, recherche avanc&eacute;e).

**Cette architecture est donn&eacute;e comme exemple** :
- vous n'&ecirc;tes **pas oblig&eacute;s** de la reproduire &agrave; l'identique ;
- vous pouvez la simplifier ou l'adapter ;
- votre propre architecture doit &ecirc;tre **coh&eacute;rente, justifi&eacute;e et expliqu&eacute;e** dans le rapport.

**Sch&eacute;ma d'architecture (exemple)** :  
![Voir l'exemple d'architecture](architecture.png)

